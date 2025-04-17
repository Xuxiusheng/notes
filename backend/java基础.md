# Java基础

## Java vs C++

- Java只支持单继承，但可以实现多个接口，弥补单继承的短板，而C++支持多继承
  - 多继承本身存在菱形问题，B和C继承了A，D继承了B和C，如果B和C重写了A的方法，在D中调用时会存在歧义
  - 多继承会让类之间的关系变得复杂，难以维护
- Java的内存管理是由垃圾回收线程在后台自动处理的，C++的内存管理需要开发者手动维护
- Java具有平台无关性，字节码可以跨平台使用
- Java只支持方法重载，不支持运算符重载，C++两者都支持

## 多接口实现不存在菱形问题

多实现不会存在菱形问题，因为一个接口中的方法，因为在实现类中，这个方法是一定要被重写的，也就是说调用的方法一定是自己重写的方法，不会存在歧义

JDK9引入了接口默认方法，默认方法存在菱形问题，默认方法特例：如果A实现了接口B，C继承了A，同时实现了B，那调用方法时，调用的是重写过的方法；如果一个类实现了A和B两个接口，两个接口有相同的默认方法，那类必须重写这个方法，否则存在歧义

如果A、B两个接口有同名方法，参数也相同，只有返回值不同，这样一个类是不能同时实现A和B的，因为方法签名只由方法名和参数决定，返回值并不属于签名的一部分，这种情况下编译器会报错

## JVM的作用

JVM是运行Java字节码的虚拟机，负责将编译器的编译结果也就是字节码加载进内存并转化成运行时数据结构，并对字节码逐行解释运行，字节码+不同系统的JVM的特定实现是实现Java语言平台无关性的关键所在

## 采用字节码的好处

- 字节码相对于Java源代码更接近计算机底层，执行效率更高
- JVM可以识别字节码，但不能识别Java源代码，字节码+JVM可以实现一次编译，处处运行

字节码保留了解释型语言执行效率低的问题，也保留了解释型语言可移植的优点

![java编译解释流程](https://github.com/Xuxiusheng/notes/blob/main/photos/compile-pipeline.png)

## 编译器

- AOT: 一次性将字节码编译成机器码，启动时间以及内存占用都很小，但是峰值性能相对较弱，而且不支持Java的动态特性，比如反射
- JIT：逐行解释字节码，在运行时会收集运行信息，针对热点代码进行编译，由于会收集运行状态信息，极限处理能力更强，但是冷启动问题以及内存占用问题比较严重

如何确定是否为热点代码：JIT会统计方法或者代码块在一定时间内的调用次数，如果超过了给定的阈值，就认为是热点代码对其编译，编译的代码存入codeCache，位于方法区

### JVM编译器

- Client编译器：注重启动速度和局部优化
  - C1编译器：启动速度快，但是性能要差一些，进行一些简单可靠的优化，比如常量传播和常量折叠（常量传播是指使用常量替换原有的变量，常量折叠是直接计算表达式结果）

- Server编译器：注重全局优化，会进行一些不可靠的激进优化，冷启动问题严重，适合长时间运行的后台程序，性能相比于Client编译器要更好
  - C2编译器：相比于C1编译器，会进行更深层次的优化，编译之后的代码质量更高，相对地，其编译耗时更久，但是代码的执行速度更快
  - Graal编译器：Graal相比于C2更加激进，倾向于基于预测进行激进优化，峰值性能更好，支持逃逸分析（一种确定指针作用范围的机制）

- 分层编译：结合C1和C2，追求启动速度和峰值性能的平衡，将JVM的执行状态分为5个层次
  - 常规的解释执行，无需提前编译，启动速度快，但执行效率低
  - 无运行状态收集的C1编译：会对代码进行简单的编译优化，编译速度快，优化程度低
  - 执行仅带有方法调用次数以及循环回边次数统计的C1编译
  - 收集运行状态的C1编译：不仅会对代码进行简单的编译优化，也会收集运行中详细的运行信息，为后续的深度优化提供依据
  - C2编译：根据收集的运行状态信息，对代码进行深度优化，编译时间长，但编译后的代码执行速度快

**以上两种编译器均属于JIT即时编译器**

### 优化：系统重启速度慢

重启负载很高，CPU使用率很高，然后一段时间后负载下降

- 问题原因：JIT编译器只有在方法调用次数达到阈值后才会将其编译存入codeCache，系统刚启动大部分代码是解释执行，所以负载很高，随着代码执行，热点代码被编译，系统负载下降
- 解决方案：启用分层编译，根据程序的运行状态动态调整编译策略（-XX:+TieredCompilation）

## Java的两大问题

- 冷启动问题：加载JVM -> JVM加载大量字节码文件 -> 代码执行，代码执行之前要进行大量的准备工作
- 运行时内存占用高：相比于提前编译，运行时编译的代码要多很多，造成无效的内存占用

解决以上问题的办法是采用提前编译AOT （静态编译）：程序运行之前进行编译，启动即巅峰，降低运行时内存占用

AOT是静态编译，不支持Java大量的动态运行时机制，比如反射，本质问题也就是将静态编译框架无法识别和处理的内容转化为其可识别的内容

### 静态编译识别动态特性

- 如果是Spring框架，可以采用社区开发的AOT Engine帮助解决
- 如果是非Spring框架，可以采用native-image-agent 的 Tracing Agent 来帮助解决动态内容无法被识别的问题

## 运算符

- a+=b与a=a+b的区别：前者存在隐式类型转换，后者如果不是int类型，需要强制转换
- 移位运算符：仅仅只有int和long支持移位，short、byte和char类型都会先转化为int再移位

## 基本类型与包装类型

### 区别

- 存储方式
- 内存占用
- 设计用途
- 默认值
- 比较方式

### 包装类型缓存机制

Byte、Short、Integer、Long和Character都支持缓存机制，其中Integer的**缓存上限**可以修改

**Float与Double不支持缓存机制**

- 相较于整型，浮点型没有比较明确的热点区间
- 存在语义歧义，由于计算机位宽的限制，会对尾数截断，精度丢失

### 浮点数精度丢失问题

计算机底层位宽有限，无法存储尾数过长的小数，会进行尾数截断，导致精度丢失

BigDecimal可以实现浮点数的运算，并且不存在精度丢失

注意：BigDecimal也是存在精度丢失的风险的，是因为double本身的限制，初始化BigDecimal必须使用String类型初始化

数值部分通过BigInteger表示，scale表示小数点位置，由于通过数组存储元素，性能相比于直接存储要差

## 接口与抽象类

- 共同点：实例化、抽象方法
- 不同点：
  - 设计目的
  - 继承和实现
  - 成员变量
  - 方法

## 变量

### 成员变量与局部变量

- 生命周期
- 存储位置
- 默认值
- 权限修饰符

### 静态变量

静态变量在类加载时分配内存，被类的所有实例共享，是否线程安全取决于变量是否是无状态变量

### 局部变量

局部变量不能被static修饰，局部变量仅作用于方法内部，而static修饰的变量是类范围可见的，容易造成访问范围混乱

## Object方法

```Java
/**
 * native 方法，用于返回当前运行时对象的 Class 对象，使用了 final 关键字修饰，故不允许子类重写。
 */
public final native Class<?> getClass()
/**
 * native 方法，用于返回对象的哈希码，主要使用在哈希表中，比如 JDK 中的HashMap。
 */
public native int hashCode()
/**
 * 用于比较 2 个对象的内存地址是否相等，String 类对该方法进行了重写以用于比较字符串的值是否相等。
 */
public boolean equals(Object obj)
/**
 * native 方法，用于创建并返回当前对象的一份拷贝。
 */
protected native Object clone() throws CloneNotSupportedException
/**
 * 返回类的名字实例的哈希码的 16 进制的字符串。建议 Object 所有的子类都重写这个方法。
 */
public String toString()
/**
 * native 方法，并且不能重写。唤醒一个在此对象监视器上等待的线程(监视器相当于就是锁的概念)。如果有多个线程在等待只会任意唤醒一个。
 */
public final native void notify()
/**
 * native 方法，并且不能重写。跟 notify 一样，唯一的区别就是会唤醒在此对象监视器上等待的所有线程，而不是一个线程。
 */
public final native void notifyAll()
/**
 * native方法，并且不能重写。暂停线程的执行。注意：sleep 方法没有释放锁，而 wait 方法释放了锁 ，timeout 是等待时间。
 */
public final native void wait(long timeout) throws InterruptedException
/**
 * 多了 nanos 参数，这个参数表示额外时间（以纳秒为单位，范围是 0-999999）。 所以超时的时间还需要加上 nanos 纳秒。。
 */
public final void wait(long timeout, int nanos) throws InterruptedException
/**
 * 跟之前的2个wait方法一样，只不过该方法一直等待，没有超时时间这个概念
 */
public final void wait() throws InterruptedException
/**
 * 实例被垃圾回收器回收的时候触发的操作
 */
protected void finalize() throws Throwable { }
```

## 反射

字节码文件属于静态存储结构，在类加载过程中，类加载器会将这个静态结构转化为运行时数据结构，这个就是这个字节码代表的类的Class对象，在运行时程序可以访问这个Class对象获取相关的字段和方法信息，从而调用实例对象的信息

```java
// 绕过成员变量的权限检查
package com.itheima.Test;

import java.lang.reflect.Field;

public class Test3 {
    public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException {
        PrivateClass pc = new PrivateClass();
        Class<?> cls = PrivateClass.class;
        Field attr = cls.getDeclaredField("privateId");
        attr.setAccessible(true);
        int value = (int) attr.get(pc);
        System.out.println(value);
    }
}
```

```java
// 绕过泛型类型检查

package com.itheima.Test;

import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.util.ArrayList;
import java.util.List;

public class Test3 {
    public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException, NoSuchMethodException, InvocationTargetException {
        List<Integer> list = new ArrayList<>();

        Class<?> clz = list.getClass();
        Method mt = clz.getDeclaredMethod("add", Object.class);
        mt.invoke(list, "123");
        for (Object o : list) {
            System.out.println(o);
        }
    }
}

```

## 注解

注解本质上是一种特殊的接口，它继承自`java.lang.annotation.Annotation`接口。通过`@interface`关键字来定义注解

可以用于提供额外的说明信息、编译检查和测试等

### 注解的三种保留策略

- Source：注解信息只会保留在源代码中，编译成字节码后会被丢弃，比如Override注解，只是用于在编译期检查方法是否被重写
- Class：默认策略，注解信息会保留在字节码文件中，但JVM加载到内存中时，不会读取注解信息
- Runtime：注解被jvm读取到内存中，运行时可以通过反射来获取和处理注解

**反射基于Runtime注解实现**

## 序列化

怎么转移对象到另一个JVM

- 序列化反序列化机制
- 共享数据库
- RPC远程调用

### Serializable 与 Externalizable

- Serializable只是一个标记接口，标识一个类可以被序列化，适用于简单的序列化场景
- Externalizable接口扩展了Serializable，开发者必须重写序列化和反序列化方法，灵活性更高，适用于追求性能的场景，避免无效的序列化内存占用

## 代理模式

代理模式的作用主要是控制对象的访问，可以增加一些自定义的检查逻辑

### 静态代理

必须为每一个目标类都创建一个代理对象，如果新增了方法，目标类和代理类都要修改，不灵活

### 动态代理

#### JDK动态代理

在JDK动态代理机制中
