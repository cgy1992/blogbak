title: androidAPT的使用
date: 2018-05-17 00:00:00
categories:  
- android学习记录
tags:
- APT
- android
---
## 前言
`APT`的概念大家应该不会陌生, 而且在很多第三方库中都有使用到, 最有名的应该就是`ButterKnife`了. 这里基础概念就略过了, 本篇主要是着重在怎么编写自己的注解处理器, 以及一些踩到的坑.
<!-- more -->
## 开始
一般要实现编译器注解处理生成, 需要新建两个module, 分别存放自定义的`Annotation`和对应`Annotation`的处理器.
### 自定义注解
我们先新建存在自定义注解的module, ***注意, 这里建议新建java-library, 便于本地调试时给存放处理器的module依赖使用***, 对应gradle配置如下
``` groovy
apply plugin: 'java-library'

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
}

sourceCompatibility = "1.8"
targetCompatibility = "1.8"

```
自定义一个新的注解
``` java
@Retention(RetentionPolicy.CLASS)
@Target(ElementType.TYPE)
public @interface TestAnnotation {
    String value();
}
```
1. 这里`Retention`注解表示设置注解保留时机, 需要传递的是[`RetentionPolicy`](https://docs.oracle.com/javase/8/docs/api/java/lang/annotation/RetentionPolicy.html)枚举类型, 值分别有:
  - `SOURCE`: 编译器时就会抛弃注解
  - `CLASS`: 注解保留到编译器, 运行期会去除
  - `RUNTIME`: 注解保留到运行期, 编译器时也会存在
2. `Target`表示注解适用的上下文, 即他的目标修饰类型, 可以传数组,值应该为[ElementType](https://docs.oracle.com/javase/8/docs/api/java/lang/annotation/ElementType.html),枚举各个值的含义可以看官方文档, 我们主要用到比较多的应该是
  - `TYPE`: 类, 接口(包括注解类型)或者枚举的声明
  - `METHOD`: 方法声明
  - `FIELD`: 字段声明, 包括枚举常量
  - `LOCAL_VARIABLE`: 局部变量声明
  - `CONSTRUCTOR`: 构造函数的声明

### 注解处理器
同样需要新建一个java-library, 对应gradle的配置如下
``` groovy
apply plugin: 'java-library'
dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])

    // 协助我们生成类文件
    implementation 'com.squareup:javapoet:1.11.0'
    // 自定义注解的库
    implementation project(':anno')
    // 协助自动注册META-INF
    implementation 'com.google.auto.service:auto-service:1.0-rc4'
}
```
然后开始编写处理器, 关于如何使用[JavaPoet](https://github.com/square/javapoet), 建议看下官方文档, 在这里不再细说.最后通过`Filer`来进行文件的写入.
``` java
// AutoService注解协助自动生成META-INF服务, 提供项目识别自定义的注解处理器
@AutoService(Processor.class)
public class TestProcessor extends AbstractProcessor {
    private Filer mFiler;
    private Messager messager;
    /**
     * init()方法可以初始化拿到一些使用的工具，比如文件相关的辅助类 Filer;元素相关的辅助类Elements;日志相关的辅助类Messager;
     * @param processingEnv
     */
    @Override
    public synchronized void init(ProcessingEnvironment processingEnv) {
        super.init(processingEnv);
        mFiler = processingEnv.getFiler();
        messager = processingEnv.getMessager();
    }

    /**
     * @return 返回Java版本
     * 也可以通过@SupportedSourceVersion来指定, 如果没有设置默认返回的是JDK1.6版本
     */
    @Override
    public SourceVersion getSupportedSourceVersion() {
        return SourceVersion.latest();
    }

    /**
     *
     * @return 支持的注解类型
     * 即是这个处理器是需要注册在哪几个注解上的, 也可以通过@SupportedAnnotationTypes来指定
     */
    @Override
    public Set<String> getSupportedAnnotationTypes() {
        LinkedHashSet<String> types = new LinkedHashSet<>();
        types.add(TestAnnotation.class.getCanonicalName());
        return types;
    }

    /**
     * 一个Processor的main函数
     * @param annotations 请求被处理的注解
     * @param roundEnv 可以用来查询特定注解的被注解元素
     * @return true 被当前处理器处理; false 可能被其他同样声明支持对应注解的处理器用来处理
     */
    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        HashMap<String, String> nameMap = new HashMap<>();
        Set<? extends Element> annotationElements = roundEnv.getElementsAnnotatedWith(TestAnnotation.class);
        for (Element element: annotationElements
             ) {
            TestAnnotation annotation = element.getAnnotation(TestAnnotation.class);
            String name = annotation.value();
            nameMap.put(name, element.getSimpleName().toString());
        }
        generateJavaFile(nameMap);
        return true;
    }

    private void generateJavaFile(Map<String, String> nameMap){
        // 通过javaPoet生成java文件
    }
}
```
## 错误信息
由于注解处理器是JVM在编译期进行运行, 所以普通的Log无法用来提示我们来打印一些日志或者用来提示错误信息.在`Processor`中, 当执行初始化的时候, 会传进来一个`ProcessingEnvironment`参数, 在上方代码注释内我也写了, 他会提供一些我们需要的参数, 比如`Messager`一个用来报告错误, 警报或者其他通知的工具, 它可以用来提醒第三方使用注解的开发者们来处理相关的错误.它有多个重载函数, 用于设置提醒到哪个地步, 具体可以自己尝试下.
``` java
public interface Messager {
    void printMessage(Diagnostic.Kind kind, CharSequence msg);

    void printMessage(Diagnostic.Kind kind, CharSequence msg, Element e);

    void printMessage(Diagnostic.Kind kind, CharSequence msg, Element e, AnnotationMirror a);

    /**
     * @param kind 通知类型
     * @param msg  内容
     * @param e    注解元素
     * @param a    包含注解的值得注解
     * @param v    提示到注解的值使用位置
     */
    void printMessage(Diagnostic.Kind kind,
                      CharSequence msg,
                      Element e,
                      AnnotationMirror a,
                      AnnotationValue v);
}
```
## 使用自定义注解
当我们需要使用的时候, 那么就跟常见的几个第三方库的使用(比如`Dagger`之类)是一样的.
``` groovy
// 依赖管理自定义注解的库
implementation project(':anno')
// apt配置注解处理库
annotationProcessor project(':aptlib')
```
值得注意的是如果你使用的是kotlin开发使用到对应的注解, 那么首先需要依赖kapt插件, 然后以`kapt`替换`annotationProcessor`添加注解处理库, 当项目里有Java文件使用到注解的时候, `kapt`也会兼顾到.
``` groovy
apply plugin: 'kotlin-kapt'
dependencies {
    implementation fileTree(include: ['*.jar'], dir: 'libs')

    implementation project(':anno')
    kapt project(':aptlib')

}
```
## Tips
我们在开始的时候, 谈到注解的声明和处理器需要分别放在不同的module里, 原因是因为, 如果放在一个module里, 那么应用项目在依赖的时候, 就会变成
``` groovy
implementation project(':aptlib')
annotationProcessor project(':aptlib')
```
而不论是我们使用的`AbstractProcessor`还是`JavaPoet`库, 都是依赖于JDK进行编译的, 当应用项目依赖于(`implementation`)这个库的时候, AS就会默认用SDK来进行编译, 导致编译器提示部分类无法加载, 所以我们才需要分成两个module, 保证到进行逻辑处理的处理器可以不会通过`implementation`被依赖进项目中.相关可以看看相关的[issue](https://issuetracker.google.com/issues/37358824)的说明
