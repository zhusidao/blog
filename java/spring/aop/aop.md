# Spring AOP pointcut的 this target within的区别:

这里我们直接结合例子， 分别对重写run()和不重写run()进行验证

```java
interface Vehicle{
    default void run(){};
}

@Component
class VehicleImpl implements Vehicle{
    @Override
    public void run() {
    }
}
```

```java
@Aspect
@ComponentScan
@EnableAspectJAutoProxy
public class PointcutTest {
    // 验证this
    @Before("this(Vehicle)")
    public void thisVehicle() {
        System.out.println("thisVehicle...");
    }

    @Before("this(VehicleImpl)")
    public void thisVehicleImpl() {
        System.out.println("thisVehicleImpl...");
    }
    // 验证target
    @Before("target(Vehicle)")
    public void targetVehicle() {
        System.out.println("targetVehicle...");
    }

    @Before("target(VehicleImpl)")
    public void targetVehicleImpl() {
        System.out.println("targetVehicleImpl...");
    }
    // 验证within
    @Before("within(Vehicle)")
    public void withinVehicle() {
        System.out.println("withinVehicle...");
    }

    @Before("within(Vehicle+)")
    public void withinWithinAdd() {
        System.out.println("withinWithinAdd...");
    }

    @Before("within(VehicleImpl)")
    public void withinVehicleImpl() {
        System.out.println("withinVehicleImpl...");
    }
  
    public static void main(String[] args) {
        ApplicationContext context = new AnnotationConfigApplicationContext(PointcutTest.class);
        Vehicle vehicle = (Vehicle) context.getBean("vehicleImpl");
        vehicle.run();
    }
}
```

> 重写run()的执行结果
>
> targetVehicle...
> targetVehicleImpl...
> thisVehicle...
> withinVehicleImpl...
> withinWithinAdd...
>
> 未重写run()的执行结果
>
> targetVehicle...
> targetVehicleImpl...
> thisVehicle...
> withinWithinAdd...

总结：

1.this(VehicleImp)匹配不到,因为context.getBean()获得的对象v1是个被包装过的代理对象.不是VehicleImp类型.

![image-20200824153747926](/Users/zhusidao/Library/Application Support/typora-user-images/image-20200824153747926.png)

target(VehicleImp)能匹配到,匹配的是被代理的实际(VehicleImp)对象.

2.代理类型和被代理类型都符合接口Vehicle,this(Vehicle)和target(Vehicle)都能匹配到.

3.within和target的目标对象一样是实际被代理的对象(本文中VehicleImp).within不支持继承关系.所以within(Vehicle)没匹配到.可以使用+,within(Vehicle+)匹配.