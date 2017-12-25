title: ObjectBox-Java (android)使用手册
intro: ObjectBox使用手册翻译
categories:  
- android学习记录
tags:
- android
- ObjectBox
---
## 前前言
本篇主要是方便自己记忆所写, 基本是撸完官方文档后的笔记
## 前言
ObjectBox是一款由greenrobot出的基于noSql的ORM数据库, 但又支持表关系的定义以及事务的处理, 另外在性能上有着非常卓越的表现
(关于性能比较, 可以看[这篇](1)),
同时可以接入rxJava的扩展库, 并与google最新出的框架组件(Android Architecture Components)中的LiveData结合使用, 支持Kotlin.
目前版本更新到1.2.1
<!-- more -->
## 依赖
1. 在项目根目录的gradle添加它的依赖仓库地址
``` groovy
buildscript {
    ext.objectboxVersion = '1.2.1'
    repositories {
        jcenter()
        maven { url "http://objectbox.net/beta-repo/" }
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:2.3.3'
        classpath "io.objectbox:objectbox-gradle-plugin:$objectboxVersion"
    }
}
allprojects {
    repositories {
        jcenter()
        maven { url "http://objectbox.net/beta-repo/" }
    }
}
```
2. 在应用项目模块(app module)中添加插件
``` groovy
apply plugin: 'com.android.application'
apply plugin: 'io.objectbox'
```

## 基本使用
1. 准备`ObjectBox`对象单例并实例化, 可以放在application的`onCreate()`中
``` java
// MyObjectBox类文件这时候是引用不到, 它是根据实体类自动生成(build), 用来设置BoxStore对象
boxStore = MyObjectBox.builder().androidContext(applicationContext).build();
```
2. 添加一个对象类, 添加`@Entity`注解, 进行表映射
``` java
@Entity
public class User{
    // 主键, 必须有, 并且必须是long类型
    @Id
    private long id;

    private String userName;
    private int userAge;
    // 必须有
    public User(){}
}
```
P.S 这时候记得build一下, MyObjectBox就自动生成了
3. 这时候我们就可以通过`Box<User>`对象来针对这张表做增删改查工作了
``` java
Box<User> userBox = boxStore.boxFor(User.class).build();
```

## 增删改查
`Box`分别有`put` 添加or修改, `query` 查找, `remove` 移除 等开放API可调用.
在调用`put`时, 当Entity的Id不设置, 则会自动为其赋值, 并新增数据. 当有设置Id, 并在表内有对应Id, 则会被覆盖, 相当于更新对应数据.
另外关于`@Id`,有几点需要注意:
- `0`和`-1(0xFFFFFFFFFFFFFFFF)`不能作为Id的值使用
- `0` 或者`null`(如果类型是Long, 但不建议使用Long, 使用long的速度会更快)会是通知永远新增一笔新数据
- 如果`put`一个id比当前最大id大的对象, ObjectBox可能会抛出异常
- 如果要自己分配id, 可以使用注解`@Id(assignable = true)`

相关的方法, 可以参考[JavaDoc](2)中关于`Box`和`QueryBuilder`类中的方法
## 注解
除了本文其他地方已提到的注解, 补充几个比较大概率会用到的, 其他的建议大家可以看看[JavaDoc][2]中的`io.objectbox.annotation`包:
``` java
@Entity
public class User{
    @Id
    private long id;

    @Index
    private String uid;
    @NameInDb("userName")
    private String name;
    @Transient
    private boolean country;
}
```
- `@Index`: 因为在ObjectBox中主键是必须设置为long类型的id, 当我们业务上需要另外主键时, 可以再标注`@Index`, 在ObjectBox中查询时根据他标注的字段来查询, 会加快查询速度
- `@NameInDb`: 字段在数据库中的命名
- `@Transient`: 忽略字段, 不在表中生成

## 数据迁移
ObjectBox可以实现大部分的数据迁移自动化, 当我们要删除或者新增一个字段的时候, 针对数据库我们是不需要做任何操作的.

但是当我们需要重命名字段名或者表名, 或者需要更改字段类型时, 我们需要使用`@Uid`通知ObjectBox

下面我们会分别举两个例子:
1. 重命名操作, 实体类重命名或者字段重命名操作流程都一样, 区别只在于是在在哪里放`@Uid`, 以实体类重命名为例:
  - 在类名上添加`@Uid`
  ``` java
  @Entity
  @Uid
  public class User{
      @Id
      private long id;

      private String userName;
      private int userAge;

      public User(){}
  }
  ```
  - `rebuild`一下, 在`Gradle Console`中会找到下面类似一段
  ```
  错误: [ObjectBox] UID operations for entity "User2":
   [Rename] apply the current UID using @Uid(6966387148602341622L) - [Change/reset] apply a new UID using @Uid(2383770126231565339L)
1 个错误
  ```
  - copy `[Rename]`的`@Uid`值6966387148602341622L, 并针对实体类进行重命名
  ``` java
  @Entity
  @Uid(6966387148602341622L)
  public class User2{
      @Id
      private long id;

      private String userName;
      private int userAge;

      public User2(){}
  }
  ```
  - 重新编译, 就已经迁移成功, 这时候`@Uid(6966387148602341622L)`这条代码就没有用了, 相关记录会在`objectbox-models/default.json`中体现

2. 变更字段类型, 要注意的是, 会导致原类型字段Column的数据会被清空, 大体流程与重命名流程大致相同, 但是赋值的`@Uid` 需要使用的是`[Change/reset]`的值, 表示是一个`新字段`.

P.S 前文提到了`objectbox-models/default.json`这个JSON文件, 这个文件相当于是我们做Migration时处理的文件记录, 所以是需要加入VCS控制
## 关系
- 以后补充

##  事务
ObjectBox的所有操作都是在事务中运行的, 只是这个对我们来说是透明的, 我们不需要关注, 但是当我们需要进行多个操作的时候, 通过显示事务来控制, 可以大大提高app的效率和一致性.
在`BoxStore`中, 提供了四个方法来执行显示事务:
- `runInReadTx` : 在事务中运行给定的Runnable, 不可并发处理
- `runIxTx` : 只读事务, 可以并发处理
- `runInTxAsync` : 在单独的线程中运行, 事务完成后会回调callback(可能为空)
- `callInTx` : 和`runIxTx`类似, 不过允许返回值并可以抛出一个异常

要注意的是, 事务的提交开销较大, 所以在使用隐式事务时, 譬如大批量调用`put`时, 我们需要统一写到一个事务里去提交
``` java
for(User user: userList){
  user.plusAge();
  box.put(user);
}
```
以上的demo我们应该优化为下面这种:
``` java
for(User user: userList){
 user.plusAge();
}
box.put(userList);
```

## 数据库查看
1. 在项目app gradle文件中, 必须在`'io.objectbox'`插件apply之前依赖一下代码
``` groovy
debugCompile "io.objectbox:objectbox-android-objectbrowser:1.2.1"
releaseCompile "io.objectbox:objectbox-android:1.2.1"
```
2. 清单文件申请权限
``` java
<uses-permission android:name="android.permission.INTERNET"/>
```
3. 然后在BoxStore构建`之后`, 加入以下代码
``` java
if(BuildConfig.DEBUG){
            new AndroidObjectBrowser(boxStore).start(this);
  }
```

运行后, 就可以从设备的通知栏点击进入查看数据库, 也可以通过在cmd中输入`adb forward tcp:8090 tcp:8090`, 打开浏览器, 输入http://localhost:8090/index.html 网址查看

## 后记
关于它和rxJava如何使用, 如何做数据的观测, 以及和LiveData的搭配使用, 鉴于目前篇幅过长, 而且LiveData我目前还没有玩过, 所以暂时不写. 后续可能会新补一篇.

具体可以看[Demo](https://github.com/YuTianTina/DatabaseChoice)

[1]:https://notes.devlabs.bg/realm-objectbox-or-room-which-one-is-for-you-3a552234fd6e

[2]:http://objectbox.io/files/objectbox-java/current/
