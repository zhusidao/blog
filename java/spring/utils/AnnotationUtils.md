# getAnnotation、findAnnotation、getMergedAnnotation、findMergedAnnotation、isAnnotated、hasAnnotation

# 前言

介绍几个常用spring注解相关方法getAnnotation、findAnnotation、getMergedAnnotation、findMergedAnnotation、isAnnotated、hasAnnotation的区别，这里做个总结。

| 方法                 | 描述                                                    |
| -------------------- | ------------------------------------------------------- |
| getAnnotation        | 获取当前类指定的注解                                    |
| findAnnotation       | 获取当前类及其父类上面的注解                            |
| getMergedAnnotation  | 获取当前类的指定的注解，并合并@AliasFor相关的属性       |
| findMergedAnnotation | 获取当前类及其父类指定的注解，并合并@AliasFor相关的属性 |
| isAnnotated          | 当前类是否含有指定的注解                                |
| hasAnnotation        | 当前类及其父类是否含有指定的注解                        |

### 下面给出测试demo进行差异理解

首先声明下面2个注解

RequestMapping1.class

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface RequestMapping1 {

   String name() default "";

   @AliasFor("path")
   String[] value() default {};

   @AliasFor("value")
   String[] path() default {};

   RequestMethod[] method() default {};

   String[] params() default {};

   String[] headers() default {};

   String[] consumes() default {};

   String[] produces() default {};
}
```

PostMapping1.class

```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@RequestMapping1(method = RequestMethod.POST)
public @interface PostMapping1 {

   @AliasFor(annotation = RequestMapping1.class)
   String name() default "";

   @AliasFor(annotation = RequestMapping1.class)
   String[] value() default {};

   @AliasFor(annotation = RequestMapping1.class)
   String[] path() default {};

   @AliasFor(annotation = RequestMapping1.class)
   String[] params() default {};

   @AliasFor(annotation = RequestMapping1.class)
   String[] headers() default {};

   @AliasFor(annotation = RequestMapping1.class)
   String[] consumes() default {};

   @AliasFor(annotation = RequestMapping1.class)
   String[] produces() default {};
}
```

#### 1.当ParentController上有@RequestMapping1注解，ChildController无注解

```java
public class Test{
    @RequestMapping1(name = "parent", path="parent/controller", produces = "ggogogo")
    static class ParentController {
    }

//    @PostMapping1(name="child", value = "child/controller",  consumes = "application/json")
 //   @RequestMapping1(name = "parent", path="parent/controller", produces = "ggogogo")
    class ChildController extends ParentController {
    }

    public static void main(String[] args) {
        System.out.println("ParentController isAnnotated @RequestMapping: " + AnnotatedElementUtils.isAnnotated(ParentController.class, RequestMapping1.class));
        System.out.println("ChildController isAnnotated @RequestMapping: " + AnnotatedElementUtils.isAnnotated(ChildController.class, RequestMapping1.class));
        System.out.println("ParentController hasAnnotation @RequestMapping: " + AnnotatedElementUtils.hasAnnotation(ParentController.class, RequestMapping1.class));
        System.out.println("ChildController hasAnnotation @RequestMapping: " + AnnotatedElementUtils.hasAnnotation(ChildController.class, RequestMapping1.class));
        System.out.println();

        System.out.println("ParentController getAnnotation @RequestMapping: " + AnnotationUtils.getAnnotation(ParentController.class, RequestMapping1.class));
        System.out.println("ChildController getAnnotation @RequestMapping: " + AnnotationUtils.getAnnotation(ChildController.class, RequestMapping1.class));
        System.out.println();

        System.out.println("ParentController findAnnotation @RequestMapping: " + AnnotationUtils.findAnnotation(ParentController.class, RequestMapping1.class));
        System.out.println("ParentController findAnnotation @RequestMapping: " + AnnotationUtils.findAnnotation(ChildController.class, RequestMapping1.class));
        System.out.println();

        System.out.println("ParentController getMergedAnnotation @RequestMapping: " + AnnotatedElementUtils.getMergedAnnotation(ParentController.class, RequestMapping1.class));
        System.out.println("ChildController getMergedAnnotation @RequestMapping: " + AnnotatedElementUtils.getMergedAnnotation(ChildController.class, RequestMapping1.class));
        System.out.println();

        System.out.println("ParentController findMergedAnnotation @RequestMapping: " + AnnotatedElementUtils.findMergedAnnotation(ParentController.class, RequestMapping1.class));
        System.out.println("ChildController findMergedAnnotation @RequestMapping: " + AnnotatedElementUtils.findMergedAnnotation(ChildController.class, RequestMapping1.class));
        System.out.println();
    }

}
```

测试结果如下:

```java
ParentController isAnnotated @RequestMapping: true
ChildController isAnnotated @RequestMapping: false
ParentController hasAnnotation @RequestMapping: true
ChildController hasAnnotation @RequestMapping: true

ParentController getAnnotation @RequestMapping: @org.example.controller.RequestMapping1(consumes=[], headers=[], method=[], name=parent, params=[], path=[parent/controller], produces=[ggogogo], value=[parent/controller])
ChildController getAnnotation @RequestMapping: null

ParentController findAnnotation @RequestMapping: @org.example.controller.RequestMapping1(consumes=[], headers=[], method=[], name=parent, params=[], path=[parent/controller], produces=[ggogogo], value=[parent/controller])
ParentController findAnnotation @RequestMapping: @org.example.controller.RequestMapping1(consumes=[], headers=[], method=[], name=parent, params=[], path=[parent/controller], produces=[ggogogo], value=[parent/controller])

ParentController getMergedAnnotation @RequestMapping: @org.example.controller.RequestMapping1(consumes=[], headers=[], method=[], name=parent, params=[], path=[parent/controller], produces=[ggogogo], value=[parent/controller])
ChildController getMergedAnnotation @RequestMapping: null

ParentController findMergedAnnotation @RequestMapping: @org.example.controller.RequestMapping1(consumes=[], headers=[], method=[], name=parent, params=[], path=[parent/controller], produces=[ggogogo], value=[parent/controller])
ChildController findMergedAnnotation @RequestMapping: @org.example.controller.RequestMapping1(consumes=[], headers=[], method=[], name=parent, params=[], path=[parent/controller], produces=[ggogogo], value=[parent/controller])

```

> 说明has、find类别开头的几个方法能运用到当前类及其父类上的注解。
>
> is、get类别开头的几个方法都只能运用到当前类上面的注解。

#### 2.父类没有注解，子类有@RequestMapping1注解

```java
public class Test{
    //@RequestMapping1(name = "parent", path="parent/controller", produces = "ggogogo")
    static class ParentController {
    }

    @RequestMapping1(name = "parent", path="parent/controller", produces = "ggogogo")
    class ChildController extends ParentController {
    }
    // 输出代码忽略，和上面输出代码一样
}
```

输出结果如下

```java
ParentController isAnnotated @RequestMapping: false
ChildController isAnnotated @RequestMapping: true
ParentController hasAnnotation @RequestMapping: false
ChildController hasAnnotation @RequestMapping: true

ParentController getAnnotation @RequestMapping: null
ChildController getAnnotation @RequestMapping: @org.example.controller.RequestMapping1(consumes=[], headers=[], method=[], name=parent, params=[], path=[parent/controller], produces=[ggogogo], value=[parent/controller])

ParentController findAnnotation @RequestMapping: null
ParentController findAnnotation @RequestMapping: @org.example.controller.RequestMapping1(consumes=[], headers=[], method=[], name=parent, params=[], path=[parent/controller], produces=[ggogogo], value=[parent/controller])

ParentController getMergedAnnotation @RequestMapping: null
ChildController getMergedAnnotation @RequestMapping: @org.example.controller.RequestMapping1(consumes=[], headers=[], method=[], name=parent, params=[], path=[parent/controller], produces=[ggogogo], value=[parent/controller])

ParentController findMergedAnnotation @RequestMapping: null
ChildController findMergedAnnotation @RequestMapping: @org.example.controller.RequestMapping1(consumes=[], headers=[], method=[], name=parent, params=[], path=[parent/controller], produces=[ggogogo], value=[parent/controller])

```

#### 3.父类上有@PostMapping1，子类上无注解

```java
public class Test{
		@PostMapping1(name = "parent", path="parent/controller", produces = "ggogogo")
    static class ParentController {
    }

    //@RequestMapping1(name = "parent", path="parent/controller", produces = "ggogogo")
    class ChildController extends ParentController {
    }
    // 输出代码忽略，和上面输出代码一样
}
```

输出结果如下

```java
ParentController isAnnotated @RequestMapping: true
ChildController isAnnotated @RequestMapping: false
ParentController hasAnnotation @RequestMapping: true
ChildController hasAnnotation @RequestMapping: true

ParentController getAnnotation @RequestMapping: @org.example.controller.RequestMapping1(consumes=[], headers=[], method=[POST], name=, params=[], path=[], produces=[], value=[])
ChildController getAnnotation @RequestMapping: null

ParentController findAnnotation @RequestMapping: @org.example.controller.RequestMapping1(consumes=[], headers=[], method=[POST], name=, params=[], path=[], produces=[], value=[])
ParentController findAnnotation @RequestMapping: @org.example.controller.RequestMapping1(consumes=[], headers=[], method=[POST], name=, params=[], path=[], produces=[], value=[])

ParentController getMergedAnnotation @RequestMapping: @org.example.controller.RequestMapping1(consumes=[], headers=[], method=[POST], name=parent, params=[], path=[parent/controller], produces=[ggogogo], value=[parent/controller])
ChildController getMergedAnnotation @RequestMapping: null

ParentController findMergedAnnotation @RequestMapping: @org.example.controller.RequestMapping1(consumes=[], headers=[], method=[POST], name=parent, params=[], path=[parent/controller], produces=[ggogogo], value=[parent/controller])
ChildController findMergedAnnotation @RequestMapping: @org.example.controller.RequestMapping1(consumes=[], headers=[], method=[POST], name=parent, params=[], path=[parent/controller], produces=[ggogogo], value=[parent/controller])

```

> 因为PostMapping1注解上有RequestMapping1，所以也能获取@RequestMapping1注解。这里其实也能看出带Merged型的方法和非Merged型方法的区别:xxxMergedAnnotation方法有对@PostMapping1和其上面@RequestMapping1的属性进行合并（因为PostMapping1属性都有相关的@AliasFor注解进行对应）

#### 4.父类无注解，子类上有@PostMapping1注解

```java
public class Test{
//    @PostMapping1(name = "parent", path="parent/controller", produces = "ggogogo")
    static class ParentController {
    }

//    @PostMapping1(name="child", value = "child/controller",  consumes = "application/json")
//    @RequestMapping1(name = "parent", path="parent/controller", produces = "ggogogo")
    @PostMapping1(name = "parent", path="parent/controller", produces = "ggogogo")
    class ChildController extends ParentController {
    }
 	  // 输出代码忽略，和上面输出代码一样
}
```

输出结果如下

```java
ParentController isAnnotated @RequestMapping: false
ChildController isAnnotated @RequestMapping: true
ParentController hasAnnotation @RequestMapping: false
ChildController hasAnnotation @RequestMapping: true

ParentController getAnnotation @RequestMapping: null
ChildController getAnnotation @RequestMapping: @org.example.controller.RequestMapping1(consumes=[], headers=[], method=[POST], name=, params=[], path=[], produces=[], value=[])

ParentController findAnnotation @RequestMapping: null
ParentController findAnnotation @RequestMapping: @org.example.controller.RequestMapping1(consumes=[], headers=[], method=[POST], name=, params=[], path=[], produces=[], value=[])

ParentController getMergedAnnotation @RequestMapping: null
ChildController getMergedAnnotation @RequestMapping: @org.example.controller.RequestMapping1(consumes=[application/json], headers=[], method=[POST], name=child, params=[], path=[child/controller], produces=[], value=[child/controller])

ParentController findMergedAnnotation @RequestMapping: null
ChildController findMergedAnnotation @RequestMapping: @org.example.controller.RequestMapping1(consumes=[application/json], headers=[], method=[POST], name=child, params=[], path=[child/controller], produces=[], value=[child/controller])

```

#### 5.父类有@RequestMapping1注解、子类有@PostMapping1注解

```java
public class Test{
	  @RequestMapping1(name = "parent", path="parent/controller", produces = "ggogogo")
    static class ParentController {
    }

    @PostMapping1(name="child", value = "child/controller",  consumes = "application/json")
//    @RequestMapping1(name = "parent", path="parent/controller", produces = "ggogogo")
//    @PostMapping1(name = "parent", path="parent/controller", produces = "ggogogo")
    class ChildController extends ParentController {
    }
    
    // 输出代码忽略，和上面输出代码一样
}
```

输出结果如下

```java
ParentController isAnnotated @RequestMapping: true
ChildController isAnnotated @RequestMapping: true
ParentController hasAnnotation @RequestMapping: true
ChildController hasAnnotation @RequestMapping: true

ParentController getAnnotation @RequestMapping: @org.example.controller.RequestMapping1(consumes=[], headers=[], method=[], name=parent, params=[], path=[parent/controller], produces=[ggogogo], value=[parent/controller])
ChildController getAnnotation @RequestMapping: @org.example.controller.RequestMapping1(consumes=[], headers=[], method=[POST], name=, params=[], path=[], produces=[], value=[])

ParentController findAnnotation @RequestMapping: @org.example.controller.RequestMapping1(consumes=[], headers=[], method=[], name=parent, params=[], path=[parent/controller], produces=[ggogogo], value=[parent/controller])
ParentController findAnnotation @RequestMapping: @org.example.controller.RequestMapping1(consumes=[], headers=[], method=[POST], name=, params=[], path=[], produces=[], value=[])

ParentController getMergedAnnotation @RequestMapping: @org.example.controller.RequestMapping1(consumes=[], headers=[], method=[], name=parent, params=[], path=[parent/controller], produces=[ggogogo], value=[parent/controller])
ChildController getMergedAnnotation @RequestMapping: @org.example.controller.RequestMapping1(consumes=[application/json], headers=[], method=[POST], name=child, params=[], path=[child/controller], produces=[], value=[child/controller])

ParentController findMergedAnnotation @RequestMapping: @org.example.controller.RequestMapping1(consumes=[], headers=[], method=[], name=parent, params=[], path=[parent/controller], produces=[ggogogo], value=[parent/controller])
ChildController findMergedAnnotation @RequestMapping: @org.example.controller.RequestMapping1(consumes=[application/json], headers=[], method=[POST], name=child, params=[], path=[child/controller], produces=[], value=[child/controller])

```

> 这里可以看出，如果父类拥有@RequestMapping1，子类也拥有@RequestMapping1，获取到的为子类上的@RequestMapping1