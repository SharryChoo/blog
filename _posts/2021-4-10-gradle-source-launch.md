---
title: Gradle 源码分析 - 启动
permalink: gradle-source/launch
key: gradle-source-launch
tags: Gradle
---
# 前言(关于GradleWrapper)

## 一) 定义与意义

### 定义

Gradle Wrapper 是帮助工程实现 Gradle 版本兼容的工具

### 意义

 Gradle 是一个不断发展的工具, 新版本可能会打破向后兼容性, 如果升级了系统的 Gradle 版本, 可能会导致依赖 Gradle 构建的工程无法编译通过

<!--more-->

## 获取 Gradle Wrapper

安装了 Gradle 并添加到环境变量之后, 创建以下的 `build.gradle` 文件

```groovy
task wrapper(type: Wrapper) {
		gradleVersion = '4.10'
}
```

运行 `gradle wrapper —gradle-version 4.10`  来生成 gradle wrapper 文件

## 二) 组成结构

```groovy
myapp/
- gradlew
- gradlew.bat
- gradle/wrapper/
	- gradle-wrapper.jar
	- gradle-wrapper.properties
```

- `gradlew`: `gradle wapper` 在 `unix` 操作系统上使用的 `shell` 脚本
- `gradlew.bat`: `gradle wapper` 在 `Windows` 操作系统上使用的 `batch` 文件
- `gradle-wrapper.jar`: `batch` 文件和 `shell` 脚本所需要使用的 `jar` 文件
- `gradle-wrapper.properties`: 一个配置文件

### gradle-wrapper.properties

```groovy
#Thu Sep 20 22:31:38 CST 2018
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
// 指定 gradle 版本, gradlew 会做好版本管理与分发
distributionUrl=https\://services.gradle.org/distributions/gradle-4.10-all.zip
```

## 三) gradlew.bat

我们平时使用指令一般会 `./gradlew xxx`, 这其实是在执行 shell 脚本, 我们需要关注的代码如下

```bash
// gradlew 的 jar 包
CLASSPATH=$APP_HOME/gradle/wrapper/gradle-wrapper.jar

// 声明 java home.
# Determine the Java command to use to start the JVM.
if [ -n "$JAVA_HOME" ] ; then
    if [ -x "$JAVA_HOME/jre/sh/java" ] ; then
        # IBM's JDK on AIX uses strange locations for the executables
        JAVACMD="$JAVA_HOME/jre/sh/java"
    else
        JAVACMD="$JAVA_HOME/bin/java"
    fi
    if [ ! -x "$JAVACMD" ] ; then
        die "ERROR: JAVA_HOME is set to an invalid directory: $JAVA_HOME

// 执行 shell 脚本
exec "$JAVACMD" "${JVM_OPTS[@]}" -classpath "$CLASSPATH" org.gradle.wrapper.GradleWrapperMain "$@"
```
这里我们从 `GradleWrapperMain` 开始分析一下 Gradle 脚本的启动流程


# Gradle Wrapper 启动

```
public class GradleWrapperMain {
    public static final String GRADLE_USER_HOME_OPTION = "g";
    public static final String GRADLE_USER_HOME_DETAILED_OPTION = "gradle-user-home";
    public static final String GRADLE_QUIET_OPTION = "q";
    public static final String GRADLE_QUIET_DETAILED_OPTION = "quiet";

    public static void main(String[] args) throws Exception {
				// 获取 gradle-wrapper.jar 文件
        File wrapperJar = wrapperJar();
				// 获取 gradle-wrapper.properties 文件
        File propertiesFile = wrapperProperties(wrapperJar);
				// 获取项目的根目录
        File rootDir = rootDir(wrapperJar);

				// 解析命令行参数
				CommandLineParser parser = new CommandLineParser();
        parser.allowUnknownOptions();
        parser.option(GRADLE_USER_HOME_OPTION, GRADLE_USER_HOME_DETAILED_OPTION).hasArgument();
        parser.option(GRADLE_QUIET_OPTION, GRADLE_QUIET_DETAILED_OPTION);

        SystemPropertiesCommandLineConverter converter = new SystemPropertiesCommandLineConverter();
        converter.configure(parser);

        ParsedCommandLine options = parser.parse(args);

        Properties systemProperties = System.getProperties();
        systemProperties.putAll(converter.convert(options, new HashMap<String, String>()));

        File gradleUserHome = gradleUserHome(options);

        addSystemProperties(gradleUserHome, rootDir);

        Logger logger = logger(options);

				// WrapperExecutor 执行后续任务
        WrapperExecutor wrapperExecutor = WrapperExecutor.forWrapperPropertiesFile(propertiesFile);
        wrapperExecutor.execute(
                args,
                new Install(logger, new Download(logger, "gradlew", wrapperVersion()), new PathAssembler(gradleUserHome)),
                new BootstrapMainStarter());
    }
}

```

`GradleWrapperMain` 的 `main` 函数主要做了如下的事情

*   获取了 `gradle-wrapper` 相关的文件
*   解析命令行参数
*   便调用了 `WrapperExecutor.execute` 继续执行后续的任务

```
public class WrapperExecutor {

		public void execute(String[] args, Install install, BootstrapMainStarter bootstrapMainStarter) throws Exception {
			  // 1\. 调用 Install.createDist 方法
				// 解析 gradle-wrapper.properties 中的 distributionUrl, 找寻对应的 gradle 版本是否存在
				// 不存在则执行下载操作
        File gradleHome = install.createDist(config);
				// 2\. 调用 BootstrapMainStarter.start 方法执行后续操作
        bootstrapMainStarter.start(args, gradleHome);
    }
}

```

WrapperExecutor 的 execute 主要负责找寻对应的 gradle 版本, 然后委托给 BootstrapMainStarter 执行后续操作

```
public class BootstrapMainStarter {
    public void start(String[] args, File gradleHome) throws Exception {
				// 从 gradle home 中找寻 gradle 的 jar 包
        File gradleJar = findLauncherJar(gradleHome);
				// 创建一个类加载器
        URLClassLoader contextClassLoader = new URLClassLoader(new URL[]{gradleJar.toURI().toURL()}, ClassLoader.getSystemClassLoader().getParent());
			  // 为当前线程设置一个能找到 gradle 类文件的类加载器
        Thread.currentThread().setContextClassLoader(contextClassLoader);
				// 获取 gradle 的入口函数所在类 GradleMain
        Class<?> mainClass = contextClassLoader.loadClass("org.gradle.launcher.GradleMain");
				// 反射调用入口类中的 GradleMain.main 函数
        Method mainMethod = mainClass.getMethod("main", String[].class);
        mainMethod.invoke(null, new Object[]{args});
        if (contextClassLoader instanceof Closeable) {
            ((Closeable) contextClassLoader).close();
        }
    }

    private File findLauncherJar(File gradleHome) {
        for (File file : new File(gradleHome, "lib").listFiles()) {
            if (file.getName().matches("gradle-launcher-.*\\\\.jar")) {
                return file;
            }
        }
        throw new RuntimeException(String.format("Could not locate the Gradle launcher JAR in Gradle distribution '%s'.", gradleHome));
    }
}

```

这里可以看到 `BootstrapMainStarter` 的主要职责是找寻 `gradle` 脚本的构建入口, 将启动任务委托给 `GradleMain.main` 执行

## 回顾

`gradle wrapper` 的启动流程比较简明, 主要有如下几个步骤

*   解析命令行参数
*   找寻或者下载 [`gradle-wrapper.properties`](http://gradle-wrapper.properties) 文件中定义的指定版本的 `gradle` 的 `jar` 包
*   交付给 `GradleMain.main` 执行后续的启动流程

# Gradle 启动流程

这里选取的是 gradle 4.10 版本, 它的 `GradleMain.main` 入口函数如下

```
public class GradleMain {
    public static void main(String[] args) throws Exception {
				// 委托给 Main 的 main 方法
        new ProcessBootstrap().run("org.gradle.launcher.Main", args);
    }
}

/**
 * The main command-line entry-point for Gradle.
 */
public class Main extends EntryPoint {
    public static void main(String[] args) {
				// 执行父类的 run 方法, 会走到子类的 doAction 中去
        new Main().run(args);
    }

    protected void doAction(String[] args, ExecutionListener listener) {
        UnsupportedJavaRuntimeException.assertUsingVersion("Gradle", JavaVersion.VERSION_1_7);
				// 1\. 执行 CommandLineActionFactory 的 convert, 获取一个 Action<ExecutionListener>
				// 2\. 执行 Action<ExecutionListener> 的 execute
        createActionFactory().convert(Arrays.asList(args)).execute(listener);
    }

    CommandLineActionFactory createActionFactory() {
        return new CommandLineActionFactory();
    }
}

```

可以看到 `GradleMain` 中的 `main` 方法最终会走到 `Main` 的 `doAction` 中, 会执行 `CommandLineActionFactory` 的 `convert`, 获取一个 `Action<ExecutionListener>`, 然后执行 `Action<ExecutionListener>` 的 `execute` 函数

```
public class CommandLineActionFactory {

    public Action<ExecutionListener> convert(List<String> args) {
        ServiceRegistry loggingServices = createLoggingServices();

        LoggingConfiguration loggingConfiguration = new DefaultLoggingConfiguration();
				// 1\. 实例化了 WithLogging 的对象
        return new WithLogging(loggingServices,
            buildLayoutFactory,
            args,
            loggingConfiguration,
						// 注入了一个 action
            new ParseAndBuildAction(loggingServices, args),
            new BuildExceptionReporter(loggingServices.get(StyledTextOutputFactory.class), loggingConfiguration, clientMetaData()));
    }

		private static class WithLogging implements Action<ExecutionListener> {
        private final ServiceRegistry loggingServices;
        private final BuildLayoutFactory buildLayoutFactory;
        private final List<String> args;
        private final LoggingConfiguration loggingConfiguration;
        private final Action<ExecutionListener> action;
        private final Action<Throwable> reporter;

        WithLogging(ServiceRegistry loggingServices, BuildLayoutFactory buildLayoutFactory, List<String> args, LoggingConfiguration loggingConfiguration, Action<ExecutionListener> action, Action<Throwable> reporter) {
            this.loggingServices = loggingServices;
            this.buildLayoutFactory = buildLayoutFactory;
            this.args = args;
            this.loggingConfiguration = loggingConfiguration;
            this.action = action;
            this.reporter = reporter;
        }

				// 2\. 执行 execute
        public void execute(ExecutionListener executionListener) {
            ......
						// 2.1 这里将 action 包装成 ExceptionReportingAction 对象
            Action<ExecutionListener> exceptionReportingAction = new ExceptionReportingAction(action, reporter, loggingManager);
            try {
                ......
								// 2.2 执行 ExceptionReportingAction.execute
                exceptionReportingAction.execute(executionListener);
            } finally {
                loggingManager.stop();
            }
        }
    }
}

```

`Action<ExecutionListener>` 的 `execute` 最终会执行到 `ParseAndBuildAction` 的 `execute` 中

这里我们继续分析

```
public class CommandLineActionFactory {

		private class ParseAndBuildAction implements Action<ExecutionListener> {

        private final ServiceRegistry loggingServices;
        private final List<String> args;

        private ParseAndBuildAction(ServiceRegistry loggingServices, List<String> args) {
            this.loggingServices = loggingServices;
            this.args = args;
        }

        public void execute(ExecutionListener executionListener) {
						// 1\. 创建 action 的 Factory 集合
            List<CommandLineAction> actions = new ArrayList<CommandLineAction>();
            actions.add(new BuiltInActions());
            createActionFactories(loggingServices, actions);

            CommandLineParser parser = new CommandLineParser();
            for (CommandLineAction action : actions) {
                action.configureCommandLineParser(parser);
            }

            Action<? super ExecutionListener> action;
            try {
                ParsedCommandLine commandLine = parser.parse(args);
								// 2\.  通过 Factory 集合, 创建任务委托的 action
                action = createAction(actions, parser, commandLine);
            } catch (CommandLineArgumentException e) {
                action = new CommandLineParseFailureAction(parser, e);
            }
						// 3\. 执行最终的任务
            action.execute(executionListener);
        }

				// 这里是父类的 Action 的函数, 贴到这里便于分析
				protected void createActionFactories(ServiceRegistry loggingServices, Collection<CommandLineAction> actions) {
						// 1.1 这里添加一个 BuildActionsFactory
		        actions.add(new BuildActionsFactory(loggingServices, new ParametersConverter(buildLayoutFactory), new CachingJvmVersionDetector(new DefaultJvmVersionDetector(new DefaultExecActionFactory(new IdentityFileResolver())))));
		    }

        private Action<? super ExecutionListener> createAction(Iterable<CommandLineAction> factories, CommandLineParser parser, ParsedCommandLine commandLine) {
            for (CommandLineAction factory : factories) {
								// 2.1 通过 BuildActionsFactory 创建 runnable
                Runnable action = factory.createAction(parser, commandLine);
								// 2.2 将 runnable 包装成 action.
                if (action != null) {
                    return Actions.toAction(action);
                }
            }
            throw new UnsupportedOperationException("No action factory for specified command-line arguments.");
        }
    }

}

```

这里主要有三个步骤

*   收集可以创建 `action` 的 `Factory`
    *   这里只需要关注 `BuildActionsFactory`
*   通过 `BuildActionsFactory` 创建一个 `action`
*   委托给这个 `action` 执行后续的启动任务

这里我们分析一下 `BuildActionsFactory` 创建 `action` 以及后续执行的流程

## 一) 创建任务委托 action

```
class BuildActionsFactory implements CommandLineAction {

    public Runnable createAction(CommandLineParser parser, ParsedCommandLine commandLine) {
        Parameters parameters = parametersConverter.convert(commandLine, new Parameters());
        parameters.getStartParameter().setInteractive(ConsoleStateUtil.isInteractive());

        parameters.getDaemonParameters().applyDefaultsFor(jvmVersionDetector.getJavaVersion(parameters.getDaemonParameters().getEffectiveJvm()));
				......

				// 这里我们主要关注 runBuildInProcess 这个函数
        if (canUseCurrentProcess(parameters.getDaemonParameters())) {
            return runBuildInProcess(parameters.getStartParameter(), parameters.getDaemonParameters(), loggingServices);
        }

				......
    }

		private Runnable runBuildInProcess(StartParameterInternal startParameter, DaemonParameters daemonParameters, ServiceRegistry loggingServices) {
				// 构建一个 ServiceRegistry
        ServiceRegistry globalServices = ServiceRegistryBuilder.builder()
                .displayName("Global services")
                .parent(loggingServices)
                .parent(NativeServices.getInstance())
								// 注入了 GlobalScopeServices
                .provider(new GlobalScopeServices(startParameter.isContinuous()))
                .build();

        // 将相关参数包装成 runnable
        return runBuildAndCloseServices(
									startParameter,
									daemonParameters, globalServices.get(BuildExecuter.class), globalServices, globalServices.get(GradleUserHomeScopeServiceRegistry.class));
    }

}

```

这里主要有两个步骤, 分别是构建 `ServiceRegistry` 和 创建 `Runnable` 这里我们逐一分析一下

### 1\. ServiceRegistry 的构建

从上面 `ServiceRegistry` 构建的代码中我们可以看到首先是通过 `provider` 方法注入了一个 `GlobalScopeServices` 服务, 然后调用了 `build` 方法

这里我们先分析一个 `build` 方法

```
public class ServiceRegistryBuilder {
    private final List<ServiceRegistry> parents = new ArrayList<ServiceRegistry>();
    private final List<Object> providers = new ArrayList<Object>();
    private String displayName;

		......

    public ServiceRegistryBuilder provider(Object provider) {
        this.providers.add(provider);
        return this;
    }

    public ServiceRegistry build() {
				// 1\. 创建服务注册器 DefaultServiceRegistry
        DefaultServiceRegistry registry = new DefaultServiceRegistry(displayName, parents.toArray(new ServiceRegistry[0]));
				// 2\. 为服务注册器添加 GlobalScopeServices
        for (Object provider : providers) {
            registry.addProvider(provider);
        }
        return registry;
    }

}

```

可以看到这里 [`ServiceRegistryBuilder.build`](http://serviceregistrybuilder.build) 主要处理了两件事情

*   创建服务注册器 `DefaultServiceRegistry`
*   为 `DefaultServiceRegistry` 添加 `GlobalScopeServices`

这里我们逐一分析, 先分析一下 `DefaultServiceRegistry` 的创建流程

```
public class DefaultServiceRegistry implements ServiceRegistry, Closeable {

		public DefaultServiceRegistry(String displayName, ServiceRegistry... parents) {
        this.displayName = displayName;
        this.ownServices = new OwnServices();
        this.thisAsServiceProvider = new ParentServices(this);
        if (parents.length == 0) {
            this.parentServices = null;
            this.allServices = ownServices;
        } else {
            parentServices = setupParentServices(parents);
            allServices = new CompositeServiceProvider(ownServices, parentServices);
        }
				// 收集当前对象中的一些信息
        findProviderMethods(this);
    }

		private void findProviderMethods(Object target) {
        Class<?> type = target.getClass();
				// 1\. 获取当前对象的一些 method
        RelevantMethods methods = RelevantMethods.getMethods(type);
        for (ServiceMethod method : methods.decorators) {
            if (parentServices == null) {
                throw new ServiceLookupException(String.format("Cannot use decorator method %s.%s() when no parent registry is provided.", type.getSimpleName(), method.getName()));
            }
						// 1.2 将 decorate 开头的 method 构建成 FactoryMethodService, 保存到 ownServices 中
            ownServices.add(new FactoryMethodService(this, target, method));
        }
        for (ServiceMethod method : methods.factories) {
						// 1.3 将 create 开头的 method 构建成 FactoryMethodService, 保存到 ownServices 中
            ownServices.add(new FactoryMethodService(this, target, method));
        }
				// 2\. 执行 configure method.
        for (ServiceMethod method : methods.configurers) {
            applyConfigureMethod(method, target);
        }
    }

}

public class RelevantMethods {
    private static final ConcurrentMap<Class<?>, RelevantMethods> METHODS_CACHE = new ConcurrentHashMap<Class<?>, RelevantMethods>();
    private static final ServiceMethodFactory SERVICE_METHOD_FACTORY = new DefaultServiceMethodFactory();

    final List<ServiceMethod> decorators;
    final List<ServiceMethod> factories;
    final List<ServiceMethod> configurers;

		public static RelevantMethods getMethods(Class<?> type) {
				// 1.1 从 class 中找寻指定的 method 暂存起来
        RelevantMethods relevantMethods = METHODS_CACHE.get(type);
        if (relevantMethods == null) {
            relevantMethods = buildRelevantMethods(type);
            METHODS_CACHE.putIfAbsent(type, relevantMethods);
        }

        return relevantMethods;
    }

    private static RelevantMethods buildRelevantMethods(Class<?> type) {
        RelevantMethodsBuilder builder = new RelevantMethodsBuilder(type);
        RelevantMethods relevantMethods;
				// 1.1.1 暂存 decorate 开头的 method.
        addDecoratorMethods(builder);
				// 1.1.2 暂存 create 开头有返回值的非 static 的方法
        addFactoryMethods(builder);
				// 1.1.3 暂存 configure method.
        addConfigureMethods(builder);
        relevantMethods = builder.build();
        return relevantMethods;
    }
}

```

`DefaultServiceRegistry` 的构造函数主要操作如下

*   通过 `findProviderMethods` 找寻如下的 `method`
    *   将 `decorate` 开头的方法, 并构建成 `FactoryMethodService` 保存在 `DefaultServiceRegistry.ownServices` 中
    *   将 `create` 开头有返回值的非 `static` 的方法, 并构建成 `FactoryMethodService` 保存在 `DefaultServiceRegistry.ownServices` 中
*   找寻 `configure` 方法, 直接执行

接下来我们看看 `DefaultServiceRegistry` 注入`GlobalScopeServices` 的流程

```
public class DefaultServiceRegistry implements ServiceRegistry, Closeable {
		/**
     * Adds a service provider bean to this registry. This provider may define factory and decorator methods.
     */
    public DefaultServiceRegistry addProvider(Object provider) {
        assertMutable();
				// 一样是收集 provider 中指定的 method.
        findProviderMethods(provider);
        return this;
    }
}

```

`DefaultServiceRegistry.addProvider` 同样会执行 `findProviderMethods` 收集一些 `method`

`GlobalScopeServices` 中存在 `configure` 方法, 因此被收集之后, 会执行其 `configure` 方法

```
public class GlobalScopeServices extends BasicGlobalScopeServices {

    void configure(ServiceRegistration registration, ClassLoaderRegistry classLoaderRegistry) {
				// 1\. 获取插件服务注册器集合
        final List<PluginServiceRegistry> pluginServiceFactories = new DefaultServiceLocator(
									classLoaderRegistry.getRuntimeClassLoader(),
									classLoaderRegistry.getPluginsClassLoader()
							)
							.getAll(PluginServiceRegistry.class);
        for (PluginServiceRegistry pluginServiceRegistry : pluginServiceFactories) {
						// 2\. 将每一个 PluginServiceRegistry 添加到 DefaultServiceRegistry.ownServices 中
            registration.add(PluginServiceRegistry.class, pluginServiceRegistry);
						// 3\. 调用 DefaultServiceRegistry.findProviderMethods 收集 PluginServiceRegistry 插件中的方法
            pluginServiceRegistry.registerGlobalServices(registration);
        }
    }

}

public class DefaultServiceLocator implements ServiceLocator {
    ......

    @Override
    public <T> List<T> getAll(Class<T> serviceType) throws UnknownServiceException {
				// 委托给 findFactoriesForServiceType 获取 PluginServiceRegistry 类型的服务
        List<ServiceFactory<T>> factories = findFactoriesForServiceType(serviceType);
        ArrayList<T> services = new ArrayList<T>();
        for (ServiceFactory<T> factory : factories) {
            services.add(factory.create());
        }
        return services;
    }

    private <T> List<ServiceFactory<T>> findFactoriesForServiceType(Class<T> serviceType) {
				// 获取服务集合
				// Wrapper 成 Factory
        return factoriesFor(serviceType, implementationsOf(serviceType));
    }

    public <T> List<Class<? extends T>> implementationsOf(Class<T> serviceType) {
        try {
						// 获取服务
            return findServiceImplementations(serviceType);
        } catch (ServiceLookupException e) {
            throw e;
        } catch (Exception e) {
            throw new ServiceLookupException(String.format("Could not determine implementation classes for service '%s'.", serviceType.getName()), e);
        }
    }

    private <T> List<Class<? extends T>> findServiceImplementations(Class<T> serviceType) throws IOException {
				// 收集 META-INF/services/ 中 org.gradle.internal.service.scopes.PluginServiceRegistry 中的服务
        String resourceName = "META-INF/services/" + serviceType.getName();
        Set<String> implementationClassNames = new HashSet<String>();
        List<Class<? extends T>> implementations = new ArrayList<Class<? extends T>>();
        for (ClassLoader classLoader : classLoaders) {
            Enumeration<URL> resources = classLoader.getResources(resourceName);
            while (resources.hasMoreElements()) {
                URL resource = resources.nextElement();
                List<String> implementationClassNamesFromResource;
                try {
                    implementationClassNamesFromResource = extractImplementationClassNames(resource);
                    if (implementationClassNamesFromResource.isEmpty()) {
                        throw new RuntimeException(String.format("No implementation class for service '%s' specified.", serviceType.getName()));
                    }
                } catch (Throwable e) {
                    throw new ServiceLookupException(String.format("Could not determine implementation class for service '%s' specified in resource '%s'.", serviceType.getName(), resource), e);
                }

                for (String implementationClassName : implementationClassNamesFromResource) {
                    if (implementationClassNames.add(implementationClassName)) {
                        try {
                            Class<?> implClass = classLoader.loadClass(implementationClassName);
                            if (!serviceType.isAssignableFrom(implClass)) {
                                throw new RuntimeException(String.format("Implementation class '%s' is not assignable to service class '%s'.", implementationClassName, serviceType.getName()));
                            }
                            implementations.add(implClass.asSubclass(serviceType));
                        } catch (Throwable e) {
                            throw new ServiceLookupException(String.format("Could not load implementation class '%s' for service '%s' specified in resource '%s'.", implementationClassName, serviceType.getName(), resource), e);
                        }
                    }
                }
            }
        }
        return implementations;
    }

}

```

`GlobalScopeServices.configure` 的主要任务如下下

*   收集 `META-INF/services/` 中 `org.gradle.internal.service.scopes.PluginServiceRegistry` 中的服务注册器, 为其创建 `Factory` 并且缓存到 `DefaultServiceRegistry` 中
![image.png](https://upload-images.jianshu.io/upload_images/4147272-78fcaa3e724062cc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
*   将每一个 `PluginServiceRegistry` 添加到 `DefaultServiceRegistry.ownServices` 属性中缓存

*   调用 `DefaultServiceRegistry.findProviderMethods` 收集每一个 `PluginServiceRegistry` 插件中的方法

接下来我们看看 `Runnable` 的创建流程

### 2\. Runnable 的创建

```
class BuildActionsFactory implements CommandLineAction {

		private Runnable runBuildInProcess(StartParameterInternal startParameter, DaemonParameters daemonParameters, ServiceRegistry loggingServices) {

				...... // 构建一个 ServiceRegistry

        // 将相关参数包装成 runnable
        return runBuildAndCloseServices(
									startParameter,
									daemonParameters,
									// 1\. 获取 BuildActionExecuter
									globalServices.get(BuildExecuter.class),
									globalServices,
									globalServices.get(GradleUserHomeScopeServiceRegistry.class)
				);
    }

		private Runnable runBuildAndCloseServices(StartParameterInternal startParameter,
																							DaemonParameters daemonParameters,
																							BuildActionExecuter<BuildActionParameters> executer,
																							ServiceRegistry sharedServices,
																							Object... stopBeforeSharedServices) {
        BuildActionParameters parameters = createBuildActionParameters(startParameter, daemonParameters);
        Stoppable stoppable = new CompositeStoppable().add(stopBeforeSharedServices).add(sharedServices);
				// 2\. 创建 RunBuildAction 对象
        return new RunBuildAction(executer, startParameter, clientMetaData(), getBuildStartTime(), parameters, sharedServices, stoppable);
    }

}

```

`Runnable` 的创建的过程最主要的是获取 `BuildExecuter`, 然后将其构建成 `RunBuildAction` 对象

这个 `BuildExecuter` 对象是`DefaultServiceRegistry` 通过 `LauncherServices.ToolingGlobalScopeServices.createBuildExecuter` 构建得到的

这里比较有意思, 之所以能找到 `BuildExecuter` 的构建方法, 是因为前面将 `LauncherServices` 类中的 `create` 方法收集到了 `DefaultServiceRegistry` 中(构造时的参数也是通过服务发现注入找到, 外界无需关心, 详情见 `DefaultServiceRegistry.FactoryService.bind`), 它的实现如下

```
public class LauncherServices extends AbstractPluginServiceRegistry {

    @Override
    public void registerGlobalServices(ServiceRegistration registration) {
        registration.addProvider(new ToolingGlobalScopeServices());
    }

    @Override
    public void registerGradleUserHomeServices(ServiceRegistration registration) {
        registration.addProvider(new ToolingBuildSessionScopeServices());
    }

    static class ToolingGlobalScopeServices {

        BuildExecuter createBuildExecuter(List<BuildActionRunner> buildActionRunners,
                                          List<SubscribableBuildActionRunnerRegistration> registrations,
                                          BuildOperationListenerManager buildOperationListenerManager,
                                          TaskInputsListener inputsListener,
                                          StyledTextOutputFactory styledTextOutputFactory,
                                          ExecutorFactory executorFactory,
                                          LoggingManagerInternal loggingManager,
                                          GradleUserHomeScopeServiceRegistry userHomeServiceRegistry,
                                          FileSystemChangeWaiterFactory fileSystemChangeWaiterFactory,
                                          ParallelismConfigurationManager parallelismConfigurationManager
        ) {
						// 这里是一层层的 Wrapper, 最终会返回 SetupLoggingActionExecuter 到上层
            return new SetupLoggingActionExecuter(
                new SessionFailureReportingActionExecuter(
                    new StartParamsValidatingActionExecuter(
                        new ParallelismConfigurationBuildActionExecuter(
                            new GradleThreadBuildActionExecuter(
                                new ServicesSetupBuildActionExecuter(
                                    new ContinuousBuildActionExecuter(
                                        new BuildTreeScopeBuildActionExecuter(
                                            new InProcessBuildActionExecuter(
                                                new SubscribableBuildActionRunner(
                                                    new RunAsBuildOperationBuildActionRunner(
                                                        new ValidatingBuildActionRunner(
                                                            new ChainingBuildActionRunner(buildActionRunners))),
                                                    buildOperationListenerManager,
                                                    registrations)
                                            )
                                        ),
                                        fileSystemChangeWaiterFactory,
                                        inputsListener,
                                        styledTextOutputFactory,
                                        executorFactory),
                                    userHomeServiceRegistry)),
                            parallelismConfigurationManager)),
                    styledTextOutputFactory,
                    Time.clock()),
                loggingManager);
        }
}

```

`LauncherServices.ToolingGlobalScopeServices.createBuildExecuter` 最终会得到一个装饰后的 `SetupLoggingActionExecuter` 对象返回给调用方使用

接下来通过 `RunBuildAction` 的执行分析一下启动流程

## 二) 执行任务委托 action

执行委托的 action 即执行 `RunBuildAction.run` 方法, 它的实现如下

```
public class RunBuildAction implements Runnable {
    private final BuildActionExecuter<BuildActionParameters> executer;
    private final StartParameterInternal startParameter;
    private final BuildClientMetaData clientMetaData;
    private final long startTime;
    private final BuildActionParameters buildActionParameters;
    private final ServiceRegistry sharedServices;
    private final Stoppable stoppable;

    public RunBuildAction(BuildActionExecuter<BuildActionParameters> executer, StartParameterInternal startParameter, BuildClientMetaData clientMetaData, long startTime,
                          BuildActionParameters buildActionParameters, ServiceRegistry sharedServices, Stoppable stoppable) {
        this.executer = executer;
        this.startParameter = startParameter;
        this.clientMetaData = clientMetaData;
        this.startTime = startTime;
        this.buildActionParameters = buildActionParameters;
        this.sharedServices = sharedServices;
        this.stoppable = stoppable;
    }

    public void run() {
        try {
				    // 执行 SetupLoggingActionExecuter 的 execute 方法
            executer.execute(
                    new ExecuteBuildAction(startParameter),
                    new DefaultBuildRequestContext(new DefaultBuildRequestMetaData(clientMetaData, startTime), new DefaultBuildCancellationToken(), new NoOpBuildEventConsumer()),
                    buildActionParameters,
                    sharedServices);
        } finally {
            if (stoppable != null) {
                stoppable.stop();
            }
        }
    }
}

```

`RunBuildAction` 中的 `execute` 主要委托给了 `SetupLoggingActionExecuter.execute` 往下分发, 这里我们关注一下如下几个委托调用

*   `InProcessBuildActionExecuter.execute`
*   `ChainingBuildActionRunner.execute`

### `InProcessBuildActionExecuter.execute`

```
public class InProcessBuildActionExecuter implements BuildActionExecuter<BuildActionParameters> {
    private final BuildActionRunner buildActionRunner;

    public InProcessBuildActionExecuter(BuildActionRunner buildActionRunner) {
        this.buildActionRunner = buildActionRunner;
    }

    public Object execute(final BuildAction action, BuildRequestContext buildRequestContext, BuildActionParameters actionParameters, ServiceRegistry contextServices) {
				// 与 LauncherServices.ToolingGlobalScopeServices.createBuildExecuter 类似
				// 1\. 通过 DefaultServiceRegistry 找到 CompositeBuildServices.CompositeBuildTreeScopeServices.createIncludedBuildRegistry 创建 BuildStateRegistry 对象 DefaultIncludedBuildRegistry
        BuildStateRegistry buildRegistry = contextServices.get(BuildStateRegistry.class);
        ......
        try {
						// 2\. 构建一个 RootBuildState 将 action 委托给它的 run 方法往下分发
            RootBuildState rootBuild = buildRegistry.addRootBuild(BuildDefinition.fromStartParameter(action.getStartParameter(), null), buildRequestContext);
            return rootBuild.run(new Transformer<Object, BuildController>() {
                @Override
                public Object transform(BuildController buildController) {
										// 委托给底层的 Runnable 执行
                    buildActionRunner.run(action, buildController);
                    return buildController.getResult();
                }
            });
        } finally {
            buildOperationNotificationValve.stop();
        }
    }
}

```

`InProcessBuildActionExecuter.execute` 主要做了如下的事情

*   通过 `DefaultServiceRegistry` 找到 `CompositeBuildServices.CompositeBuildTreeScopeServices.createIncludedBuildRegistry` 创建 `BuildStateRegistry` 对象 `DefaultIncludedBuildRegistry`
*   通过 `DefaultServiceRegistry.addRootBuild` 构建一个 `RootBuildState`
*   将 `action` 委托给它的 `RootBuildState.run`方法往下分发

这里我们看看 `DefaultServiceRegistry.addRootBuild` 是如何构建 `RootBuildState` 对象的

```
public class DefaultIncludedBuildRegistry implements BuildStateRegistry, Stoppable {

		@Override
    public RootBuildState addRootBuild(BuildDefinition buildDefinition, BuildRequestContext requestContext) {
        if (rootBuild != null) {
            throw new IllegalStateException("Root build already defined.");
        }
				// 1\. 创建 RootBuildState
        rootBuild = new DefaultRootBuildState(buildDefinition, requestContext, gradleLauncherFactory, listenerManager, rootServices);
        addBuild(rootBuild);
        return rootBuild;
    }

}

class DefaultRootBuildState extends AbstractBuildState implements RootBuildState, Stoppable {
    private final ListenerManager listenerManager;
    private GradleLauncher gradleLauncher;

    DefaultRootBuildState(BuildDefinition buildDefinition, BuildRequestContext requestContext, GradleLauncherFactory gradleLauncherFactory, ListenerManager listenerManager, ServiceRegistry parentServices) {
        this.listenerManager = listenerManager;
				// 2\. 通过 DefaultGradleLauncherFactory.newInstance 创建了一个 DefaultGradleLauncher 对象
        gradleLauncher = gradleLauncherFactory.newInstance(buildDefinition, this, requestContext, parentServices);
    }

		@Override
    public <T> T run(Transformer<T, ? super BuildController> buildAction) {
				// 3\. 将 GradleBuildController
        final GradleBuildController buildController = new GradleBuildController(gradleLauncher);
        RootBuildLifecycleListener buildLifecycleListener = listenerManager.getBroadcaster(RootBuildLifecycleListener.class);
        buildLifecycleListener.afterStart();
        try {
            return buildAction.transform(buildController);
        } finally {
            buildLifecycleListener.beforeComplete();
        }
    }

}

```

`RootBuildState` 的实例为 `DefaultRootBuildState` 中通过 `DefaultGradleLauncherFactory.newInstance` 创建了一个 `DefaultGradleLauncher` 对象

它的 `run` 方法最终会以 `GradleBuildController` 为入参, 分发给 `buildAction` 执行

### `ChainingBuildActionRunner.execute`

```
public class ChainingBuildActionRunner implements BuildActionRunner {
    private List<? extends BuildActionRunner> runners;

    public ChainingBuildActionRunner(List<? extends BuildActionRunner> runners) {
        this.runners = runners;
    }

    @Override
    public void run(BuildAction action, BuildController buildController) {
				//
        for (BuildActionRunner runner : runners) {
            runner.run(action, buildController);
            if (buildController.hasResult()) {
                return;
            }
        }
    }
}

```

`ChainingBuildActionRunner.execute` 主要是遍历 `BuildActionRunner` 集合并执行

这里的 `BuildActionRunner` 其中有一个是 `DefaultRootBuildState.run`, 因此最终会走到 [`GradleBuildController.run`](http://gradlebuildcontroller.run) 中

```
public class GradleBuildController implements BuildController {
    ......

    public GradleLauncher getLauncher() {
        if (state == State.Completed) {
            throw new IllegalStateException("Cannot use launcher after build has completed.");
        }
        return gradleLauncher;
    }

    public GradleInternal run() {
        return doBuild(new Callable<GradleInternal>() {
            @Override
            public GradleInternal call() {
								// 执行 GradleBuildController 中的 run 方法
                return getLauncher().executeTasks();
            }
        });
    }

    private GradleInternal doBuild(final Callable<GradleInternal> build) {
        try {
            // TODO:pm Move this to RunAsBuildOperationBuildActionRunner when BuildOperationWorkerRegistry scope is changed
            return workerLeaseService.withLocks(Collections.singleton(workerLeaseService.getWorkerLease()), build);
        } finally {
            state = State.Completed;
        }
    }

    @Override
    public void stop() {
        gradleLauncher.stop();
    }
}

```

最终会走到 `DefaultGradleLauncher.executeTasks` 中进入到构建阶段

# 总结

## Gradle Wrapper 启动

主要有三个步骤

*   收集可以创建 `action` 的 `Factory`
    *   这里只需要关注 `BuildActionsFactory`
*   通过 `BuildActionsFactory` 创建一个 `action`
*   委托给这个 `action` 执行后续的启动任务

## Gradle 脚本启动

*   `RunBuildAction` 的创建
    *   创建服务注册器 `DefaultServiceRegistry,` 其中收集了 `DefaultServiceRegistry`, `GlobalScopeServices` 和 `org.gradle.internal.service.scopes.PluginServiceRegistry` 文件下所有服务注册器的方法信息
        *   通过 `findProviderMethods` 找寻如下的 `method`
            *   将 `decorate` 开头的方法, 并构建成 `FactoryMethodService` 保存在 `DefaultServiceRegistry.ownServices` 中
            *   将 `create` 开头有返回值的非 `static` 的方法, 并构建成 `FactoryMethodService` 保存在 `DefaultServiceRegistry.ownServices` 中
        *   找寻 `configure` 方法, 直接执行
    *   `DefaultServiceRegistry` 通过暂存的 `create` 方法对应的 `FactoryMethodService`, 进而找到 `LauncherServices.ToolingGlobalScopeServices.createBuildExecuter` 创建构建执行器 `SetupLoggingActionExecuter`
    *   将 `SetupLoggingActionExecuter` 包装成 `RunBuildAction` 并执行
*   `RunBuildAction` 的主要执行流程如下
    *   `InProcessBuildActionExecuter.execute`
        *   通过 `DefaultServiceRegistry` 找到 `CompositeBuildServices.CompositeBuildTreeScopeServices.createIncludedBuildRegistry` 创建 `BuildStateRegistry` 对象 `DefaultIncludedBuildRegistry`
        *   通过 `DefaultServiceRegistry.addRootBuild` 构建一个 `DefaultRootBuildState`
            *   通过 `DefaultGradleLauncherFactory.newInstance` 创建了一个 `DefaultGradleLauncher` 对象
            *   `RootBuildState.run` 方法最终会以 `GradleBuildController` 为入参, 分发给 `buildAction` 执行 `transform`
        *   将 `action` 委托给它的 `DefaultRootBuildState.run`方法往下分发
    *   `ChainingBuildActionRunner.execute`
        *   其中有一个 `BuildActionRunner` 是 `RootBuildState.run`, 最终会走到 `DefaultGradleLauncher.executeTasks` 中进入到构建阶段

## 思考

在阅读 `Gradle` 启动源码中, 令人印象最深刻的是通过 `DefaultServiceRegistry` 解析 `create/decorate` 方法并通过 `get` 直接获取指定对象的过程, 这里有几个问题需要反思一下

*   为什么要采用这样的设计, 有什么好处? 有什么弊端?
    *   好处
        *   **让使用方需要了解的信息最小化**: `get` 方法让 `DefaultServiceRegistry` 承担了工厂的职责, 使用方不需要知道谁能够生产这个对象, 而且不用构造函数需要哪些参数, `FactoryMethodService` 可以通过服务发现的方式去找到每个参数的对象
        *   **解耦**: 每个服务只需要暴露需要提供的方法, 方法会通过反射去查找和注册, 没有其他类会产生直接的依赖, 代码迁移以及隔离的成本极低
    *   劣势
        *   **拓展成本高:** 每新增一个 `Service` 的 `create` 方法时, 要为其提供对应参数的服务发现, 后续其他同学接手维护时需要了解这一套设计规则
        *   **代码可读性差:** 因为解耦彻底, 没有直接依赖, IDE 上无法直接找到调用方, 出现问题时有一定的排查成本
        *   **性能损耗:** 每次新增一个 `ServiceRegistry` 都会通过反射去搜集方法, 可能会带来一些性能损耗
*   在以后架构设计的过程中, 是否可以借鉴这种方案?
    *   在高速迭代的业务层代码中谨慎使用, 降低业务逻辑的排查成本
    *   在提供基础服务的架构侧可以尝试, 由稳定的提供方去维护

# 参考文献
*   [](https://www.jianshu.com/p/625bc82003d7)[https://www.jianshu.com/p/625bc82003d7](https://www.jianshu.com/p/625bc82003d7)
*   [](https://www.jianshu.com/p/d1099b77a753)[https://www.jianshu.com/p/d1099b77a753](https://www.jianshu.com/p/d1099b77a753)
