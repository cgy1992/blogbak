title: Enum的正确使用方式
categories:
- android学习记录
tags:
- android
---
## 前言

看到目前项目里用到蛮多枚举, 才有了这篇小文章分享

## 为什么使用Enum

java中的Enum是包含固定常量集的数据类型.当我们需要预定义一组代表某种数据的值时一般都会使用枚举, 而当要保证类型安全时, 我们经常会使用Enum。

比如, 当我们要保证常量使用正常时, 我们经常使用Enum在编译时校验确保类型安全
<!-- more -->

## 使用Enum的缺点

在[Android开发者官网](https://developer.android.com/topic/performance/memory.html?hl=zh-cn)上, 有这样一段话

> enums often require more than twice as much memory as static constants. You should strictly avoid using enums on Android.

Enum中的每个值都是一个对象,每个声明都将使用一些运行时内存来简单引用该对象,所以Enum相较于static int会`占用更多的内存`.

另外添加单个Enum将`增加最终DEX文件的大小`（是static int的**13**倍）.

## 解决方案

Google提供了[注解库](https://developer.android.com/studio/write/annotations.html#enum-annotations)通过`Typedef`协助我们解决了Enum的问题,它可以确保特定参数, 返回值或字段引用特定的常量集,还可以完成代码以自动提供允许的常量.

我们可以通过使用`@IntDef`和`@StringDef`来帮助我们在编译时检查像Enum这项的变量赋值.

## 使用姿势
1. 首先我们需要依赖注解库
``` groovy
dependencies {compile 'com.android.support:support-annotations:24.2.0'}
```
2. 直接上代码了, 因为还是蛮简单的
``` java
public class Person {
        public static final int MALE = 0;
        public static final int FEMALE = 1;
        private  int sex;
        public String getSexValue(){
            if(MALE == sex){
                return "男";
            }else if(FEMALE == sex){
                return "女";
            }
            return "";
        }
        public void setSex(@sexDef int sex) {
            this.sex = sex;
        }
        //  定义该注解被保留的时间长短
        //  RetentionPolicy.CLASS      注解被保留到class文件, 但jvm加载class文件时候被遗弃, 这是默认生命周期; 用于在编译时进行一些预处理操作, 比如生成一些辅助代码(ButterKnife)
        //  RetentionPolicy.RUNTIME    注解不仅被保存到class文件中, jvm加载class文件之后, 仍然存在;用于在运行时去动态获取注解信息
        //  RetentionPolicy.SOURCE      注解只保留在源文件, 当Java文件编译成class文件的时候, 注解被遗弃; 用于做一些检查性操作
        @Retention(RetentionPolicy.SOURCE)
        //  使用@IntDef定义声明常量作为枚举
        @IntDef({MALE, FEMALE})
        //  使用@interface声明新的枚举注解类型
        public @interface sexDef{}
    }
```
3. 当我们调用setSex设置性别的时候, 如果输入非指定类型, 则编译不会通过

## 总结

与普通static常量相比, Enum的使用至少为整个apk添加至少两倍以上的字节数，并且使用5到10倍的RAM内存。所以建议尽量避免使用Enum, 当需要使用上述特性时,建议以@IntDef 或 @StringDef 替代使用。

更多的可以看看[The price of ENUMs][https://www.youtube.com/watch?v=Hzs6OBcvNQE&feature=youtu.be]视频, 需科学上网
