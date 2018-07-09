title: AppPlugin源码解析
date: 2018-07-06 00:00:00
categories:  
- 源码解析
tags:
- Android
- 源码解析
---
之前为了优化内部的Route, 去看了下`TransformAPI`, 为了了解`TransformAPI`的执行顺序, 就看了下`AppPlugin`的源码.
本篇源码基于android gradle 3.0.1的版本.
<!--more-->
### 总入口
我们直接从入口`apply`开始看, 他调到了父类`BasePlugin`的apply方法.
``` java
protected void apply(@NonNull Project project) {
        // 一些基础path和插件版本的校验
        // 以及一些流程记录的初始化
        ...

        // 真正执行的是configureProject方法
        // 项目配置
        threadRecorder.record(
                ExecutionType.BASE_PLUGIN_PROJECT_CONFIGURE,
                project.getPath(),
                null,
                this::configureProject);

        threadRecorder.record(
                ExecutionType.BASE_PLUGIN_PROJECT_BASE_EXTENSION_CREATION,
                project.getPath(),
                null,
                this::configureExtension);

        threadRecorder.record(
                ExecutionType.BASE_PLUGIN_PROJECT_TASKS_CREATION,
                project.getPath(),
                null,
                this::createTasks);
    }

public interface Recorder {
  ...
  // record方法用来执行block, 然后将执行结果记录
    void record(
            @NonNull ExecutionType executionType,
            @NonNull String projectPath,
            @Nullable String variant,
            @NonNull VoidBlock block);
          }
```
`apply`入口的代码非常简洁, 主要执行就三个方法, `configureProject`, `configureExtension`和`createTasks`, 看方法名可以猜到, 他们的作用应该是配置Project, 配置插件的Extensions, 并且创建任务(task).
### configureProject
``` java
private void configureProject() {
        extraModelInfo = new ExtraModelInfo(projectOptions, project.getLogger());
        checkGradleVersion();

        sdkHandler = new SdkHandler(project, getLogger());

        // sdk, dependence的下载
        if (!project.getGradle().getStartParameter().isOffline()
                && projectOptions.get(BooleanOption.ENABLE_SDK_DOWNLOAD)
                && !projectOptions.get(BooleanOption.IDE_INVOKED_FROM_IDE)) {
            SdkLibData sdkLibData = SdkLibData.download(getDownloader(), getSettingsController());
            sdkHandler.setSdkLibData(sdkLibData);
        }

        // 构建androidBuilder
        androidBuilder = new AndroidBuilder(
                project == project.getRootProject() ? project.getName() : project.getPath(),
                creator,
                new GradleProcessExecutor(project),
                new GradleJavaProcessExecutor(project),
                extraModelInfo,
                getLogger(),
                isVerbose());
        dataBindingBuilder = new DataBindingBuilder();
        dataBindingBuilder.setPrintMachineReadableOutput(
                extraModelInfo.getErrorFormatMode() ==
                        ExtraModelInfo.ErrorFormatMode.MACHINE_PARSABLE);

        // Apply the Java and Jacoco plugins.
        // JavaBasePlugin, 用于编译java代码成class, 并组装成一个jar文件
        project.getPlugins().apply(JavaBasePlugin.class);
        project.getPlugins().apply(JacocoPlugin.class);

        // 设置assmble task的描述
        project.getTasks()
                .getByName("assemble")
                .setDescription(
                        "Assembles all variants of all applications and secondary packages.");

        // 注册编译监听
        // 会在每个project build结束后调用到, 而不单是当前的project
        project.getGradle()
                .addBuildListener(
                        new BuildListener() {
                            @Override
                            public void buildStarted(Gradle gradle) {
                                TaskInputHelper.enableBypass();
                            }

                            @Override
                            public void settingsEvaluated(Settings settings) {}

                            @Override
                            public void projectsLoaded(Gradle gradle) {}

                            @Override
                            public void projectsEvaluated(Gradle gradle) {}

                            @Override
                            public void buildFinished(BuildResult buildResult) {
                                // 复合构建时不会多次重复清除
                                if (buildResult.getGradle().getParent() != null) {
                                    return;
                                }
                                // 清除dex缓存
                                ExecutorSingleton.shutdown();
                                sdkHandler.unload();
                                threadRecorder.record(
                                        ExecutionType.BASE_PLUGIN_BUILD_FINISHED,
                                        project.getPath(),
                                        null,
                                        () -> {
                                            PreDexCache.getCache()
                                                    .clear(
                                                            FileUtils.join(
                                                                    project.getRootProject()
                                                                            .getBuildDir(),
                                                                    FD_INTERMEDIATES,
                                                                    "dex-cache",
                                                                    "cache.xml"),
                                                            getLogger());
                                            Main.clearInternTables();
                                        });
                            }
                        });

        // 注册taskGraph构建完成时的回调
        project.getGradle()
                .getTaskGraph()
                .addTaskExecutionGraphListener(
                        taskGraph -> {
                            TaskInputHelper.disableBypass();
                            // 遍历所有task, 如果是dexTransform, 则读取对应的dexTransform的缓存
                            for (Task task : taskGraph.getAllTasks()) {
                                if (task instanceof TransformTask) {
                                    Transform transform = ((TransformTask) task).getTransform();
                                    if (transform instanceof DexTransform) {
                                        PreDexCache.getCache()
                                                .load(
                                                        FileUtils.join(
                                                                project.getRootProject()
                                                                        .getBuildDir(),
                                                                FD_INTERMEDIATES,
                                                                "dex-cache",
                                                                "cache.xml"));
                                        break;
                                    }
                                }
                            }
                        });
    }
```
看代码我们可以梳理`configureProject`流程如下
1. 校验gradle版本
2. 下载dependence
3. 构建`androidBuilder`
4. 如果有设置`dataBinding`, 则会构建`dataBindingBuilder`(这个不在本篇重点)
5. 分别引用`JavaBasePlugin`和`JacocoPlugin`插件, 分别用于编译java代码成class和用于检查测试用例的覆盖率
6. 注册taskGraph构建准备时的回调, 在准备完成时读取DexTransform的dex-cache缓存
7. 注册编译完成时的回调, 进行dex缓存的清除工作
### configureExtension

``` java
private void configureExtension() {
        // 分别创建buildType, ProductFlavor, SigningConfig, BaseVariantOutput对应存储的集合对象, 设置默认的Extensions
        final NamedDomainObjectContainer<BuildType> buildTypeContainer =
                project.container(
                        BuildType.class,
                        new BuildTypeFactory(instantiator, project, extraModelInfo));
        final NamedDomainObjectContainer<ProductFlavor> productFlavorContainer =
                project.container(
                        ProductFlavor.class,
                        new ProductFlavorFactory(
                                instantiator, project, project.getLogger(), extraModelInfo));
        final NamedDomainObjectContainer<SigningConfig> signingConfigContainer =
                project.container(SigningConfig.class, new SigningConfigFactory(instantiator));

        final NamedDomainObjectContainer<BaseVariantOutput> buildOutputs =
                project.container(BaseVariantOutput.class);
        // 设置project的buildOutputs属性的扩展为buildOutputs
        project.getExtensions().add("buildOutputs", buildOutputs);
        // 在appPlugin中实现, 创建android扩展属性
        extension =
                createExtension(
                        project,
                        projectOptions,
                        instantiator,
                        androidBuilder,
                        sdkHandler,
                        buildTypeContainer,
                        productFlavorContainer,
                        signingConfigContainer,
                        buildOutputs,
                        extraModelInfo);

        // ndk处理器实例化
        ndkHandler =
                new NdkHandler(
                        project.getRootDir(),
                        null, /* compileSkdVersion, this will be set in afterEvaluate */
                        "gcc",
                        "" /*toolchainVersion*/,
                        false /* useUnifiedHeaders */);


        @Nullable
        FileCache buildCache = BuildCacheUtils.createBuildCacheIfEnabled(project, projectOptions);

        GlobalScope globalScope =
                new GlobalScope(
                        project,
                        projectOptions,
                        androidBuilder,
                        extension,
                        sdkHandler,
                        ndkHandler,
                        registry,
                        buildCache);
        // ApplicationVariantFactory
        variantFactory = createVariantFactory(globalScope, instantiator, androidBuilder, extension);

        // ApplicationTaskManager, 管理application的任务创建
        taskManager =
                createTaskManager(
                        globalScope,
                        project,
                        projectOptions,
                        androidBuilder,
                        dataBindingBuilder,
                        extension,
                        sdkHandler,
                        ndkHandler,
                        registry,
                        threadRecorder);

        // 变体的创建和管理的管理器
        variantManager =
                new VariantManager(
                        globalScope,
                        project,
                        projectOptions,
                        androidBuilder,
                        extension,
                        variantFactory,
                        taskManager,
                        threadRecorder);
        // 注册自定义工具模型, 和native工具模型
        registerModels(registry, globalScope, variantManager, extension, extraModelInfo);

        // map the whenObjectAdded callbacks on the containers.
        signingConfigContainer.whenObjectAdded(variantManager::addSigningConfig);

        buildTypeContainer.whenObjectAdded(
                buildType -> {
                    SigningConfig signingConfig =
                            signingConfigContainer.findByName(BuilderConstants.DEBUG);
                    buildType.init(signingConfig);
                    variantManager.addBuildType(buildType);
                });

        productFlavorContainer.whenObjectAdded(variantManager::addProductFlavor);

        // map whenObjectRemoved on the containers to throw an exception.
        signingConfigContainer.whenObjectRemoved(
                new UnsupportedAction("Removing signingConfigs is not supported."));
        buildTypeContainer.whenObjectRemoved(
                new UnsupportedAction("Removing build types is not supported."));
        productFlavorContainer.whenObjectRemoved(
                new UnsupportedAction("Removing product flavors is not supported."));

        // create default Objects, signingConfig first as its used by the BuildTypes.
        variantFactory.createDefaultComponents(
                buildTypeContainer, productFlavorContainer, signingConfigContainer);
    }
```
`configureExtension`主要处理了`extension`
1. 分别创建buildType, ProductFlavor, SigningConfig, BaseVariantOutput对应存储的集合对象
2. 创建android Extension, `createExtension`是个抽象方法, 具体实现代码可以看`AppPlugin`里, 在我们的工程配置gradle文件里, 就是我们熟悉的android闭包里的配置
``` java
protected BaseExtension createExtension(
            @NonNull Project project,
            @NonNull ProjectOptions projectOptions,
            @NonNull Instantiator instantiator,
            @NonNull AndroidBuilder androidBuilder,
            @NonNull SdkHandler sdkHandler,
            @NonNull NamedDomainObjectContainer<BuildType> buildTypeContainer,
            @NonNull NamedDomainObjectContainer<ProductFlavor> productFlavorContainer,
            @NonNull NamedDomainObjectContainer<SigningConfig> signingConfigContainer,
            @NonNull NamedDomainObjectContainer<BaseVariantOutput> buildOutputs,
            @NonNull ExtraModelInfo extraModelInfo) {
        return project.getExtensions()
                .create(
                        "android",
                        AppExtension.class,
                        project,
                        projectOptions,
                        instantiator,
                        androidBuilder,
                        sdkHandler,
                        buildTypeContainer,
                        productFlavorContainer,
                        signingConfigContainer,
                        buildOutputs,
                        extraModelInfo);
    }
```
3. 分别创建`taskManager`和`variantManager`, 看名字就可以知道他负责的分别是task和variant
4. 针对签名配置signingConfig, buildType, productFlavor 注册新增时候的监听, 对应添加到variantManager中做管理. 同时, 他们是不支持删除的.
5. 创建默认的variant

### createTasks

``` java
private void createTasks() {
        threadRecorder.record(
                ExecutionType.TASK_MANAGER_CREATE_TASKS,
                project.getPath(),
                null,
                // 在before evaluate新增部分tasks
                () ->
                        taskManager.createTasksBeforeEvaluate(
                                new TaskContainerAdaptor(project.getTasks())));

        project.afterEvaluate(
                project ->
                        threadRecorder.record(
                                ExecutionType.BASE_PLUGIN_CREATE_ANDROID_TASKS,
                                project.getPath(),
                                null,
                                () -> createAndroidTasks(false)));
    }
```
在`createTask`阶段, 首先会在配置收集之前(即在apply plugin)阶段会创建一些task, 这里`TaskContainerAdaptor`可以理解为对task的又一层管理封装.

在配置收集获取所有task之后, 我们会调用到`createAndroidTasks`
在执行真正的创建tasks之前, 主要是对extension内一些配置进行校验, 譬如针对buildToolsVersion, compileSdkVersion是否指定, 是否另外引用了JavaPlugin等等, 这里的代码可以自行看源码了解
``` java
final void createAndroidTasks(boolean force) {
        ...
        threadRecorder.record(
                ExecutionType.VARIANT_MANAGER_CREATE_ANDROID_TASKS,
                project.getPath(),
                null,
                // 根据variant创建tasks
                () -> {
                    variantManager.createAndroidTasks();
                    ApiObjectFactory apiObjectFactory =
                            new ApiObjectFactory(
                                    androidBuilder,
                                    extension,
                                    variantFactory,
                                    instantiator,
                                    project.getObjects());
                    for (VariantScope variantScope : variantManager.getVariantScopes()) {
                        BaseVariantData variantData = variantScope.getVariantData();
                        apiObjectFactory.create(variantData);
                    }
                });
        ...
    }
```
这里我们可以主要看下`variantManager.createAndroidTasks()`方法内容, 这里根据variant创建对应的tasks
可以看到具体流程如下:
1. 校验是否引用apt插件(这在3.0里使用annotationProcessor来声明注解处理器)
2. 填充收集所以variant
3. 根据variant创建对应的tasks

``` java
public void createAndroidTasks() {
        variantFactory.validateModel(this);
        // 校验是否引用apt插件, 如果有, 抛出异常
        variantFactory.preVariantWork(project);

        final TaskFactory tasks = new TaskContainerAdaptor(project.getTasks());
        if (variantScopes.isEmpty()) {
            recorder.record(
                    ExecutionType.VARIANT_MANAGER_CREATE_VARIANTS,
                    project.getPath(),
                    null /*variantName*/,
                    // 填充所有variant
                    this::populateVariantDataList);
        }

        // Create top level test tasks.
        recorder.record(
                ExecutionType.VARIANT_MANAGER_CREATE_TESTS_TASKS,
                project.getPath(),
                null /*variantName*/,
                // 创建测试tasks
                () -> taskManager.createTopLevelTestTasks(tasks, !productFlavors.isEmpty()));

        for (final VariantScope variantScope : variantScopes) {
            recorder.record(
                    ExecutionType.VARIANT_MANAGER_CREATE_TASKS_FOR_VARIANT,
                    project.getPath(),
                    variantScope.getFullVariantName(),
                    // 根据variant创建不同的tasks, 创建assemble任务也在这里
                    () -> createTasksForVariantData(tasks, variantScope));
        }

        taskManager.createReportTasks(tasks, variantScopes);
    }
```
### 总结
最后我们总结下, `appPlugin` `apply`的整体流程(当然, 因为他执行的是父类的`apply`, 所以流程适用于`libraryPlugin`)
1. project的配置
      1. 检验gradle版本
      2. 下载dependence
      3. 构建`androidBuilder`
      4. 引用`JavaBasePlugin`和`JacocoPlugin`
      5. 定义编译过程的回调, 负责处理dex-cache的加载和清除工作
2. extension的配置
      1. 创建和配置extension
      2. 创建taskManager和variantManager
      3. 配置签名设置, buildType, productFlavor, 供后续task以及编译时使用
3. 创建task(任务)
      1. 一些extension设置的检验
      2. 编译task的创建
