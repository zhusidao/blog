# interrupt()中断对LockSupport.park()的影响

## 原理简单讲解

首先声明，本文不会去贴native方法的cpp实现，而是以伪代码的形式来理解这些native方法。

Thread对象的native实现里有一个成员代表线程的中断状态，我们可以认为它是一个boolean型的变量。初始为false。
Thread对象的native实现里有一个成员代表线程是否可以阻塞的许可permit，我们可以认为它是一个int型的变量，但它的值只能为0或1。当为1时，再累加也会维持1。初始为0。



## park/unpark实现的伪代码

下面将以伪代码的实现来说明**park/unpark**的实现。

```java
park() {
    if(permit > 0) {
        permit = 0;
        return;
    }

    if(中断状态 == true) {
        return;
    }
    
    阻塞当前线程;  // 将来会从这里被唤醒
    
    if(permit > 0) {
        permit = 0;
    }

}
```

可见，只要`permit`为1或者`中断状态`为true，那么执行`park`就不能够阻塞线程。`park`只可能消耗掉`permit`，但不会去消耗掉`中断状态`。

```java
unpark(Thread thread) {
    if(permit < 1) {
        permit = 1;
        if(thread处于阻塞状态)
            唤醒线程thread;
    }
}
```

`unpark`一定会将`permit`置为1，如果线程阻塞，再将其唤醒。从实现可见，无论调用几次`unpark`，`permit`只能为1。



## interrupt()

##### interrupt()实现的伪代码

```java
interrupt(){
    if(中断状态 == false) {
        中断状态 = true;
    }
    unpark(this);    //注意这是Thread的成员方法，所以我们可以通过this获得Thread对象
}
```

**sleep()伪代码实现**

```java
sleep(){//这里我忽略了参数，假设参数是大于0的即可
    if(中断状态 == true) {
        中断状态 = false;
        throw new InterruptedException();
    }
    

    线程开始睡觉;   
    
    if(中断状态 == true) {
        中断状态 = false;
        throw new InterruptedException();
    }

}
```

**总结**

- park调用后一定会消耗掉permit，无论unpark操作先做还是后做。
- 如果中断状态为true，那么park无法阻塞。
- unpark会使得permit为1，并唤醒处于阻塞的线程。
- interrupt()会使得中断状态为true，并调用unpark。
- sleep() / wait() / join()调用后一定会消耗掉中断状态，无论interrupt()操作先做还是后做。
  

参考：https://blog.csdn.net/anlian523/article/details/106752414