# 属性注入

在完成Bean实例化后，相关属性值（成员变量和方法中的变量）赋值在**populateBean**方法中完成完成。

## 测试源码

基本类

```java
class Student {

    private String name;

    public Student(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    @Override
    public String toString() {
        return "Student [name=" + name + "]";
    }
}


// 需要进行依赖注入的类
class TestBean {

    private Student student;

    public void echo() {
        System.out.println("I'm a student : " + student);
    }

    public Student getStudent() {
        return student;
    }

    public void setStudent(Student student) {
        this.student = student;
    }
}
```

spring配置

```java
<bean class="org.example.abc.xml.Student" id="student">
    <constructor-arg index="0" value="test"/>
</bean>
<bean class="org.example.abc.xml.TestBean" id="testBean">
    <property name="student" ref="student"/>
</bean>
```

## 源码分析

直接从populateBean方法开始

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
   if (bw == null) {
      if (mbd.hasPropertyValues()) {
         throw new BeanCreationException(
               mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
      }
      else {
         // Skip property population phase for null instance.
         return;
      }
   }

   // Give any InstantiationAwareBeanPostProcessors the opportunity to modify the
   // state of the bean before properties are set. This can be used, for example,
   // to support styles of field injection.
   // 给InstantiationAwareBeanPostProcessors最后一次机会在属性注入前修改Bean的属性值
	 // 具体通过调用postProcessAfterInstantiation方法，如果调用返回false,表示不必继续进行依赖注入，直接返回
   // 默认的实现都是true
   if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
      for (BeanPostProcessor bp : getBeanPostProcessors()) {
         if (bp instanceof InstantiationAwareBeanPostProcessor) {
            InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
            if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
               return;
            }
         }
      }
   }
   // xml中获取配置的属性值
   PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);
   // 处理由xml配置的bean
   int resolvedAutowireMode = mbd.getResolvedAutowireMode();
   // <bean autowire="byType || byName" />
   if (resolvedAutowireMode == AUTOWIRE_BY_NAME || resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
      MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
      // Add property values based on autowire by name if applicable.
      // 名称注入
      if (resolvedAutowireMode == AUTOWIRE_BY_NAME) {
         // <bean autowire="byType" /> 且xml不存在的简单类型的属性，会进行值解析，然后put到newPvs
         autowireByName(beanName, mbd, bw, newPvs);
      }
      // 类型注入
      // Add property values based on autowire by type if applicable.
      if (resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
         // <bean autowire="byName" /> 且xml不存在的简单类型的属性，会进行值解析，然后put到newPvs
         autowireByType(beanName, mbd, bw, newPvs);
      }
      pvs = newPvs;
   }
   // 容器是否注册了InstantiationAwareBeanPostProcessor
   boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
   // 是否进行依赖检查
   boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);

   PropertyDescriptor[] filteredPds = null;
   if (hasInstAwareBpps) {
      if (pvs == null) {
         pvs = mbd.getPropertyValues();
      }
      for (BeanPostProcessor bp : getBeanPostProcessors()) {
         if (bp instanceof InstantiationAwareBeanPostProcessor) {
            InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
            // 执行postProcessProperties
            // @value @Autowired注解的注入都是在AutowiredAnnotationBeanPostProcessor这里完成的
            // @Resource通过CommonAnnotationBeanPostProcessor进行处理这里不做讲解
            PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
            if (pvsToUse == null) {
               if (filteredPds == null) {
                  // 过滤出所有需要进行依赖检查的属性编辑器
                  filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
               }
               // 执行postProcessPropertyValues TODO
               pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
               if (pvsToUse == null) {
                  return;
               }
            }
            pvs = pvsToUse;
         }
      }
   }
   // 检查是否满足相关依赖关系，对应的depends-on属性，需要确保所有依赖的Bean先完成初始化
   if (needsDepCheck) {
      if (filteredPds == null) {
         filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
      }
       // 将pvs上所有的属性填充到BeanWrapper对应的Bean实例中，注意到这一步，TestBean的student属性还是RuntimeBeanReference，即还未解析实际的Student实例
      checkDependencies(beanName, mbd, filteredPds, pvs);
   }

   if (pvs != null) {
      applyPropertyValues(beanName, mbd, bw, pvs);
   }
}
```

> 处理xml定义的属性，并将属性赋值（必须有set方法，否则抛出异常）
>
> 当<bean autowire="byType || byName" />时，处理非简单类型的字段属性赋值（xml没有对应字段定义、含有set方法）
>
> 注意：非xml中定义的简单属性这了不做处理

AbstractAutowireCapableBeanFactory#autowireByName

```java
/**
 * 如果自动装配设置为“byName”，则使用对该工厂中其他bean的引用来填写所有缺少的属性值。
 */
protected void autowireByName(
      String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {
	 // 过滤出满足装配条件的Bean属性
   String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
   for (String propertyName : propertyNames) {
      if (containsBean(propertyName)) {
         // 获取bean
         Object bean = getBean(propertyName);
         pvs.add(propertyName, bean);
         // 注册为bean依赖
         registerDependentBean(propertyName, beanName);
         if (logger.isTraceEnabled()) {
            logger.trace("Added autowiring by name from bean name '" + beanName +
                  "' via property '" + propertyName + "' to bean named '" + propertyName + "'");
         }
      }
      else {
         if (logger.isTraceEnabled()) {
            logger.trace("Not autowiring property '" + propertyName + "' of bean '" + beanName +
                  "' by name: no matching bean found");
         }
      }
   }
}
```

AbstractAutowireCapableBeanFactory#unsatisfiedNonSimpleProperties

```java
/**
 * 返回一个不满足的非简单bean属性数组，这些属性可能是对工厂中其他bean的不满足引用。不包括简单属性，例如基本类型或字符串。
 */
protected String[] unsatisfiedNonSimpleProperties(AbstractBeanDefinition mbd, BeanWrapper bw) {
   Set<String> result = new TreeSet<>();
   // xml中的属性
   PropertyValues pvs = mbd.getPropertyValues();
   // 全部属性
   PropertyDescriptor[] pds = bw.getPropertyDescriptors();
   for (PropertyDescriptor pd : pds) {
      if (pd.getWriteMethod() != null && !isExcludedFromDependencyCheck(pd) && !pvs.contains(pd.getName()) &&
            !BeanUtils.isSimpleProperty(pd.getPropertyType())) {
         result.add(pd.getName());
      }
   }
   return StringUtils.toStringArray(result);
}
```

> 就是类型为对象类型的属性，但是这里并不是将所有的对象类型都都会找到，比如 8 个原始类型，String 类型 ，Number类型、Date类型、URL类型、URI类型等都会被忽略
>
> unsatisfiedNonSimpleProperties得到符合下面条件的属性名称
> 1.有setter方法
> 2.需要进行依赖检查
> 3.不包含在XML配置中
> 4.不是简单类型（基本数据类型，枚举，日期等）
>  这里可以看到XML配置优先级高于自动注入的优先级
>  不进行依赖检查的属性，也不会进行属性注入

AbstractAutowireCapableBeanFactory#autowireByType

```java
protected void autowireByType(
      String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {
	 // 获取类型转化器
   TypeConverter converter = getCustomTypeConverter();
   if (converter == null) {
      converter = bw;
   }

   Set<String> autowiredBeanNames = new LinkedHashSet<>(4);
   // 过滤出满足装配条件的Bean属性
   String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
   for (String propertyName : propertyNames) {
      try {
         PropertyDescriptor pd = bw.getPropertyDescriptor(propertyName);
         // Don't try autowiring by type for type Object: never makes sense,
         // even if it technically is a unsatisfied, non-simple property.
         // 非object类型
         if (Object.class != pd.getPropertyType()) {
            // 获取set方法中的参数
            MethodParameter methodParam = BeanUtils.getWriteMethodParameter(pd);
            // Do not allow eager init for type matching in case of a prioritized post-processor.
            // 定义是否允许懒加载。
            boolean eager = !(bw.getWrappedInstance() instanceof PriorityOrdered);
            DependencyDescriptor desc = new AutowireByTypeDependencyDescriptor(methodParam, eager);
            // 这里会根据传入desc里的入参类型，作为依赖装配的类型
					  // 再根据这个类型在BeanFacoty中查找所有类或其父类相同的BeanName
				    // 最后根据BeanName获取或初始化相应的类，然后将所有满足条件的BeanName填充到autowiredBeanNames中。
            Object autowiredArgument = resolveDependency(desc, beanName, autowiredBeanNames, converter);
            if (autowiredArgument != null) {
               pvs.add(propertyName, autowiredArgument);
            }
            for (String autowiredBeanName : autowiredBeanNames) {
               // 注册依赖
               registerDependentBean(autowiredBeanName, beanName);
               if (logger.isTraceEnabled()) {
                  logger.trace("Autowiring by type from bean name '" + beanName + "' via property '" +
                        propertyName + "' to bean named '" + autowiredBeanName + "'");
               }
            }
            autowiredBeanNames.clear();
         }
      }
      catch (BeansException ex) {
         throw new UnsatisfiedDependencyException(mbd.getResourceDescription(), beanName, propertyName, ex);
      }
   }
}
```

> resolveDependency这里的逻辑就不重复介绍了，上一篇详细介绍过

#### 属性的自动注入

AutowiredAnnotationBeanPostProcessor#postProcessProperties

```java
@Override
public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {
   // 这里获取到的是解析过的缓存好的注入元数据（@Value @Auowired修饰的字段和方法这里都存在这个对象中）
   InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);
   try {
      // 直接调用inject方法
      // 存在两种InjectionMetadata
      // 1.AutowiredFieldElement
      // 2.AutowiredMethodElement
      // 分别对应字段的属性注入以及方法的属性注入
      metadata.inject(bean, beanName, pvs);
   }
   catch (BeanCreationException ex) {
      throw ex;
   }
   catch (Throwable ex) {
      throw new BeanCreationException(beanName, "Injection of autowired dependencies failed", ex);
   }
   return pvs;
}
```

> @Value、@Autowired修饰的属性会被自动赋值；@Value、@Autowire修饰的方法会的入参也会被自动赋值，同时方法会被执行
>

InjectionMetadata#inject

```java
public void inject(Object target, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
   Collection<InjectedElement> checkedElements = this.checkedElements;
   Collection<InjectedElement> elementsToIterate =
         (checkedElements != null ? checkedElements : this.injectedElements);
   if (!elementsToIterate.isEmpty()) {
      for (InjectedElement element : elementsToIterate) {
         if (logger.isTraceEnabled()) {
            logger.trace("Processing injected element of bean '" + beanName + "': " + element);
         }
         element.inject(target, beanName, pvs);
      }
   }
}
```

AutowiredAnnotationBeanPostProcessor#inject

```java
@Override
protected void inject(Object bean, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
      // 获取字段属性
      Field field = (Field) this.member;
      Object value;
      // 是否有缓存
      if (this.cached) {
         // 取出缓存中DependencyDescriptor并调用beanFactory.resolveDependency进行依赖解析
         value = resolvedCachedArgument(beanName, this.cachedFieldValue);
      }
      else {
         // 创建DependencyDescriptor对象
         DependencyDescriptor desc = new DependencyDescriptor(field, this.required);
         desc.setContainingClass(bean.getClass());
         // autowiredBeanNames为自动注入beanName的集合
         Set<String> autowiredBeanNames = new LinkedHashSet<>(1);
         Assert.state(beanFactory != null, "No BeanFactory available");
         TypeConverter typeConverter = beanFactory.getTypeConverter();
         try {
            // 解析依赖
            value = beanFactory.resolveDependency(desc, beanName, autowiredBeanNames, typeConverter);
         }
         catch (BeansException ex) {
            throw new UnsatisfiedDependencyException(null, beanName, new InjectionPoint(field), ex);
         }
         synchronized (this) {
            if (!this.cached) {
               if (value != null || this.required) {
                  this.cachedFieldValue = desc;
                  // 注册依赖bean
                  registerDependentBeans(beanName, autowiredBeanNames);
                  if (autowiredBeanNames.size() == 1) {
                     String autowiredBeanName = autowiredBeanNames.iterator().next();
                     // bean工厂是存在autowiredBeanName && 类型和字段属性一样
                     if (beanFactory.containsBean(autowiredBeanName) &&
                           beanFactory.isTypeMatch(autowiredBeanName, field.getType())) {
                        // 缓存一个ShortcutDependencyDescriptor对象
                        this.cachedFieldValue = new ShortcutDependencyDescriptor(
                              desc, autowiredBeanName, field.getType());
                     }
                  }
               }
               else {
                  // 解析的依赖为null或者required=false
                  this.cachedFieldValue = null;
               }
               // 标记为缓存
               this.cached = true;
            }
         }
      }
      if (value != null) {
         // 打破封装
         ReflectionUtils.makeAccessible(field);
         // 对字段赋值
         field.set(bean, value);
      }
   }
}
```

> 反射对方法的进行执行

AutowiredAnnotationBeanPostProcessor.AutowiredMethodElement#inject

```java
@Override
protected void inject(Object bean, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
   if (checkPropertySkipping(pvs)) {
      return;
   }
   Method method = (Method) this.member;
   Object[] arguments;
   // 已经缓存
   if (this.cached) {
      // Shortcut for avoiding synchronization...
      // 直接获取缓存中的参数，进行依赖解析
      arguments = resolveCachedArguments(beanName);
   }
   else {
      // 获取参数的个数
      int argumentCount = method.getParameterCount();
      arguments = new Object[argumentCount];
      // 创建DependencyDescriptor[]用来包装字段
      DependencyDescriptor[] descriptors = new DependencyDescriptor[argumentCount];
      Set<String> autowiredBeans = new LinkedHashSet<>(argumentCount);
      Assert.state(beanFactory != null, "No BeanFactory available");
      TypeConverter typeConverter = beanFactory.getTypeConverter();
      for (int i = 0; i < arguments.length; i++) {
         // 对字段遍历并进行包装
         MethodParameter methodParam = new MethodParameter(method, i);
         DependencyDescriptor currDesc = new DependencyDescriptor(methodParam, this.required);
         currDesc.setContainingClass(bean.getClass());
         descriptors[i] = currDesc;
         try {
            // 逻辑基本一样的，最终也是调用beanFactory.resolveDependency方法
            Object arg = beanFactory.resolveDependency(currDesc, beanName, autowiredBeans, typeConverter);
            if (arg == null && !this.required) {
               arguments = null;
               break;
            }
            arguments[i] = arg;
         }
         catch (BeansException ex) {
            throw new UnsatisfiedDependencyException(null, beanName, new InjectionPoint(methodParam), ex);
         }
      }
      synchronized (this) {
         if (!this.cached) {
            if (arguments != null) {
               DependencyDescriptor[] cachedMethodArguments = Arrays.copyOf(descriptors, arguments.length);
               // 注册依赖
               registerDependentBeans(beanName, autowiredBeans);
               if (autowiredBeans.size() == argumentCount) {
                  Iterator<String> it = autowiredBeans.iterator();
                  Class<?>[] paramTypes = method.getParameterTypes();
                  for (int i = 0; i < paramTypes.length; i++) {
                     String autowiredBeanName = it.next();
                     // bean工厂是存在autowiredBeanName && 类型和字段属性一样
                     if (beanFactory.containsBean(autowiredBeanName) &&
                           beanFactory.isTypeMatch(autowiredBeanName, paramTypes[i])) {
                        // 缓存一个ShortcutDependencyDescriptor对象
                        cachedMethodArguments[i] = new ShortcutDependencyDescriptor(
                              descriptors[i], autowiredBeanName, paramTypes[i]);
                     }
                  }
               }
               // 缓存参数
               this.cachedMethodArguments = cachedMethodArguments;
            }
            else {
               // 没有参数
               this.cachedMethodArguments = null;
            }
            // 标记缓存
            this.cached = true;
         }
      }
   }
   if (arguments != null) {
      try {
         // 打破封装(非public方法可以执行)
         ReflectionUtils.makeAccessible(method);
         // 执行方法
         method.invoke(bean, arguments);
      }
      catch (InvocationTargetException ex) {
         throw ex.getTargetException();
      }
   }
}
```

> 最终赋值还是通过java反射进行的

AbstractAutowireCapableBeanFactory#applyPropertyValues

```java
protected void applyPropertyValues(String beanName, BeanDefinition mbd, BeanWrapper bw, PropertyValues pvs) {
   if (pvs.isEmpty()) {
      return;
   }

   if (System.getSecurityManager() != null && bw instanceof BeanWrapperImpl) {
      ((BeanWrapperImpl) bw).setSecurityContext(getAccessControlContext());
   }

   MutablePropertyValues mpvs = null;
   List<PropertyValue> original;
	 // 若果为MutablePropertyValues类型
   if (pvs instanceof MutablePropertyValues) {
      mpvs = (MutablePropertyValues) pvs;
      // 是否已经转化
      if (mpvs.isConverted()) {
         // Shortcut: use the pre-converted values as-is.
         try {
            bw.setPropertyValues(mpvs);
            return;
         }
         catch (BeansException ex) {
            throw new BeanCreationException(
                  mbd.getResourceDescription(), beanName, "Error setting property values", ex);
         }
      }
      // 获取属性值
      original = mpvs.getPropertyValueList();
   }
   else {
      original = Arrays.asList(pvs.getPropertyValues());
   }

   TypeConverter converter = getCustomTypeConverter();
   if (converter == null) {
      converter = bw;
   }
   BeanDefinitionValueResolver valueResolver = new BeanDefinitionValueResolver(this, beanName, mbd, converter);

   // Create a deep copy, resolving any references for values.
   List<PropertyValue> deepCopy = new ArrayList<>(original.size());
   boolean resolveNecessary = false;
   for (PropertyValue pv : original) {
      // 单个属性是否已经转化了
      if (pv.isConverted()) {
         deepCopy.add(pv);
      }
      else {
         // 属性名称
         String propertyName = pv.getName();
         // 原始值
         Object originalValue = pv.getValue();
         if (originalValue == AutowiredPropertyMarker.INSTANCE) {
            // 获取writeMethod
            Method writeMethod = bw.getPropertyDescriptor(propertyName).getWriteMethod();
            if (writeMethod == null) {
               throw new IllegalArgumentException("Autowire marker for property without write method: " + pv);
            }
            originalValue = new DependencyDescriptor(new MethodParameter(writeMethod, 0), true);
         }
         // 解析的值
         Object resolvedValue = valueResolver.resolveValueIfNecessary(pv, originalValue);
         // 转化的值
         Object convertedValue = resolvedValue;
         boolean convertible = bw.isWritableProperty(propertyName) &&
               !PropertyAccessorUtils.isNestedOrIndexedProperty(propertyName);
         // 可转化
         if (convertible) {
            // 
            convertedValue = convertForProperty(resolvedValue, propertyName, bw, converter);
         }
         // Possibly store converted value in merged bean definition,
         // in order to avoid re-conversion for every created bean instance.
         if (resolvedValue == originalValue) {
            if (convertible) {
               pv.setConvertedValue(convertedValue);
            }
            deepCopy.add(pv);
         }
         else if (convertible && originalValue instanceof TypedStringValue &&
               !((TypedStringValue) originalValue).isDynamic() &&
               !(convertedValue instanceof Collection || ObjectUtils.isArray(convertedValue))) {
            pv.setConvertedValue(convertedValue);
            deepCopy.add(pv);
         }
         else {
            resolveNecessary = true;
            deepCopy.add(new PropertyValue(pv, convertedValue));
         }
      }
   }
   if (mpvs != null && !resolveNecessary) {
      // 标识已经被转化
      mpvs.setConverted();
   }

   // Set our (possibly massaged) deep copy.
   try {
      bw.setPropertyValues(new MutablePropertyValues(deepCopy));
   }
   catch (BeansException ex) {
      throw new BeanCreationException(
            mbd.getResourceDescription(), beanName, "Error setting property values", ex);
   }
}
```

> 该方式就是把已经解析好的值进行赋值操作：
> 1.xml中解析的属性值
> 2.<bean autowire="byType || byName" />时（xml没有相关属性定义），非简单类型的值的自动注入

到这里整个populateBean()方法以及介绍完毕，属性赋值包含：xml配置的