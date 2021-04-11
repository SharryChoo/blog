---
title: Gradle 源码分析 - 构建
permalink: gradle-source/build
key: gradle-source-build
tags: Gradle
---
# 前言

前面分析了 Gradle 脚本的启动分析, 这里我们看看启动后, Gradle 脚本的执行流程

<!--more-->

```
public class DefaultGradleLauncher implements GradleLauncher {

    private enum Stage {
        Load, LoadBuild, Configure, TaskGraph, Build, Finished
    }

    public GradleInternal executeTasks() {
        doBuildStages(Stage.Build);
        return gradle;
    }

    private void doBuildStages(Stage upTo) {
        try {
            loadSettings();
            if (upTo == Stage.Load) {
                return;
            }
            configureBuild();
            if (upTo == Stage.Configure) {
                return;
            }
            constructTaskGraph();
            if (upTo == Stage.TaskGraph) {
                return;
            }
            runTasks();
            finishBuild();
        } catch (Throwable t) {
            Throwable failure = exceptionAnalyser.transform(t);
            finishBuild(new BuildResult(upTo.name(), gradle, failure));
            throw new ReportedException(failure);
        }
    }

}

```

`DefaultGradleLauncher.executeTasks` 会委托给 `doBuildStages` 执行, 从其内部实现就是 `Gradle` 脚本的构建流程, 主要有如下几个阶段

*   `loadSettings`: 读取配置文件
*   `configureBuild`: 配置工程
*   `constructTaskGraph`: 构建任务有向无环图
*   `runTasks`: 执行任务
*   `finishBuild`: 构建结束

接下来我们对每个阶段逐一分析

# loadSettings

```
public class DefaultGradleLauncher implements GradleLauncher {

		private void loadSettings() {
        if (stage == null) {
						// 1\. 回调 buildStarted
            buildListener.buildStarted(gradle);
						// 2\. 执行 LoadBuild 中的 run 方法
            buildOperationExecutor.run(new LoadBuild());
						// 3\. 更新当前阶段为 Load
            stage = Stage.Load;
        }
    }

		private final InitScriptHandler initScriptHandler;
		private final SettingsLoader settingsLoader;

		private class LoadBuild implements RunnableBuildOperation {
        @Override
        public void run(BuildOperationContext context) {
            // 2.1 加载 init 文件夹下的脚本
            initScriptHandler.executeScripts(gradle);
            // 2.2 编译 buildSrc 目录, 加载 settings.gradle 文件
            settings = settingsLoader.findAndLoadSettings(gradle);
            context.setResult(RESULT);
        }
    }
}

```

`loadSettings` 中主要做了如下几件事情

*   回调 `buildStarted`
*   执行 `LoadBuild` 中的 `run` 方法
    *   加载 `init` 文件夹下的脚本
    *   编译 `buildSrc` 目录, 加载 `settings.gradle` 文件

## 加载 `init` 文件夹下的 `.gradle` 脚本

```
public class InitScriptHandler {
    private final InitScriptProcessor processor;
    private final BuildOperationExecutor buildOperationExecutor;

    public InitScriptHandler(InitScriptProcessor processor, BuildOperationExecutor buildOperationExecutor) {
        this.processor = processor;
        this.buildOperationExecutor = buildOperationExecutor;
    }

    public void executeScripts(final GradleInternal gradle) {
				// 1\. 获取所有的 init 脚本
        final List<File> initScripts = gradle.getStartParameter().getAllInitScripts();
        if (initScripts.isEmpty()) {
            return;
        }

        buildOperationExecutor.run(new RunnableBuildOperation() {
            @Override
            public void run(BuildOperationContext context) {
								// 2\. 执行所有的 init 脚本
                BasicTextResourceLoader resourceLoader = new BasicTextResourceLoader();
                for (File script : initScripts) {
                    TextResource resource = resourceLoader.loadFile("initialization script", script);
                    processor.process(new TextResourceScriptSource(resource), gradle);
                }
            }

            @Override
            public BuildOperationDescriptor.Builder description() {
                return BuildOperationDescriptor.displayName("Run init scripts").progressDisplayName("Running init scripts");
            }
        });
    }
}

```

`init` 脚本执行的流程比较简单, 通过 `gradle` 对象获取到所有的 `init` 脚本文件, 然后依次执行

### init 脚本查找

```
public class StartParameter implements LoggingConfiguration, ParallelismConfiguration, Serializable {

    @Incubating
    public List<File> getAllInitScripts() {
				// 这里聚合了两个 Init 脚本的 Finder
        CompositeInitScriptFinder initScriptFinder = new CompositeInitScriptFinder(
            new UserHomeInitScriptFinder(getGradleUserHomeDir()),
						new DistributionInitScriptFinder(gradleHomeDir)
        );
				// 通过聚合的 finder 获取 init 脚本
        List<File> scripts = new ArrayList<File>(getInitScripts());
        initScriptFinder.findScripts(scripts);
        return Collections.unmodifiableList(scripts);
    }

}

public class UserHomeInitScriptFinder extends DirectoryInitScriptFinder implements InitScriptFinder {

    private final File userHomeDir;

    public UserHomeInitScriptFinder(File userHomeDir) {
        this.userHomeDir = userHomeDir;
    }

    public void findScripts(Collection<File> scripts) {
				// 获取 init 文件夹
        File userInitScript = resolveScriptFile(userHomeDir, "init");
        if (userInitScript != null) {
            scripts.add(userInitScript);
        }
        findScriptsInDir(new File(userHomeDir, "init.d"), scripts);
    }
}

public class DistributionInitScriptFinder extends DirectoryInitScriptFinder {

    final File gradleHome;

    public DistributionInitScriptFinder(File gradleHome) {
        this.gradleHome = gradleHome;
    }

    public void findScripts(Collection<File> scripts) {
        if (gradleHome == null) {
            return;
        }
				// 获取 init.d 脚本文件
        findScriptsInDir(new File(gradleHome, "init.d"), scripts);
    }

}

```

这里主要是从两个方面查找

*   从 gradleHome 目录中查找 `init.d` 脚本文件
*   从用户自定义的 init 文件夹下找 `init.d` 文件

如这个命令`gradle --init-script init.gradle clean`. 通过`--init-script` 参数设置全局的查找`init`文件夹下`.gradle`文件。

## 编译 buildSrc 目录, 加载 settings.gradle 文件

```
public class DefaultSettingsLoader implements SettingsLoader {

		......

    @Override
    public SettingsInternal findAndLoadSettings(GradleInternal gradle) {
        StartParameter startParameter = gradle.getStartParameter();
				// 查找 settings 文件并加载
        SettingsInternal settings = findSettingsAndLoadIfAppropriate(gradle, startParameter);
				......
        return settings;
    }

		private SettingsInternal findSettingsAndLoadIfAppropriate(GradleInternal gradle,
                                                              StartParameter startParameter) {
				// 1\. 查找 Settings 文件
        SettingsLocation settingsLocation = findSettings(startParameter);
				// 2\. 找寻 buildSrc, 构建 ClassLoader
        ClassLoaderScope buildSourceClassLoaderScope = buildSourceBuilder.buildAndCreateClassLoader(settingsLocation.getSettingsDir(), startParameter);
				// 3\. 解析 Settings 文件
        return settingsProcessor.process(gradle, settingsLocation, buildSourceClassLoaderScope, startParameter);
    }

}

```

主要有三个步骤, 这里逐一分析

### 查找 Settings 文件

```
public class DefaultSettingsLoader implements SettingsLoader {

		private ISettingsFinder settingsFinder;

		private SettingsLocation findSettings(StartParameter startParameter) {
				// 转发给了 settingsFinder.find
        return settingsFinder.find(startParameter);
    }

}

public class DefaultSettingsFinder implements ISettingsFinder {

    private final BuildLayoutFactory layoutFactory;

    public BuildLayout find(StartParameter startParameter) {
				// 委托给了 BuildLayoutFactory.layoutFactory
        return layoutFactory.getLayoutFor(new BuildLayoutConfiguration(startParameter));
    }
}

public class BuildLayoutFactory {

    private static final String DEFAULT_SETTINGS_FILE_BASENAME = "settings";

    public BuildLayout getLayoutFor(BuildLayoutConfiguration configuration) {
				//  1\. 配置中声明了不使用 settings 直接返回
        if (configuration.isUseEmptySettings()) {
            return buildLayoutFrom(configuration, null);
        }
				// 2\. 如果通过 -c 或者 --settings-file 的方式设置好了 settings.gradle 文件直接返回
        File explicitSettingsFile = configuration.getSettingsFile();
        if (explicitSettingsFile != null) {
            if (!explicitSettingsFile.isFile()) {
                throw new MissingResourceException(explicitSettingsFile.toURI(), String.format("Could not read settings file '%s' as it does not exist.", explicitSettingsFile.getAbsolutePath()));
            }
            return buildLayoutFrom(configuration, explicitSettingsFile);
        }
				// 3\. 从当前目录开始, 向上递归查找 settings 文件
        File currentDir = configuration.getCurrentDir();
        boolean searchUpwards = configuration.isSearchUpwards();
        return getLayoutFor(currentDir, searchUpwards ? null : currentDir.getParentFile());
    }

		BuildLayout getLayoutFor(File currentDir, File stopAt) {
        File settingsFile = findExistingSettingsFileIn(currentDir);
        if (settingsFile != null) {
            return layout(currentDir, settingsFile);
        }
        for (File candidate = currentDir.getParentFile(); candidate != null && !candidate.equals(stopAt); candidate = candidate.getParentFile()) {
            settingsFile = findExistingSettingsFileIn(candidate);
            if (settingsFile == null) {
                settingsFile = findExistingSettingsFileIn(new File(candidate, "master"));
            }
            if (settingsFile != null) {
                return layout(candidate, settingsFile);
            }
        }
        return layout(currentDir, new File(currentDir, Settings.DEFAULT_SETTINGS_FILE));
    }

    private BuildLayout layout(File rootDir, File settingsFile) {
        return new BuildLayout(rootDir, settingsFile.getParentFile(), FileUtils.canonicalize(settingsFile));
    }

}

```

查找 `settings.gradle` 文件的流程如下

*   配置中声明了不使用 `settings.gradle` 直接返回
*   如果通过 `-c` 或者 `--settings-file` 的方式设置好了 `settings.gradle` 文件直接返回
*   向上递归查找 `settings.gradle` 文件

### 构建 buildSrc 的 ClassLoader

```
public class BuildSourceBuilder {
    private static final Logger LOGGER = LoggerFactory.getLogger(BuildSourceBuilder.class);
    private static final BuildBuildSrcBuildOperationType.Result BUILD_BUILDSRC_RESULT = new BuildBuildSrcBuildOperationType.Result() {
    };
    public static final String BUILD_SRC = "buildSrc";

		......

    public ClassLoaderScope buildAndCreateClassLoader(File rootDir, StartParameter containingBuildParameters) {
				// 从跟目录中获取 buildSrc 文件夹
        File buildSrcDir = new File(rootDir, DefaultSettings.DEFAULT_BUILD_SRC_DIR);
				// 创建 class path
        ClassPath classpath = createBuildSourceClasspath(buildSrcDir, containingBuildParameters);
				// 创建 ClassLoaderScope
        return classLoaderScope.createChild(buildSrcDir.getAbsolutePath())
            .export(classpath)
            .lock();
    }

}

```

`BuildSourceBuilder.buildAndCreateClassLoader` 会找寻 `buildSrc` 目录, 构建这个目录下的 `classPath`, 后续编译皆可使用

### 解析 settings 文件

解析 settings 文件主要由 `settingsProcessor.process` 发起, `settingsProcessor` 是一个 wrapper 后的对象, 我们这里先关注 `PropertiesLoadingSettingsProcessor`

```
public class PropertiesLoadin
gSettingsProcessor implements SettingsProcessor {
    private final SettingsProcessor processor;
    private final IGradlePropertiesLoader propertiesLoader;

    public PropertiesLoadingSettingsProcessor(SettingsProcessor processor, IGradlePropertiesLoader propertiesLoader) {
        this.processor = processor;
        this.propertiesLoader = propertiesLoader;
    }

    public SettingsInternal process(GradleInternal gradle,
                                    SettingsLocation settingsLocation,
                                    ClassLoaderScope buildRootClassLoaderScope,
                                    StartParameter startParameter) {
				// 根据 setting.gradle 的目录位置, 找寻 properties 文件进行装载
        propertiesLoader.loadProperties(settingsLocation.getSettingsDir());
        return processor.process(gradle, settingsLocation, buildRootClassLoaderScope, startParameter);
    }
}

public class DefaultGradlePropertiesLoader implements IGradlePropertiesLoader {
    private static final Logger LOGGER = LoggerFactory.getLogger(DefaultGradlePropertiesLoader.class);

    private Map<String, String> defaultProperties = new HashMap<String, String>();
    private Map<String, String> overrideProperties = new HashMap<String, String>();
    private final StartParameter startParameter;

    public DefaultGradlePropertiesLoader(StartParameter startParameter) {
        this.startParameter = startParameter;
    }
    public void loadProperties(File settingsDir) {
        loadProperties(settingsDir, startParameter, getAllSystemProperties(), getAllEnvProperties());
    }

    void loadProperties(File settingsDir, StartParameter startParameter, Map<String, String> systemProperties, Map<String, String> envProperties) {
        defaultProperties.clear();
        overrideProperties.clear();
				// gradle home (gradle 具体执行文件夹) 目录下的 gradle.properties
        addGradleProperties(defaultProperties, new File(settingsDir, Project.GRADLE_PROPERTIES));
				// User home 下的（用户 gradle 根目录）文件夹下的gradle.properties
        addGradleProperties(overrideProperties, new File(startParameter.getGradleUserHomeDir(), Project.GRADLE_PROPERTIES));
				// 命令行设置的系统参数
        setSystemProperties(startParameter.getSystemPropertiesArgs());
				// 环境变量配置参数
        overrideProperties.putAll(getEnvProjectProperties(envProperties));
				// 系统配置参数
        overrideProperties.putAll(getSystemProjectProperties(systemProperties));
				// 命令行传递参数
        overrideProperties.putAll(startParameter.getProjectProperties());
    }

}

```

在这里解析如下几种位置的`gradle.properties`

*   Gradle home (gradle 具体执行文件夹) 目录下的 `gradle.properties`
*   User home 下的（用户 gradle 根目录）文件夹下的 `gradle.properties`
*   命令行设置的系统参数
*   环境变量配置参数
*   系统配置参数
*   命令行传递参数

接下来看看 `ScriptEvaluatingSettingsProcessor` 这个 processor 的 `process` 方法

```
public class ScriptEvaluatingSettingsProcessor implements SettingsProcessor {
    private static final Logger LOGGER = LoggerFactory.getLogger(ScriptEvaluatingSettingsProcessor.class);

    public SettingsInternal process(GradleInternal gradle,
                                    SettingsLocation settingsLocation,
                                    ClassLoaderScope buildRootClassLoaderScope,
                                    StartParameter startParameter) {
        Timer settingsProcessingClock = Time.startTimer();
        Map<String, String> properties = propertiesLoader.mergeProperties(Collections.<String, String>emptyMap());
				// 创建 DefaultSettings 对象
        SettingsInternal settings = settingsFactory.createSettings(gradle, settingsLocation.getSettingsDir(),
                settingsLocation.getSettingsScriptSource(), properties, startParameter, buildRootClassLoaderScope);
				// 解析编译脚本的参数到对象中暂存
        applySettingsScript(settingsLocation, settings);
        LOGGER.debug("Timing: Processing settings took: {}", settingsProcessingClock.getElapsed());
        return settings;
    }

}

```

这里会将一些参数合并到 `DefaultSettings` 对象保存,

接下来看看 `configureBuild` 的流程

# configureBuild

```
public class DefaultGradleLauncher implements GradleLauncher {

		private void configureBuild() {
        if (stage == Stage.Load) {
            buildOperationExecutor.run(new ConfigureBuild());

            stage = Stage.Configure;
        }
    }

    private class ConfigureBuild implements RunnableBuildOperation {
        @Override
        public void run(BuildOperationContext context) {
						// 加载 settings 中 include 的项目, 生成对应的 project
            buildLoader.load(settings, gradle);
						// 配置生成的 projects
            buildConfigurer.configure(gradle);
						......
            context.setResult(CONFIGURE_BUILD_RESULT);
        }

    }

}

```

可以看到 Configure 阶段主要有两个步骤

*   加载 settings 中 include 的项目, 生成对应的 project
*   配置生成的 projects

## 生成 Project

```
public class InstantiatingBuildLoader implements BuildLoader {
    private final IProjectFactory projectFactory;

    public InstantiatingBuildLoader(IProjectFactory projectFactory) {
        this.projectFactory = projectFactory;
    }

    @Override
    public void load(SettingsInternal settings, GradleInternal gradle) {
        load(settings.getRootProject(), settings.getDefaultProject(), gradle, settings.getRootClassLoaderScope());
    }

    private void load(ProjectDescriptor rootProjectDescriptor, ProjectDescriptor defaultProject, GradleInternal gradle, ClassLoaderScope buildRootClassLoaderScope) {
				// 递归创建 Projects
        createProjects(rootProjectDescriptor, gradle, buildRootClassLoaderScope);
				// 绑定默认的 project, 即 root project.
        attachDefaultProject(defaultProject, gradle);
    }

    private void attachDefaultProject(ProjectDescriptor defaultProject, GradleInternal gradle) {
        gradle.setDefaultProject(gradle.getRootProject().getProjectRegistry().getProject(defaultProject.getPath()));
    }

    private void createProjects(ProjectDescriptor rootProjectDescriptor, GradleInternal gradle, ClassLoaderScope buildRootClassLoaderScope) {
				// 1\. 创建 root project.
        ProjectInternal rootProject = projectFactory.createProject(rootProjectDescriptor, null, gradle, buildRootClassLoaderScope.createChild("root-project"), buildRootClassLoaderScope);
				// 2 保存 root project 到 gradle 中
        gradle.setRootProject(rootProject);
				// 3\. 递归创建子 project
        addProjects(rootProject, rootProjectDescriptor, gradle, buildRootClassLoaderScope);
    }

    private void addProjects(ProjectInternal parent, ProjectDescriptor parentProjectDescriptor, GradleInternal gradle, ClassLoaderScope buildRootClassLoaderScope) {
        for (ProjectDescriptor childProjectDescriptor : parentProjectDescriptor.getChildren()) {
            ProjectInternal childProject = projectFactory.createProject(childProjectDescriptor, parent, gradle, parent.getClassLoaderScope().createChild("project-" + childProjectDescriptor.getName()), buildRootClassLoaderScope);
            addProjects(childProject, childProjectDescriptor, gradle, buildRootClassLoaderScope);
        }
    }
}

```

BuildLoader.load 方法主要做了如下几件事情

*   创建 root project 对象 `DefaultProject`
*   保存 root project 到 gradle 中
*   递归创建子 project 对象 `DefaultProject`

## 配置 Project

```
public class DefaultBuildConfigurer implements BuildConfigurer {
    private final ProjectConfigurer projectConfigurer;
    private final BuildStateRegistry buildRegistry;

    public DefaultBuildConfigurer(ProjectConfigurer projectConfigurer, BuildStateRegistry buildRegistry) {
        this.projectConfigurer = projectConfigurer;
        this.buildRegistry = buildRegistry;
    }

    public void configure(GradleInternal gradle) {
        maybeInformAboutIncubatingMode(gradle);
        if (gradle.getParent() == null) {
            buildRegistry.beforeConfigureRootBuild();
        }
				// 如果设置了参数 --configure-on-demand 进行按需构建，则只会配置 root project 和 tasks 需要的 projects
        if (gradle.getStartParameter().isConfigureOnDemand()) {
            projectConfigurer.configure(gradle.getRootProject());
        } else {
						// 配置每一个 project
            projectConfigurer.configureHierarchy(gradle.getRootProject());
        }
    }
}
public class TaskPathProjectEvaluator implements ProjectConfigurer {

		public void configureHierarchy(ProjectInternal project) {
        configure(project);
				// 递归调用子 project 的 configure.
        for (Project sub : project.getSubprojects()) {
            configure((ProjectInternal) sub);
        }
    }

		public void configure(ProjectInternal project) {
        if (cancellationToken.isCancellationRequested()) {
            throw new BuildCancelledException();
        }
				// 调用 project 的 evaluate.
        project.evaluate();
    }
}

```

配置 Project 的流程主要是递归调用了 `Project.evaluate()`

Project 的 evaluate 操作会执行当前的 Project 注入的 plugin 的 apply 方法, 会为当前的 project 注入一些默认的 task, 这里就不详细跟下去看了

# 构建 TaskGraph

```
public class DefaultGradleLauncher implements GradleLauncher {

		private void constructTaskGraph() {
        if (stage == Stage.Configure) {
            buildOperationExecutor.run(new CalculateTaskGraph());

            stage = Stage.TaskGraph;
        }
    }

    private class CalculateTaskGraph implements RunnableBuildOperation {
        @Override
        public void run(BuildOperationContext buildOperationContext) {
						// 1\. 这里调用了 buildConfigurationActionExecuter.select 方法
            buildConfigurationActionExecuter.select(gradle);
						......
            final TaskExecutionGraphInternal taskGraph = gradle.getTaskGraph();
					  // 2\. 构建有向无环图
            taskGraph.populate();

            includedBuildControllers.populateTaskGraphs();
						.......
        }

    }

}

```

`constructTaskGraph` 方法中主要有两个操作

*   `buildConfigurationActionExecuter.select` 收集要执行的 task
*   `taskGraph.populate()` 构建一个有向无环图

这里我们主要关注一下 `buildConfigurationActionExecuter.select` 收集要执行的 task 的过程

```
public class DefaultBuildConfigurationActionExecuter implements BuildConfigurationActionExecuter {
    private final List<BuildConfigurationAction> configurationActions;
    private List<? extends BuildConfigurationAction> taskSelectors;

    public void select(GradleInternal gradle) {
				// 1\. 获取 BuildConfigurationAction 集合
        List<BuildConfigurationAction> processingBuildActions = CollectionUtils.flattenCollections(
								BuildConfigurationAction.class, configurationActions, taskSelectors);
				// 2\. 执行每个 action 的 configure.
        configure(processingBuildActions, gradle, 0);
    }

    private void configure(final List<BuildConfigurationAction> processingConfigurationActions, final GradleInternal gradle, final int index) {
        if (index >= processingConfigurationActions.size()) {
            return;
        }
				// 2.1 configure 每一个 action.
        processingConfigurationActions.get(index).configure(new BuildExecutionContext() {
            public GradleInternal getGradle() {
                return gradle;
            }

            public void proceed() {
								// 2.2 执行下一个 action 的 configure.
                configure(processingConfigurationActions, gradle, index + 1);
            }

        });
    }
}

```

可以看到 `[buildConfigurationActionExecuter.select](<http://buildconfigurationactionexecuter.select>)` 函数主要是找到 BuildConfigurationAction 集合, 然后执行每一个 Action 的 configure 方法, 这个 BuildConfigurationAction 主要有如下三个

*   `ExcludedTaskFilteringBuildConfigurationAction`: 添加任务过滤器
*   `DefaultTasksBuildExecutionAction`: 确定要执行的任务
*   `TaskNameResolvingBuildConfigurationAction`: 添加关联任务

这里我们逐个分析

## 添加任务过滤器

```
public class ExcludedTaskFilteringBuildConfigurationAction implements BuildConfigurationAction {

    private final TaskSelector taskSelector;

    public void configure(BuildExecutionContext context) {
        GradleInternal gradle = context.getGradle();
				// 1\. 这里会获取通过参数 -x 或者 --exclude-task 指定的 tasks
        Set<String> excludedTaskNames = gradle.getStartParameter().getExcludedTaskNames();
        if (!excludedTaskNames.isEmpty()) {
            final Set<Spec<Task>> filters = new HashSet<Spec<Task>>();
            for (String taskName : excludedTaskNames) {
								// 2\. 解析这些 tasks，封装成 Spec<Task>
                filters.add(taskSelector.getFilter(taskName));
            }
						// 3\. 通过 useFilter() 将 Spec 设置给 TaskGraph
            gradle.getTaskGraph().useFilter(Specs.intersect(filters));
        }
				// 执行下一个 action
        context.proceed();
    }
}

```

`ExcludedTaskFilteringBuildConfigurationAction.configure` 处理的事情如下

*   这里会获取通过参数 `-x` 或者 `--exclude-task` 指定的 `tasks`
*   解析这些 `tasks`，封装成 `Spec<Task>`
*   通过 `useFilter()` 将 `Spec` 设置给 `TaskGraph`

## 确认要执行的任务

```
public class DefaultTasksBuildExecutionAction implements BuildConfigurationAction {
    private static final Logger LOGGER = LoggerFactory.getLogger(DefaultTasksBuildExecutionAction.class);
    private final ProjectConfigurer projectConfigurer;

    public DefaultTasksBuildExecutionAction(ProjectConfigurer projectConfigurer) {
        this.projectConfigurer = projectConfigurer;
    }

    public void configure(BuildExecutionContext context) {
        StartParameter startParameter = context.getGradle().getStartParameter();
				// 1\. 首先看有没有指定执行的 task，如果有指定执行的 task，则直接返回；
				// 比如 ./gradlew clean，指定了需要执行 clean task，这里的 args 即 clean
        for (TaskExecutionRequest request : startParameter.getTaskRequests()) {
            if (!request.getArgs().isEmpty()) {
                context.proceed();
                return;
            }
        }

        // 2\. 如果没有指定要执行的 task，则获取 default project 的 default tasks
				// 比如执行 ./gradlew ，这种就是没有指定执行的 task
        ProjectInternal project = context.getGradle().getDefaultProject();

        //so that we don't miss out default tasks
        projectConfigurer.configure(project);

        List<String> defaultTasks = project.getDefaultTasks();
				// 3\. 如果 default project 没有设置 default tasks，则将 help 任务设置为默认任务
        if (defaultTasks.size() == 0) {
            defaultTasks = Collections.singletonList(ProjectInternal.HELP_TASK);
            LOGGER.info("No tasks specified. Using default task {}", GUtil.toString(defaultTasks));
        }
				// 设置需要执行的 tasks
        startParameter.setTaskNames(defaultTasks);
				// 执行下一个 action
        context.proceed();
    }
}

```

`DefaultTasksBuildExecutionAction.configure` 主要是确认要执行的任务, 具体细节如下

*   首先看有没有指定执行的 task，如果有指定执行的 task，则直接返回
*   如果没有指定要执行的 task，则获取 default project 的 default tasks
    *   如果 default project 没有设置 default tasks，则指定为 help task

## 添加关联任务

```
/**
 * A {@link BuildConfigurationAction} which selects tasks which match the provided names. For each name, selects all tasks in all
 * projects whose name is the given name.
 */
public class TaskNameResolvingBuildConfigurationAction implements BuildConfigurationAction {
    private static final Logger LOGGER = LoggerFactory.getLogger(TaskNameResolvingBuildConfigurationAction.class);
    private final CommandLineTaskParser commandLineTaskParser;

    public void configure(BuildExecutionContext context) {
        GradleInternal gradle = context.getGradle();
        TaskExecutionGraphInternal taskGraph = gradle.getTaskGraph();
				// 1\. 获取指定执行的 TaskExecutionRequest
        List<TaskExecutionRequest> taskParameters = gradle.getStartParameter().getTaskRequests();
        for (TaskExecutionRequest taskParameter : taskParameters) {
						// 2\. 通过 commandLineTaskParser.parseTasks 解析与指定 task 依赖的 task
            List<TaskSelector.TaskSelection> taskSelections = commandLineTaskParser.parseTasks(taskParameter);
            for (TaskSelector.TaskSelection taskSelection : taskSelections) {
								// 3\. 添加所有符合条件的 task 到 TaskGraph
                LOGGER.info("Selected primary task '{}' from project {}", taskSelection.getTaskName(), taskSelection.getProjectPath());
                taskGraph.addTasks(taskSelection.getTasks());
            }
        }

        context.proceed();
    }

}

```

`TaskNameResolvingBuildConfigurationAction.configure` 中主要工作是收集指定任务有关联的任务并添加到 taskGraph 中

### 解析 tasks

```
public class CommandLineTaskParser {
    private final CommandLineTaskConfigurer taskConfigurer;
    private final TaskSelector taskSelector;

    public List<TaskSelector.TaskSelection> parseTasks(TaskExecutionRequest taskExecutionRequest) {
        List<TaskSelector.TaskSelection> out = Lists.newArrayList();
				// 比如 :app:clean, args 即 [:app:clean]
        List<String> remainingPaths = new LinkedList<String>(taskExecutionRequest.getArgs());
        while (!remainingPaths.isEmpty()) {
            String path = remainingPaths.remove(0);
						// 查找所有符合条件的 tasks
            TaskSelector.TaskSelection selection = taskSelector.getSelection(taskExecutionRequest.getProjectPath(), taskExecutionRequest.getRootDir(), path);
            Set<Task> tasks = selection.getTasks();
            remainingPaths = taskConfigurer.configureTasks(tasks, remainingPaths);
            out.add(selection);
        }
        return out;
    }

}

```

### 添加到 taskGraph

```
@NonNullApi
public class DefaultTaskExecutionGraph implements TaskExecutionGraphInternal {

		public void addTasks(Iterable<? extends Task> tasks) {
        assert tasks != null;

        final Timer clock = Time.startTimer();

        Set<Task> taskSet = new LinkedHashSet<Task>();
        for (Task task : tasks) {
            taskSet.add(task);
            requestedTasks.add(task);
        }
				// 这里调用 DefaultTaskExecutionPlan.addToTaskGraph 来缓存 tasks
        taskExecutionPlan.addToTaskGraph(taskSet);
        taskGraphState = TaskGraphState.DIRTY;

        LOGGER.debug("Timing: Creating the DAG took " + clock.getElapsed());
    }

}

```

`DefaultTaskExecutionGraph.addTasks` 中将添加任务的操作转发给了`DefaultTaskExecutionPlan.addToTaskGraph`

下面看看它的具体实现

```
public class DefaultTaskExecutionPlan implements TaskExecutionPlan {

		public void addToTaskGraph(Collection<? extends Task> tasks) {
        final Deque<WorkInfo> queue = new ArrayDeque<WorkInfo>();

        List<Task> sortedTasks = new ArrayList<Task>(tasks);
        Collections.sort(sortedTasks);
        for (Task task : sortedTasks) {
            TaskInfo node = nodeFactory.getOrCreateNode(task);
            if (node.isMustNotRun()) {
                requireWithDependencies(node);
            } else if (filter.isSatisfiedBy(task)) {
                node.require();
            }
            entryTasks.add(node);
            queue.add(node);
        }

        final Set<WorkInfo> visiting = Sets.newHashSet();

        while (!queue.isEmpty()) {
            WorkInfo node = queue.getFirst();
            if (node.getDependenciesProcessed()) {
                // Have already visited this task - skip it
								// 已经添加过了
                queue.removeFirst();
                continue;
            }

            boolean filtered = !nodeSatisfiesTaskFilter(node);
            if (filtered) {
                // Task is not required - skip it
								// 已经处理过了
                queue.removeFirst();
                node.dependenciesProcessed();
                node.doNotRequire();
                filteredNodes.add(node);
                continue;
            }

            if (visiting.add(node)) {
                // Have not seen this task before - add its dependencies to the head of the queue and leave this
                // task in the queue
                // Make sure it has been configured
								// 处理任务之间的依赖关系
                node.prepareForExecution();
                node.resolveDependencies(dependencyResolver, new Action<WorkInfo>() {
                    @Override
                    public void execute(WorkInfo targetNode) {
                        if (!visiting.contains(targetNode)) {
                            queue.addFirst(targetNode);
                        }
                    }
                });
                if (node.isRequired()) {
                    for (WorkInfo successor : node.getDependencySuccessors()) {
                        if (nodeSatisfiesTaskFilter(successor)) {
                            successor.require();
                        }
                    }
                } else {
                    workInUnknownState.add(node);
                }
            } else {
                // Have visited this task's dependencies - add it to the graph
                queue.removeFirst();
                visiting.remove(node);
                node.dependenciesProcessed();
            }
        }
        resolveWorkInUnknownState();
    }
}

```

这里主要关注一下 resolveDependencies 中的逻辑实现

```
public class LocalTaskInfo extends TaskInfo {
		@Override
    public void resolveDependencies(TaskDependencyResolver dependencyResolver, Action<WorkInfo> processHardSuccessor) {
				// 处理 dependsOn
        for (WorkInfo targetNode : getDependencies(dependencyResolver)) {
            addDependencySuccessor(targetNode);
            processHardSuccessor.execute(targetNode);
        }
				// 处理 finalizedBy
        for (WorkInfo targetNode : getFinalizedBy(dependencyResolver)) {
            if (!(targetNode instanceof TaskInfo)) {
                throw new IllegalStateException("Only tasks can be finalizers: " + targetNode);
            }
            addFinalizerNode((TaskInfo) targetNode);
            processHardSuccessor.execute(targetNode);
        }
				// 处理 mustRunAfter
        for (WorkInfo targetNode : getMustRunAfter(dependencyResolver)) {
            addMustSuccessor(targetNode);
        }
				// 处理 shouldRunAfter
        for (WorkInfo targetNode : getShouldRunAfter(dependencyResolver)) {
            addShouldSuccessor(targetNode);
        }
    }
}

```

可以看到这里的就是对 task 之间依赖关系进行处理逻辑

# runTasks

```
public class DefaultGradleLauncher implements GradleLauncher {

		private void runTasks() {
        if (stage != Stage.TaskGraph) {
            throw new IllegalStateException("Cannot execute tasks: current stage = " + stage);
        }

        buildOperationExecutor.run(new ExecuteTasks());

        stage = Stage.Build;
    }

    private class ExecuteTasks implements RunnableBuildOperation {
        @Override
        public void run(BuildOperationContext context) {
            includedBuildControllers.startTaskExecution();
            List<Throwable> taskFailures = new ArrayList<Throwable>();
						// 这里调用了 buildExecuter.execute 执行了后续的执行任务
            buildExecuter.execute(gradle, taskFailures);
            includedBuildControllers.awaitTaskCompletion(taskFailures);
            if (!taskFailures.isEmpty()) {
                throw new MultipleBuildFailures(taskFailures);
            }
        }
    }

}

```

执行任务的操作委托给了 `DefaultBuildExecuter.execute` 执行, 它的实现如下

```
public class DefaultBuildExecuter implements BuildExecuter {
    private final List<BuildExecutionAction> executionActions;

    public DefaultBuildExecuter(Iterable<? extends BuildExecutionAction> executionActions) {
        this.executionActions = Lists.newArrayList(executionActions);
    }

    @Override
    public void execute(GradleInternal gradle, Collection<? super Throwable> taskFailures) {
        execute(gradle, 0, taskFailures);
    }

    private void execute(final GradleInternal gradle, final int index, final Collection<? super Throwable> taskFailures) {
        if (index >= executionActions.size()) {
            return;
        }
        executionActions.get(index).execute(new BuildExecutionContext() {
            public GradleInternal getGradle() {
                return gradle;
            }

            public void proceed() {
                execute(gradle, index + 1, taskFailures);
            }

        }, taskFailures);
    }
}

```

与 `DefaultBuildConfigurationActionExecuter.execute` 实现类似, 这里会挨个执行 `BuildExecutionAction` 分别如下

*   `DryRunBuildExecutionAction`
*   `SelectedTaskExecutionAction`

这里我们逐一分析

## DryRunBuildExecutionAction

```
public class DryRunBuildExecutionAction implements BuildExecutionAction {
    private final StyledTextOutputFactory textOutputFactory;

    public DryRunBuildExecutionAction(StyledTextOutputFactory textOutputFactory) {
        this.textOutputFactory = textOutputFactory;
    }

    @Override
    public void execute(BuildExecutionContext context, Collection<? super Throwable> taskFailures) {
        GradleInternal gradle = context.getGradle();
				// 1\. 判断是否有 -m 或者 --dry-run 参数
        if (gradle.getStartParameter().isDryRun()) {
						// 1.1 直接拦截 task 的执行操作, 改为打印 task 的 name 和运行顺序
            for (Task task : gradle.getTaskGraph().getAllTasks()) {
                textOutputFactory.create(DryRunBuildExecutionAction.class)
                    .append(((TaskInternal) task).getIdentityPath().getPath())
                    .append(" ")
                    .style(StyledTextOutput.Style.ProgressStatus)
                    .append("SKIPPED")
                    .println();
            }
        }
				//  2\. 往下分发
				else {
            context.proceed();
        }
    }
}

```

`DryRunBuildExecutionAction` 主要拦截处理了 `-m` 或者 `--dry-run` 参数, 如果有这个参数便会拦截执行, 直接打印 task 的 path

## SelectedTaskExecutionAction

```
public class SelectedTaskExecutionAction implements BuildExecutionAction {
    @Override
    public void execute(BuildExecutionContext context, Collection<? super Throwable> taskFailures) {
        GradleInternal gradle = context.getGradle();
        TaskExecutionGraphInternal taskGraph = gradle.getTaskGraph();
				// 1\. 如果参数中是否有 --continue, 则任务失败会继续往下执行
        if (gradle.getStartParameter().isContinueOnFailure()) {
            taskGraph.setContinueOnFailure(true);
        }

        taskGraph.addTaskExecutionGraphListener(new BindAllReferencesOfProjectsToExecuteListener());
				// 2\. 交付给 DefaultTaskExecutionGraph.execute 继续执行
        taskGraph.execute(taskFailures);
    }
}

```

这里主要做了两件事情

*   如果存在 `--continue`, 则为 `DefaultTaskExecutionGraph` 设置 `continueOnFailre` 属性, 表明任务失败了也会继续往下执行
*   交由 `DefaultTaskExecutionGraph.execute` 继续往下分发

```
public class DefaultTaskExecutionGraph implements TaskExecutionGraphInternal {

		@Override
    public void execute(Collection<? super Throwable> failures) {
        Timer clock = Time.startTimer();
				// 确保已经生成有向无环图
        ensurePopulated();
        buildOperationExecutor.run(new NotifyTaskGraphWhenReady(this, graphListeners, gradleInternal));
        try {
						// 交由 taskPlanExecutor 处理后续工作
            taskPlanExecutor.process(taskExecutionPlan, failures, new BuildOperationAwareWorkItemExecutor(workInfoExecutors, buildOperationExecutor.getCurrentOperation()));
            LOGGER.debug("Timing: Executing the DAG took " + clock.getElapsed());
        } finally {
            coordinationService.withStateLock(new Transformer<ResourceLockState.Disposition, ResourceLockState>() {
                @Override
                public ResourceLockState.Disposition transform(ResourceLockState resourceLockState) {
                    taskExecutionPlan.clear();
                    return ResourceLockState.Disposition.FINISHED;
                }
            });
        }
    }

}

```

`taskPlanExecutor` 是 `DefaultPlanExecutor` 实例, 这里我们继续分析它的 process 实现

```
@NonNullApi
public class DefaultTaskPlanExecutor implements TaskPlanExecutor {

		@Override
    public void process(TaskExecutionPlan taskExecutionPlan, Collection<? super Throwable> failures, Action<WorkInfo> workExecutor) {
				// 创建线程池
        ManagedExecutor executor = executorFactory.create("Task worker for '" + taskExecutionPlan.getDisplayName() + "'");
        try {
            WorkerLease parentWorkerLease = workerLeaseService.getCurrentWorkerLease();
						// 1\. org.gradle.parallel=true 的时候会开启多个 ExecutorWorker 并行构建
            startAdditionalWorkers(taskExecutionPlan, workExecutor, executor, parentWorkerLease);
						// 2\. 调用 ExecutorWorker.run
            new ExecutorWorker(taskExecutionPlan, workExecutor, parentWorkerLease, cancellationToken, coordinationService).run();
            awaitCompletion(taskExecutionPlan, failures);
        } finally {
            executor.stop();
        }
    }

		private static class ExecutorWorker implements Runnable {
        private final TaskExecutionPlan taskExecutionPlan;
        private final Action<? super WorkInfo> workExecutor;
        private final WorkerLease parentWorkerLease;
        private final BuildCancellationToken cancellationToken;
        private final ResourceLockCoordinationService coordinationService;

        @Override
        public void run() {
            ......
						// 2.1 遍历每一个工作项
            WorkerLease childLease = parentWorkerLease.createChild();
            boolean moreTasksToExecute = true;
            while (moreTasksToExecute) {
                moreTasksToExecute = executeWithWork(childLease, new Action<WorkInfo>() {
                    @Override
                    public void execute(WorkInfo work) {
                        ......
                        taskTimer.reset();
												// 交付给 workExecutor 处理每个任务
                        workExecutor.execute(work);
                        long taskDuration = taskTimer.getElapsedMillis();
                        ......
                    }
                });
            }
            long total = totalTimer.getElapsedMillis();
						......
        }

				private boolean executeWithWork(final WorkerLease workerLease, final Action<WorkInfo> workExecutor) {
            final MutableReference<WorkInfo> selected = MutableReference.empty();
            final MutableBoolean workRemaining = new MutableBoolean();
            coordinationService.withStateLock(new Transformer<ResourceLockState.Disposition, ResourceLockState>() {
                @Override
                public ResourceLockState.Disposition transform(ResourceLockState resourceLockState) {
										// 是否取消执行
                    if (cancellationToken.isCancellationRequested()) {
                        taskExecutionPlan.cancelExecution();
                    }

                    workRemaining.set(taskExecutionPlan.hasWorkRemaining());
										// 标记为结束
                    if (!workRemaining.get()) {
                        return FINISHED;
                    }

                    try {
                        selected.set(taskExecutionPlan.selectNext(workerLease, resourceLockState));
                    } catch (Throwable t) {
                        resourceLockState.releaseLocks();
                        taskExecutionPlan.abortAllAndFail(t);
                        workRemaining.set(false);
                    }

                    if (selected.get() == null && workRemaining.get()) {
                        return RETRY;
                    } else {
                        return FINISHED;
                    }
                }
            });

            WorkInfo selectedWorkInfo = selected.get();
						// 2.2 执行这个 workInfo.
            if (selectedWorkInfo != null) {
                execute(selectedWorkInfo, workerLease, workExecutor);
            }
            return workRemaining.get();
        }

        private void execute(final WorkInfo selected, final WorkerLease workerLease, Action<WorkInfo> workExecutor) {
            try {
                if (!selected.isComplete()) {
                    try {
												// 2.2.1 执行 ExecutorWorker 里面创建的 Action 匿名内部类实例的 execute()
                        workExecutor.execute(selected);
                    } catch (Throwable e) {
                        selected.setExecutionFailure(e);
                    }
                }
            } finally {
               ......
            }
        }

		}
}

```

`DefaultTaskPlanExecutor.process` 最终会将任务交付给 `workExecutor.execute` 处理每一个选中的任务

`workExecutor` 是一个 `BuildOperationAwareWorkItemExecutor` 的实例对象, 它的实现如下

```
@NonNullApi
public class DefaultTaskExecutionGraph implements TaskExecutionGraphInternal {

		/**
     * This action executes a task via the task executer wrapping everything into a build operation.
     */
    private class BuildOperationAwareWorkItemExecutor implements Action<WorkInfo> {
        private final BuildOperationRef parentOperation;
        private final List<WorkInfoExecutor> workInfoExecutors;

        BuildOperationAwareWorkItemExecutor(List<WorkInfoExecutor> workInfoExecutors, BuildOperationRef parentOperation) {
            this.workInfoExecutors = workInfoExecutors;
            this.parentOperation = parentOperation;
        }

        @Override
        public void execute(WorkInfo work) {
            BuildOperationRef previous = CurrentBuildOperationRef.instance().get();
            CurrentBuildOperationRef.instance().set(parentOperation);
						// 将当前任务交给一组 WorkInfoExecutor 执行
            try {
                for (WorkInfoExecutor workInfoExecutor : workInfoExecutors) {
                    if (workInfoExecutor.execute(work)) {
                        return;
                    }
                }
                throw new IllegalStateException("Unknown type of work: " + work);
            } finally {
                CurrentBuildOperationRef.instance().set(previous);
            }
        }
    }

}

```

这里 `BuildOperationAwareWorkItemExecutor.execute` 将 `WorkInfo` 交给一组 `WorkInfoExecutor` 执行, 分别如下

*   `LocalTaskInfoExecutor`
*   `TransformInfoExecutor`

我们主要关注下 `LocalTaskInfoExecutor` 的实现

```
public class LocalTaskInfoExecutor implements WorkInfoExecutor {
    // This currently needs to be lazy, as it uses state that is not available when the graph is created
    private final Factory<? extends TaskExecuter> taskExecuterFactory;

    public LocalTaskInfoExecutor(Factory<? extends TaskExecuter> taskExecuterFactory) {
        this.taskExecuterFactory = taskExecuterFactory;
    }

    @Override
    public boolean execute(WorkInfo work) {
        if (work instanceof LocalTaskInfo) {
            TaskInternal task = ((LocalTaskInfo) work).getTask();
            TaskStateInternal state = task.getState();
            TaskExecutionContext ctx = new DefaultTaskExecutionContext();
						// 通过 TaskExecutionServices 创建了 TaskExecuter 执行该任务
            TaskExecuter taskExecuter = taskExecuterFactory.create();
            assert taskExecuter != null;
            taskExecuter.execute(task, state, ctx);
            return true;
        } else {
            return false;
        }
    }
}

```

LocalTaskInfoExecutor.execute 将执行操作委托给了 TaskExecuter 执行最终的操作, 我们看看它的创建流程

```
public class TaskExecutionServices {

    TaskExecuter createTaskExecuter(TaskArtifactStateRepository repository,
                                    TaskOutputCacheCommandFactory taskOutputCacheCommandFactory,
                                    BuildCacheController buildCacheController,
                                    ListenerManager listenerManager,
                                    TaskInputsListener inputsListener,
                                    BuildOperationExecutor buildOperationExecutor,
                                    AsyncWorkTracker asyncWorkTracker,
                                    BuildOutputCleanupRegistry cleanupRegistry,
                                    TaskOutputFilesRepository taskOutputFilesRepository,
                                    BuildScanPluginApplied buildScanPlugin,
                                    PathToFileResolver resolver,
                                    PropertyWalker propertyWalker,
                                    TaskExecutionGraphInternal taskExecutionGraph,
                                    BuildInvocationScopeId buildInvocationScopeId,
                                    BuildCancellationToken buildCancellationToken
    ) {

        boolean buildCacheEnabled = buildCacheController.isEnabled();
        boolean scanPluginApplied = buildScanPlugin.isBuildScanPluginApplied();
        TaskOutputChangesListener taskOutputChangesListener = listenerManager.getBroadcaster(TaskOutputChangesListener.class);

        TaskExecuter executer = new ExecuteActionsTaskExecuter(
            taskOutputChangesListener,
            listenerManager.getBroadcaster(TaskActionListener.class),
            buildOperationExecutor,
            asyncWorkTracker,
            buildInvocationScopeId,
            buildCancellationToken
        );
        executer = new OutputDirectoryCreatingTaskExecuter(executer);
        if (buildCacheEnabled) {
            executer = new SkipCachedTaskExecuter(
                buildCacheController,
                taskOutputChangesListener,
                taskOutputCacheCommandFactory,
                executer
            );
        }
        executer = new SkipUpToDateTaskExecuter(executer);
        executer = new ResolveTaskOutputCachingStateExecuter(buildCacheEnabled, executer);
        if (buildCacheEnabled || scanPluginApplied) {
            executer = new ResolveBuildCacheKeyExecuter(executer, buildOperationExecutor, buildCacheController.isEmitDebugLogging());
        }
        executer = new ValidatingTaskExecuter(executer);
        executer = new SkipEmptySourceFilesTaskExecuter(inputsListener, cleanupRegistry, taskOutputChangesListener, executer, buildInvocationScopeId);
        executer = new FinalizeInputFilePropertiesTaskExecuter(executer);
        executer = new CleanupStaleOutputsExecuter(cleanupRegistry, taskOutputFilesRepository, buildOperationExecutor, taskOutputChangesListener, executer);
        executer = new ResolveTaskArtifactStateTaskExecuter(repository, resolver, propertyWalker, executer);
        executer = new SkipTaskWithNoActionsExecuter(taskExecutionGraph, executer);
        executer = new SkipOnlyIfTaskExecuter(executer);
        executer = new ExecuteAtMostOnceTaskExecuter(executer);
        executer = new CatchExceptionTaskExecuter(executer);
        executer = new EventFiringTaskExecuter(buildOperationExecutor, taskExecutionGraph, executer);
        return executer;
    }
}

```

任务执行器通过装饰者设计层层包装, 其中子 `Executer` 的作用如下

*   `EventFiringTaskExecuter`: 回调一个构建之前的监听
*   `CatchExceptionTaskExecuter`: 为这个任务添加`try -catch`
*   `SkipOnlyIfTaskExecuter`: 判断到在task中设置`onlyif`方法，根据该boolean方法判断当前的Task是否跳过。
*   `SkipTaskWithNoActionsExecuter`: 跳过那些通过`dry-run`之类的方法，蒋设置为`skip`的任务
*   `ResolveTaskExecutionModeExecuter`: 解析Task属性，如输出和输出。获取Task的`Execution Mode`执行模式。执行模式决定了当前的Task是否加载缓存代替执行任务，决定是否记录缓存。
*   `SkipEmptySourceFilesTaskExecuter`: 跳过那些没有任何输入文件的任务
*   `ValidatingTaskExecuter`: 对 Task 的属性进行校验

最终会调用到 `ExecuteActionsTaskExecuter.execute` 中

```
public class ExecuteActionsTaskExecuter implements TaskExecuter {
    ......

    public void execute(TaskInternal task, TaskStateInternal state, TaskExecutionContext context) {
	      //  1\. 回调 TaskActionListener.beforeActions 通知外界任务即将开始执行
        listener.beforeActions(task);
        if (task.hasTaskActions()) {
            outputsGenerationListener.beforeTaskOutputChanged();
        }
        state.setExecuting(true);
        try {
						// 2\. 执行 task 的 actions.
            GradleException failure = executeActions(task, state, context);
            if (failure != null) {
                state.setOutcome(failure);
            } else {
                state.setOutcome(
                    state.getDidWork() ? TaskExecutionOutcome.EXECUTED : TaskExecutionOutcome.UP_TO_DATE
                );
            }
            context.getTaskArtifactState().snapshotAfterTaskExecution(failure, buildInvocationScopeId.getId(), context);
        } finally {
            state.setExecuting(false);
						// 3\. 回调 TaskActionListener.afterActions 通知外界任务执行完毕
            listener.afterActions(task);
        }
    }

    private GradleException executeActions(TaskInternal task, TaskStateInternal state, TaskExecutionContext context) {
        LOGGER.debug("Executing actions for {}.", task);
				// 2.1 遍历 task 中的 Action 依次执行
        final List<ContextAwareTaskAction> actions = new ArrayList<ContextAwareTaskAction>(task.getTaskActions());
        for (ContextAwareTaskAction action : actions) {
            state.setDidWork(true);
            task.getStandardOutputCapture().start();
            try {
                executeAction(action.getDisplayName(), task, action, context);
                if (buildCancellationToken.isCancellationRequested()) {
                    return new BuildCancelledException("Build cancelled during task: " + task.getName());
                }
            } catch (StopActionException e) {
                // Ignore
                LOGGER.debug("Action stopped by some action with message: {}", e.getMessage());
            } catch (StopExecutionException e) {
                LOGGER.info("Execution stopped by some action with message: {}", e.getMessage());
                break;
            } catch (Throwable t) {
                return new TaskExecutionException(task, t);
            } finally {
                task.getStandardOutputCapture().stop();
            }
        }
        return null;
    }
}

```

主要有三个步骤

*   回调 `TaskActionListener.beforeActions` 通知外界任务即将开始执行
*   执行 `task` 的 `actions`
*   回调 `TaskActionListener.afterActions` 通知外界任务执行完毕

# finishBuild

```
public class DefaultGradleLauncher implements GradleLauncher {

		@Override
    public void finishBuild() {
        if (stage != null) {
            finishBuild(new BuildResult(stage.name(), gradle, null));
        }
    }

		private void finishBuild(BuildResult result) {
        if (stage == Stage.Finished) {
            return;
        }

        includedBuildControllers.finishBuild();
				// 通知外界执行完毕
        buildListener.buildFinished(result);

        stage = Stage.Finished;
    }

}

```

`finishBuild` 的逻辑十分简单, 最终会回调 `BuildListener.buildFinished` 通知外界执行完毕了

# 总结

这里借用大佬绘制的一张图

[图片上传失败...(image-486c29-1618109951139)]Gradle

*   `loadSettings`
    *   回调 `buildStarted`
    *   执行 `LoadBuild` 中的 `run` 方法
        *   加载 `init` 文件夹下的脚本
            *   从 gradleHome 目录中查找 `init.d` 脚本文件
            *   从用户自定义的 init 文件夹下找 `init.d` 文件
        *   编译 `buildSrc` 目录, 加载 `settings.gradle` 文件
            *   查找 `settings.gradle` 文件
            *   构建 `buildSrc` 的 `ClassLoader`
            *   解析 `gradle.properties`
            *   解析 `settings.gradle`
*   `configureBuild`
    *   生成 Project
        *   创建 root project 对象 `DefaultProject`
        *   保存 root project 到 gradle 中
        *   递归创建子 project 对象 `DefaultProject`
    *   配置 Project
        *   配置 Project 的流程主要是递归调用了 `Project.evaluate()`
        *   Project 的 evaluate 操作会执行当前的 Project 注入的 plugin 的 apply 方法, 会为当前的 project 注入一些默认的 task
*   `constructTaskGraph`
    *   `buildConfigurationActionExecuter.select` 收集要执行的 task
        *   `ExcludedTaskFilteringBuildConfigurationAction`: 添加任务过滤器
        *   `DefaultTasksBuildExecutionAction`: 确定要执行的任务
        *   `TaskNameResolvingBuildConfigurationAction`: 添加与执行任务有关联的任务
            *   解析 Tasks
            *   处理 task 之间的依赖添加到 TaskGraph 中
    *   `taskGraph.populate()` 构建一个有向无环图
*   `runTasks`
    *   `DryRunBuildExecutionAction`
        *   拦截处理了 `-m` 或者 `--dry-run` 参数, 如果有这个参数便会拦截执行, 直接打印 task 的 path
    *   `SelectedTaskExecutionAction` 最终会调用到 `ExecuteActionsTaskExecuter.execute` 中
        *   回调 `TaskActionListener.beforeActions` 通知外界任务即将开始执行
        *   执行 `task` 的 `actions`
        *   回调 `TaskActionListener.afterActions` 通知外界任务执行完毕
*   `finishBuild`: 通知外界构建完毕

# 参考

*   [效能优化 gradle 笔记](https://www.jianshu.com/p/d1099b77a753)
*   [gradle 源码分析](https://www.jianshu.com/p/625bc82003d7)
