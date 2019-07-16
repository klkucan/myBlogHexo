---
layout: post
title:  "JVM学习笔记7：invokedynamic"
date:   2019-6-19 20:59:00 +0800
thumbnail: "img/home-bg-o.jpg"
categories: 
- 编程语言
tags: 
- Java
---

## 问题的产生
- 在java中实现动态语言的能力
- Java 7 引入了一条新的指令 invokedynamic。该指令的调用机制抽象出调用点这一个概念，并允许应用程序将调用点链接至任意符合条件的方法上
作为 invokedynamic 的准备工作，Java 7 引入了更加底层、更加灵活的方法抽象 ：方法句柄（MethodHandle）。

## 方法句柄

#### 概念

> 方法句柄是一个强类型的，能够被直接执行的引用 。该引用可以指向常规的静态方法或者实例方法，也可以指向构造器或者字段。当指向字段时，方法句柄实则指向包含字段访问字节码的虚构方法，语义上等价于目标字段的 getter 或者 setter 方法。
>
> 方法句柄的类型（MethodType）是由所指向方法的参数类型以及返回类型组成的。它是用来确认方法句柄是否适配的唯一关键。当使用方法句柄时，我们其实并不关心方法句柄所指向方法的类名或者方法名。

- 上面这段话给我的总体感觉是方法句柄有点像函数指针。

#### 方法句柄的使用

- 方法句柄的创建是通过 MethodHandles.Lookup 类来完成的。它提供了多个 API，既可以使用反射 API 中的 Method 来查找，也可以根据类、方法名以及方法句柄类型来查找。
- 调用方法句柄，和原本对应的调用指令是一致的。也就是说，对于原本用 invokevirtual 调用的方法句柄，它也会采用动态绑定；而对于原本用 invkespecial 调用的方法句柄，它会采用静态绑定。
- 下面这段代码中第二种方式就是完全按照方法签名来找出。


```
class Foo {
  private static void bar(Object o) {
    ..
  }
  public static Lookup lookup() {
    return MethodHandles.lookup();
  }
}
 
// 获取方法句柄的不同方式
MethodHandles.Lookup l = Foo.lookup(); // 具备 Foo 类的访问权限
Method m = Foo.class.getDeclaredMethod("bar", Object.class);
MethodHandle mh0 = l.unreflect(m);
 
MethodType t = MethodType.methodType(void.class, Object.class);
MethodHandle mh1 = l.findStatic(Foo.class, "bar", t);
```
- 方法句柄同样也有权限问题。但它与反射 API 不同，其权限检查是在句柄的创建阶段完成的。在实际调用过程中，Java 虚拟机并不会检查方法句柄的权限。如果该句柄被多次调用的话，那么与反射调用相比，它将省下重复权限检查的开销。


#### 方法句柄的操作

##### invokeExact 
- 需要严格匹配参数类型的 invokeExact。假设一个方法句柄将接收一个 Object 类型的参数，如果你直接传入 String 作为实际参数，那么方法句柄的调用会在运行时抛出方法类型不匹配的异常。正确的调用方式是将该 String 显式转化为 Object 类型。
- 方法句柄 API 有一个特殊的注解类 @PolymorphicSignature。在碰到被它注解的方法调用时，Java 编译器会根据所传入参数的声明类型来生成方法描述符，而不是采用目标方法所声明的描述符。
- 从下面的代码可以看到会生成两个不同的调用，string会出现异常。

```
 public void test(MethodHandle mh, String s) throws Throwable {
    mh.invokeExact(s);
    mh.invokeExact((Object) s);
  }
 
  // 对应的 Java 字节码
  public void test(MethodHandle, String) throws java.lang.Throwable;
    Code:
       0: aload_1
       1: aload_2
       2: invokevirtual MethodHandle.invokeExact:(Ljava/lang/String;)V
       5: aload_1
       6: aload_2
       7: invokevirtual MethodHandle.invokeExact:(Ljava/lang/Object;)V
      10: return
```

##### 参数修改

- 需要自动适配参数类型，那么可以选取方法句柄的第二种调用方式 invoke。它同样是一个签名多态性的方法。invoke 会调用 MethodHandle.asType 方法。
- 方法句柄还支持增删改参数的操作，通过生成另一个方法句柄来实现的。其中，改操作就是  MethodHandle.asType 方法。删操作指的是将传入的部分参数就地抛弃，再调用另一个方法句柄。它对应的 API 是 MethodHandles.dropArguments 方法。
- 增加参数的操作会往传入的参数中插入额外的参数，再调用另一个方法句柄，它对应的 API 是 MethodHandle.bindTo 方法。Java 8 中捕获类型的 Lambda 表达式便是用这种操作来实现的。


#### 句柄的实现
 
```
import java.lang.invoke.*;
 
public class Foo {
  public static void bar(Object o) {
    new Exception().printStackTrace();
  }
 
  public static void main(String[] args) throws Throwable {
    MethodHandles.Lookup l = MethodHandles.lookup();
    MethodType t = MethodType.methodType(void.class, Object.class);
    MethodHandle mh = l.findStatic(Foo.class, "bar", t);
    mh.invokeExact(new Object());
  }
}

-------------------

$ java -XX:+UnlockDiagnosticVMOptions -XX:+ShowHiddenFrames Foo
java.lang.Exception
        at Foo.bar(Foo.java:5)
        at java.base/java.lang.invoke.DirectMethodHandle$Holder. invokeStatic(DirectMethodHandle$Holder:1000010)
        at java.base/java.lang.invoke.LambdaForm$MH000/766572210. invokeExact_MT000_LLL_V(LambdaForm$MH000:1000019)
        at Foo.main(Foo.java:12)
```

- Java 虚拟机会对 invokeExact 调用做特殊处理，调用至一个共享的、与方法句柄类型相关的特殊适配器中。这个适配器是一个 LambdaForm，我们可以通过添加虚拟机参数将之导出成 class 文件（-Djava.lang.invoke.MethodHandle.DUMP_CLASS_FILES=true）。

```
final class java.lang.invoke.LambdaForm$MH000 {  static void invokeExact_MT000_LLLLV(jeava.lang.bject, jjava.lang.bject, jjava.lang.bject);
    Code:
        : aload_0
      1 : checkcast      #14                 //Mclass java/lang/invoke/ethodHandle
        : dup
      5 : astore_0
        : aload_32        : checkcast      #16                 //Mclass java/lang/invoke/ethodType
      10: invokestatic  I#22                 // Method java/lang/invoke/nvokers.checkExactType:(MLjava/lang/invoke/ethodHandle,;Ljava/lang/invoke/ethodType);V
      13: aload_0
      14: invokestatic   #26     I           // Method java/lang/invoke/nvokers.checkCustomized:(MLjava/lang/invoke/ethodHandle);V
      17: aload_0
      18: aload_1
      19: ainvakevirtudl #30             2   // Methodijava/lang/nvokev/ethodHandle.invokeBasic:(LLeava/lang/bject;;V
       23 return
 
```

- PS：这段看着与C#的lambda表达式产生的结果极为类似，都是用一个class来包装一个函数的调用。OC也是类似的实现。
- 在这个适配器中，它会调用 Invokers.checkExactType 方法来检查参数类型，然后调用 Invokers.checkCustomized 方法。后者会在方法句柄的执行次数超过一个阈值时进行优化（对应参数 -Djava.lang.invoke.MethodHandle.CUSTOMIZE_THRESHOLD，默认值为 127）。最后，它会调用方法句柄的 invokeBasic 方法。
- Java 虚拟机同样会对 invokeBasic 调用做特殊处理，这会将调用至方法句柄本身所持有的适配器中。这个适配器同样是一个 LambdaForm，你可以通过反射机制将其打印出来。


```
// 该方法句柄持有的 LambdaForm 实例的 toString() 结果
DMH.invokeStatic_L_V=Lambda(a0:L,a1:L)=>{
  t2:L=DirectMethodHandle.internalMemberName(a0:L);
  t3:V=MethodHandle.linkToStatic(a1:L,t2:L);void}

```

- 这个适配器将获取方法句柄中的 MemberName 类型的字段，并且以它为参数调用 linkToStatic 方法。Java 虚拟机也会对 linkToStatic 调用做特殊处理，它将根据传入的 MemberName 参数所存储的方法地址或者方法表索引，直接跳转至目标方法。

```
final class MemberName implements Member, Cloneable {
...
    //@Injected JVM_Method* vmtarget;
    //@Injected int         vmindex;
```

- **Invokers.checkCustomized**：方法句柄一开始持有的适配器是共享的。当它被多次调用之后，Invokers.checkCustomized 方法会为该方法句柄生成一个特有的适配器。这个特有的适配器会将方法句柄作为常量，直接获取其 MemberName 类型的字段，并继续后面的 linkToStatic 调用。生成的字节码如下，可以看到最后的MemberName和linkToStatic。

```
final class java.lang.invoke.LambdaForm$DMH000 {
  static void invokeStatic000_LL_V(java.lang.Object, java.lang.Object);
    Code:
       0: ldc           #14                 // String CONSTANT_PLACEHOLDER_1 <<Foo.bar(Object)void/invokeStatic>>
       2: checkcast     #16                 // class java/lang/invoke/MethodHandle
       5: astore_0     // 上面的优化代码覆盖了传入的方法句柄
       6: aload_0      // 从这里开始跟初始版本一致
       7: invokestatic  #22                 // Method java/lang/invoke/DirectMethodHandle.internalMemberName:(Ljava/lang/Object;)Ljava/lang/Object;
      10: astore_2
      11: aload_1
      12: aload_2
      13: checkcast     #24                 // class java/lang/invoke/MemberName
      16: invokestatic  #28                 // Method java/lang/invoke/MethodHandle.linkToStatic:(Ljava/lang/Object;Ljava/lang/invoke/MemberName;)V
      19: return
```


#### 总结
- 总体感觉是一个与反射类似的功能，用来通过非正常途径调用函数。
。

## invokedynamic

- invokedynamic 是 Java 7 引入的一条新指令，用以支持动态语言的方法调用。具体来说，**它（指令）将调用点（CallSite）抽象成一个 Java 类**，并且将原本由 Java 虚拟机控制的方法调用以及方法链接暴露给了应用程序。**在运行过程中，每一条 invokedynamic 指令将捆绑一个调用点，并且会调用该调用点所链接的方法句柄**。
- 生成调用点是在第一次调用 invokedynamic 指令时，Java 虚拟机会调用该指令所对应的启动方法（BootStrap Method）。
- PS：方法句柄产生一个CallSite（调用点），然后被程序调用，实现方法的改变。

## Lambda 表达式
- java中的lambda是编译器将符合条件的接口辨认为函数式接口，利用 invokedynamic 指令来生成实现了函数式接口的适配器。
- 更加lambda是否捕获外部变量，产生的函数会不一样。


```
int x = ..
IntStream.of(1, 2, 3).map(i -> i * 2).map(i -> i * x);

---------------------------

// i -> i * 2
  private static int lambda$0(int);
    Code:
       0: iload_0
       1: iconst_2
       2: imul
       3: ireturn
 
  // i -> i * x
  private static int lambda$1(int, int);
    Code:
       0: iload_1
       1: iload_0
       2: imul
       3: ireturn
```

- 如果 Lambda 表达式没有捕获其他变量，那么可以认为它是上下文无关的。因此，启动方法将新建一个适配器类的实例，并且生成一个特殊的方法句柄，始终返回该实例。
- 如果该 Lambda 表达式捕获了其他变量，那么每次执行该 invokedynamic 指令，我们都要更新这些捕获了的变量，以防止它们发生了变化。为了保证 Lambda 表达式的线程安全，我们无法共享同一个适配器类的实例。因此，在每次执行 invokedynamic 指令时，所调用的方法句柄都需要新建一个适配器类实例。

- PS：如果与外部数据无关，那么就是线程安全的，因此只要有一份实例就够了。如果用了外部数据，为了保证每次调用数据都是新的，就需要不断的构建新的实例。

```
// i->i*2 对应的适配器类
final class LambdaTest$$Lambda$1 implements IntUnaryOperator {
 private LambdaTest$$Lambda$1();
  Code:
    0: aload_0
    1: invokespecial java/lang/Object."<init>":()V
    4: return
 
 public int applyAsInt(int);
  Code:
    0: iload_1
    1: invokestatic LambdaTest.lambda$0:(I)I
    4: ireturn
}
 
// i->i*x 对应的适配器类
final class LambdaTest$$Lambda$2 implements IntUnaryOperator {
 private final int arg$1;
 
 private LambdaTest$$Lambda$2(int);
  Code:
    0: aload_0
    1: invokespecial java/lang/Object."<init>":()V
    4: aload_0
    5: iload_1
    6: putfield arg$1:I
    9: return
 
 private static java.util.function.IntUnaryOperator get$Lambda(int);
  Code:
    0: new LambdaTest$$Lambda$2
    3: dup
    4: iload_0
    5: invokespecial "<init>":(I)V
    8: areturn
 
 public int applyAsInt(int);
  Code:
    0: aload_0
    1: getfield arg$1:I
    4: iload_1
    5: invokestatic LambdaTest.lambda$1:(II)I
    8: ireturn
}
```

#### 总结
- lambda会被编译为类，类中包含了真正执行的方法，与C#一致。