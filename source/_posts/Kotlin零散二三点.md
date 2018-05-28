title: Kotlin零散二三点
date: 2018-05-25 00:00:00
categories:
- kotlin
tags:
- android
- kotlin
---
本篇主要谈些`Kotlin`开发过程比较常用到的技巧
<!--more-->
## 空安全处理
在Kotlin中, 最出名的特性莫过于就是它的`空安全`了, 毕竟`NPE`应该是大家最不想看到的错误信息.
我们先回顾下Kotlin如何处理`空安全`
> 我们有四种方法来避免NPE
> 1. 在条件中检查null
> 2. 安全调用使用?.
> 3. 使用Elvis 操作符 ?:
> 4. 使用!!操作符

当然关于第四点使用`!!`操作符, 他的本质就回归到了当遇到null的时候仍然会抛出NPE. 所以在非必须的情况下, 我们应该尽量避免使用`!!`

为了避免NPE, 在kotlin的类型系统中, 它做到的就是强制开发者明确定义目标类型是否是可空类型(通过`?`区别), 如果一个变量是可空的, 我们需要这样写
``` java
var nullpossible: String? = null
```
而像下面这种, 是永远不会编译通过的
``` java
var nullpossible: String = null
```
当我们在定义一个变量的时候, 当能够确保他是非空类型的时候就必须要在构造器中初始化, 然而这在实际开发中是非常不方便的.
``` kotlin
var a: String = ...
```
我们可以利用几种方式来解决
1. 使用`lateinit`延迟初始化
2. 使用[委托](https://www.kotlincn.net/docs/reference/delegation.html)`Delegates.notNull()`

``` java
var test: String by Delegates.notNull()
lateinit var testinit: String
```
不管我们使用哪种方式, 都可以让我们避免在初次定义类型的时候就必须初始化工作, 当然不论哪种方式, 在初始化前调用属性都是会抛出异常的.

而关于`Delegates.notNull()`, 通过源码我们可以看到他实际返回的是`NotNullVar`的委托.而通过`NotNullVar`中的`getValue()`返回直接定义为非空属性.
``` java
public object Delegates {
    public fun <T: Any> notNull(): ReadWriteProperty<Any?, T> = NotNullVar()
}
private class NotNullVar<T: Any>() : ReadWriteProperty<Any?, T> {
    private var value: T? = null

    public override fun getValue(thisRef: Any?, property: KProperty<*>): T {
        return value ?: throw IllegalStateException("Property ${property.name} should be initialized before get.")
    }

    public override fun setValue(thisRef: Any?, property: KProperty<*>, value: T) {
        this.value = value
    }
}
```
还有一种方式是通过委托属性`by lazy`, 但是他只可以修饰`val`, 会在第一次调用对应属性的时候进行初始化, 默认是线程安全
``` java
val testlazy: String by lazy { "fff"}
```
另外他可以通过传入参数来选择不同的多线程处理
``` java
@kotlin.jvm.JvmVersion
public fun <T> lazy(mode: LazyThreadSafetyMode, initializer: () -> T): Lazy<T> =
        when (mode) {
            /*
            * 使用同步锁确保只有一条线程可以进行实例化
            */
            LazyThreadSafetyMode.SYNCHRONIZED -> SynchronizedLazyImpl(initializer)
            /*
            * 同一时期多个线程可以初始化实例，但是只有最先返回的值会作为延迟初始化的实例，使用 AtomicReferenceFieldUpdater.compareAndSet() 方法实现。
            **/
            LazyThreadSafetyMode.PUBLICATION -> SafePublicationLazyImpl(initializer)
            /**
            * 没有任何线程安全保证与开销
            */
            LazyThreadSafetyMode.NONE -> UnsafeLazyImpl(initializer)
        }
```
## 单例模式的实现
Kotlin提供了`object`来很方便的支持了单例模式的实现
``` java
object Singleton {
    fun test(){
        // ...
    }
}

// kotlin中调用
Singleton.test()
// java中调用
Singleton.INSTANCE.test();
```
我们看下他转为Java代码后是如何实现的.
``` java
public final class Singleton {
   public static final Singleton INSTANCE;

   public final void test() {
   }

   static {
      Singleton var0 = new Singleton();
      INSTANCE = var0;
   }
}
```
很好, 一个典型的饿汉式. 饿汉式的缺点我们简明讲下, 由于是类加载的第一时间就会新建实例, 所以当我们整个工程没有用到的时候, 就会导致内存空间的浪费.另外, 它无法自定义构造函数.
![object](./Kotlin.png)
如果我们不适用`object`呢, 应该如何实现单例模式?

我们来尝试用kotlin写一个`DSL`单例模式, 先看java的实现方法
``` java
public class SingletonDSL {
    private static volatile SingletonDSL instance;

    private SingletonDSL(){

    }

    public static SingletonDSL getInstance() {
        if(null == instance){
            synchronized (SingletonDSL.class){
                if(null == instance){
                    instance = new SingletonDSL();
                }
            }
        }
        return instance;
    }
}
```
ok, 下面是翻译工作
``` java
class SingleDSLKotlin private constructor (){
    companion object {
        @Volatile private var sInstance: SingleDSLKotlin? = null

        fun getInstance() = sInstance ?: synchronized(SingleDSLKotlin::class.java){
            sInstance ?: SingleDSLKotlin().also { sInstance = it }
        }
    }
}
```
根据上面`lazy`的延迟初始化的特性(通过查看源码我们可以发现他内部也是用双重锁机制来实现的), 我们还可以更加的简单实现
``` java
class SingleDSLKotlin private constructor (){
    companion object {
        val INSTANCE by lazy { SingleDSLKotlin() }
    }
}
```
当然我们也可以通过静态内部类来实现单例模式
``` java
class SingletonStaticClass private constructor(){

    fun getInstance() = INSTANCE.sInstance


    companion object INSTANCE{
        private val sInstance = SingletonStaticClass()
    }
}
```
这里关于构造函数的相关基础知识可参见[官网](https://www.kotlincn.net/docs/reference/classes.html)

## 域函数的区别
我们在前面写DSL单例的demo的时候, 用到了一个`also`.我们开发过程中会经常用到这几个作用域函数`run`, `with`, `apply`, `with`, `also`, `let`

要理解源码, 我们首先要搞明白`inline`[内联函数](https://www.kotlincn.net/docs/reference/inline-functions.html)是做什么用的.

> 在kotlin中, 函数也是作为一个对象存储在内存中.当我们调用一个函数的时候, VM首先去找你函数存储的位置, 然后执行函数, 最后再回到你调用函数的地方. 这会分别引入了内存空间的开销和虚拟调用运行的时间开销

而内联函数做的就是在编译期就将函数的`调用`替换成函数的`定义`.

然后我们再回头看这几个函数的作用
### let
我们最开始接触的作用域函数应该就是`let`了, 当我们处理一个可空对象的时候, 要获取它的内部某个属性的时候, 我们一般都是通过使用`?.let{}`来忽略掉空对象逻辑处理情况
``` java
@kotlin.internal.InlineOnly
public inline fun <T, R> T.let(block: (T) -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return block(this)
}
```
可以看到他是将自身T作为参数传入调用函数中, 然后返回最后执行的结果.
``` Java
private fun descriLet(){
        val des = "showValue"
        val letResult = des.let {
            Log.e("lettt", it)
            true
        }
        Log.e("let result", letResult.toString())
    }
```
我们可以通过输出结果里验证
![let](./let.png)
### run
``` Java
/**
 * Calls the specified function [block] and returns its result.
 */
@kotlin.internal.InlineOnly
public inline fun <R> run(block: () -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return block()
}

/**
 * Calls the specified function [block] with `this` value as its receiver and returns its result.
 */
@kotlin.internal.InlineOnly
public inline fun <T, R> T.run(block: T.() -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return block()
}
```
可以看出当我们使用`T.run`的时候, 是作为T.()扩展函数的调用块, 最后返回闭包执行的结果
``` Java
private fun describeRun(){
        val runResult = run{
            true
        }
        Log.e("run result", runResult.toString())

        val runResult1 = "T.run".run {
            Log.e("run", this)
            Log.e("length", length.toString()) // print "length: 5"
            2
        }
        Log.e("run result2", runResult1.toString())  // print "run result2: 2"
    }
```
### also
``` java
/**
 * Calls the specified function [block] with `this` value as its argument and returns `this` value.
 */
@kotlin.internal.InlineOnly
@SinceKotlin("1.1")
public inline fun <T> T.also(block: (T) -> Unit): T {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    block(this)
    return this
}
```
`also`和`let`有点像, 但是他返回的对象与闭包执行结果没有关系, 返回的是调用对象本身
``` Java
private fun describeAlso(){
        val alsoResult = "also result".also {
            it.reversed()
        }
        Log.e("also result", alsoResult) // print "also result: also result"
    }
```
### apply
``` java
/**
 * Calls the specified function [block] with `this` value as its receiver and returns `this` value.
 */
@kotlin.internal.InlineOnly
public inline fun <T> T.apply(block: T.() -> Unit): T {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    block()
    return this
}
```
作为T.()扩展函数调用块执行, 返回被调用对象本身
``` Java
private fun describeApply(){
        val applyResult = "apply".apply {
            reversed()
            length
        }
        Log.e("apply Result", applyResult) // print "apply Result: apply"
    }
```
### with
``` Java
/**
 * Calls the specified function [block] with the given [receiver] as its receiver and returns its result.
 */
@kotlin.internal.InlineOnly
public inline fun <T, R> with(receiver: T, block: T.() -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return receiver.block()
}
```
`with`需要我们传入一个参数`receiver`, 然后作为它的扩展函数执行闭包, 返回执行结果.
``` Java
private fun describeWith(){
        val withResult = with("with"){
            reversed()
        }
        Log.e("with result", withResult) // print "with result: htiw"
    }
```
关于他们在用处上的一些区别, 可以看[这里](https://medium.com/@elye.project/mastering-kotlin-standard-functions-run-with-let-also-and-apply-9cd334b0ef84)
