+++
title = 'Play framework源码解析 Part3:Play的初始化与启动'
date = 2018-01-20 15:52:41
tags = ["program","backend"]
categories = ["program"] 
+++

在上一篇中，我们分析了play的2种启动方式，这一篇，我们来看看Play类的初始化过程

<!-- more -->

# Play类

无论是Server还是ServletWrapper方式运行，在他们的入口中都会运行Play.init()来对Play类进行初始化。那在解析初始化之前，我们先来看看Play类是做什么的，它里面有什么重要的方法。
首先要明确的一点是，Play类是整个Play framework框架的管理、配置中心，它存放了大部分框架需要的成员变量，例如id,配置信息,所有加载的class,使用的插件管理器等等。下图就是Play类中的方法列表。
![Play类的方法列表](https://s1.ax1x.com/2018/02/01/9FqoHU.png)
这其中加注释的几个方法是比较重要的，我们下面便来从init开始一点点剖析Play类中的各个方法。

# Play的初始化

```java
public static void init(File root, String id) {
    // Simple things
    Play.id = id;
    Play.started = false;
    Play.applicationPath = root;

    // 加载所有 play.static 中的记录的类
    initStaticStuff();
    //猜测play framework的路径
    guessFrameworkPath();

    // 读取配置文件
    readConfiguration();

    Play.classes = new ApplicationClasses();

    // 初始化日志
    Logger.init();
    String logLevel = configuration.getProperty("application.log", "INFO");

    //only override log-level if Logger was not configured manually
    if( !Logger.configuredManually) {
        Logger.setUp(logLevel);
    }
    Logger.recordCaller = Boolean.parseBoolean(configuration.getProperty("application.log.recordCaller", "false"));

    Logger.info("Starting %s", root.getAbsolutePath());
    //设置临时文件夹
    if (configuration.getProperty("play.tmp", "tmp").equals("none")) {
        tmpDir = null;
        Logger.debug("No tmp folder will be used (play.tmp is set to none)");
    } else {
        tmpDir = new File(configuration.getProperty("play.tmp", "tmp"));
        if (!tmpDir.isAbsolute()) {
            tmpDir = new File(applicationPath, tmpDir.getPath());
        }

        if (Logger.isTraceEnabled()) {
            Logger.trace("Using %s as tmp dir", Play.tmpDir);
        }

        if (!tmpDir.exists()) {
            try {
                if (readOnlyTmp) {
                    throw new Exception("ReadOnly tmp");
                }
                tmpDir.mkdirs();
            } catch (Throwable e) {
                tmpDir = null;
                Logger.warn("No tmp folder will be used (cannot create the tmp dir)");
            }
        }
    }

    // 设置运行模式
    try {
        mode = Mode.valueOf(configuration.getProperty("application.mode", "DEV").toUpperCase());
    } catch (IllegalArgumentException e) {
        Logger.error("Illegal mode '%s', use either prod or dev", configuration.getProperty("application.mode"));
        fatalServerErrorOccurred();
    }
    if (usePrecompiled || forceProd) {
        mode = Mode.PROD;
    }

    // 获取http使用路径
    ctxPath = configuration.getProperty("http.path", ctxPath);

    // 设置文件路径
    VirtualFile appRoot = VirtualFile.open(applicationPath);
    roots.add(appRoot);
    javaPath = new CopyOnWriteArrayList<VirtualFile>();
    javaPath.add(appRoot.child("app"));
    javaPath.add(appRoot.child("conf"));

    // 设置模板路径
    if (appRoot.child("app/views").exists()) {
        templatesPath = new ArrayList<VirtualFile>(2);
        templatesPath.add(appRoot.child("app/views"));
    } else {
        templatesPath = new ArrayList<VirtualFile>(1);
    }

    // 设置路由文件
    routes = appRoot.child("conf/routes");

    // 设置模块路径
    modulesRoutes = new HashMap<String, VirtualFile>(16);

    // 加载模块
    loadModules();

    // 模板路径中加入框架自带的模板文件
    templatesPath.add(VirtualFile.open(new File(frameworkPath, "framework/templates")));

    // 初始化classloader
    classloader = new ApplicationClassloader();

    // Fix ctxPath
    if ("/".equals(Play.ctxPath)) {
        Play.ctxPath = "";
    }

    // 设置cookie域名
    Http.Cookie.defaultDomain = configuration.getProperty("application.defaultCookieDomain", null);
    if (Http.Cookie.defaultDomain!=null) {
        Logger.info("Using default cookie domain: " + Http.Cookie.defaultDomain);
    }

    // 加载插件
    pluginCollection.loadPlugins();

    // 如果是prod直接启动
    if (mode == Mode.PROD || System.getProperty("precompile") != null) {
        mode = Mode.PROD;
        //预编译
        if (preCompile() && System.getProperty("precompile") == null) {
            start();
        } else {
            return;
        }
    } else {
        Logger.warn("You're running Play! in DEV mode");
    }

    pluginCollection.onApplicationReady();

    Play.initialized = true;
}
```

如上面的代码所示，初始化过程主要的顺序为：

1. 加载所有play.static中记录的类
1. 获取框架路径
1. 读取配置文件
1. 初始化日志
1. 获取java文件、模板文件路径
1. 加载模块
1. 加载插件
1. 若为prod模式进行预编译

我们来依次看看Play在这些过程中做了什么事情。

## 加载play.static

Play在初始化过程中会调用initStaticStuff()方法来检查代码目录下是否存在play.static文件，如果存在，那么就逐行读取文件中记录的类，并通过反射加载类中的静态初始化代码段。至于作用吗，不知道有什么用，这段代码的优先级太高了，早于初始化过程运行，若在初始化过程结束后运行还可以用来覆写Play类中的配置信息。~~或者自己写插件然后在play.static中初始化插件依赖？或者绑定新的数据源？~~

```java
public static void initStaticStuff() {
    // Play! plugings
    Enumeration<URL> urls = null;
    try {
        urls = Play.class.getClassLoader().getResources("play.static");
    } catch (Exception e) {
    }
    while (urls != null && urls.hasMoreElements()) {
        URL url = urls.nextElement();
        try {
            BufferedReader reader = new BufferedReader(new InputStreamReader(url.openStream(), "utf-8"));
            String line = null;
            while ((line = reader.readLine()) != null) {
                try {
                    Class.forName(line);
                } catch (Exception e) {
                    Logger.warn("! Cannot init static: " + line);
                }
            }
        } catch (Exception ex) {
            Logger.error(ex, "Cannot load %s", url);
        }
    }
}
```

## 获取框架路径

获取框架路径很简单，就是判断是用jar模式运行还是直接用文件运行，然后读取对应的路径

```java
public static void guessFrameworkPath() {
    // Guess the framework path
    try {
        URL versionUrl = Play.class.getResource("/play/version");
        // Read the content of the file
        Play.version = new LineNumberReader(new InputStreamReader(versionUrl.openStream())).readLine();

        // This is used only by the embedded server (Mina, Netty, Jetty etc)
        URI uri = new URI(versionUrl.toString().replace(" ", "%20"));
        if (frameworkPath == null || !frameworkPath.exists()) {
            if (uri.getScheme().equals("jar")) {
                String jarPath = uri.getSchemeSpecificPart().substring(5, uri.getSchemeSpecificPart().lastIndexOf("!"));
                frameworkPath = new File(jarPath).getParentFile().getParentFile().getAbsoluteFile();
            } else if (uri.getScheme().equals("file")) {
                frameworkPath = new File(uri).getParentFile().getParentFile().getParentFile().getParentFile();
            } else {
                throw new UnexpectedException("Cannot find the Play! framework - trying with uri: " + uri + " scheme " + uri.getScheme());
            }
        }
    } catch (Exception e) {
        throw new UnexpectedException("Where is the framework ?", e);
    }
}
```

## 读取配置文件

首先要说明一下，我们这里讨论的是在Play类初始化过程中的读取配置文件过程，为什么要指出这一点呢，因为配置文件读取后会调用插件的onConfigurationRead方法，而在初始化过程中，配置文件是优先于插件加载的，所以在初始化过程中插件的方法并不会生效。等到Play调用start方法启动服务器时，会重新读取配置文件，那时候插件列表已经更新完毕，会执行onConfigurationRead方法。
配置文件的读取和启动脚本中的解析方式基本一样，步骤就是下面几步：

1. 读取application.conf，将其中的项全部加入Properties
1. 将所有配置项用正则过滤找出真实配置名，并找出所用id的配置项替换
1. 将配置项中的${..}替换为对应值
1. 读取@include引用的配置

我们来看下play对${..}替换过程
```java
Pattern pattern = Pattern.compile("\\$\\{([^}]+)}");
for (Object key : propsFromFile.keySet()) {
    String value = propsFromFile.getProperty(key.toString());
    Matcher matcher = pattern.matcher(value);
    StringBuffer newValue = new StringBuffer(100);
    while (matcher.find()) {
        String jp = matcher.group(1);
        String r;
        //重点是下面这个判断
        if (jp.equals("application.path")) {
            r = Play.applicationPath.getAbsolutePath();
        } else if (jp.equals("play.path")) {
            r = Play.frameworkPath.getAbsolutePath();
        } else {
            r = System.getProperty(jp);
            if (r == null) {
                r = System.getenv(jp);
            }
            if (r == null) {
                Logger.warn("Cannot replace %s in configuration (%s=%s)", jp, key, value);
                continue;
            }
        }
        matcher.appendReplacement(newValue, r.replaceAll("\\\\", "\\\\\\\\"));
    }
    matcher.appendTail(newValue);
    propsFromFile.setProperty(key.toString(), newValue.toString());
}
```

可以看出除了${application.path}和${play.path}，我们还可以通过使用参数和修改环境变量来添加可替换的值

## 初始化日志

Play的日志处理放在Logger类中，默认使用log4j作为日志记录工具，初始化过程的顺序如下

1. 查找配置文件中是否有日志配置文件路径，存在跳至4
1. 查找是否有log4j.xml，存在跳至4
1. 查找是否有log4j.properties，存在跳至4
1. 根据文件后缀使用对应的日志配置解析器
1. 如果是测试模式则加上写入测试结果文件的Appender

```java
public static void init() {
    //查找日志配置文件路径，没有则用log4j.xml
    String log4jPath = Play.configuration.getProperty("application.log.path", "/log4j.xml");
    URL log4jConf = Logger.class.getResource(log4jPath);
    boolean isXMLConfig = log4jPath.endsWith(".xml");
    //日志配置不存在，查找log4j.properties
    if (log4jConf == null) { // try again with the .properties
        isXMLConfig = false;
        log4jPath = Play.configuration.getProperty("application.log.path", "/log4j.properties");
        log4jConf = Logger.class.getResource(log4jPath);
    }
    //找不到配置文件就关闭日志
    if (log4jConf == null) {
        Properties shutUp = new Properties();
        shutUp.setProperty("log4j.rootLogger", "OFF");
        PropertyConfigurator.configure(shutUp);
    } else if (Logger.log4j == null) {
        //判断日志配置文件是否在应用目录下，加这条是因为play软件包目录下有默认的日志配置文件
        if (log4jConf.getFile().indexOf(Play.applicationPath.getAbsolutePath()) == 0) {
            // The log4j configuration file is located somewhere in the application folder,
            // so it's probably a custom configuration file
            configuredManually = true;
        }
        //根据不同的类型解析
        if (isXMLConfig) {
            DOMConfigurator.configure(log4jConf);
        } else {
            PropertyConfigurator.configure(log4jConf);
        }
        Logger.log4j = org.apache.log4j.Logger.getLogger("play");
        // 测试模式下，将日志追加到test-result/application.log
        if (Play.runingInTestMode()) {
            org.apache.log4j.Logger rootLogger = org.apache.log4j.Logger.getRootLogger();
            try {
                if (!Play.getFile("test-result").exists()) {
                    Play.getFile("test-result").mkdir();
                }
                Appender testLog = new FileAppender(new PatternLayout("%d{DATE} %-5p ~ %m%n"), Play.getFile("test-result/application.log").getAbsolutePath(), false);
                rootLogger.addAppender(testLog);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
}
```

初始化的代码可以看出log4j.xml的优先级高于log4j.properties，这里可以发现一个问题，Play的日志初始化完全是针对log4j来进行的，但是翻看Logger类的代码可以看到，所有的日志输出方法都会先判断是否使用java.util.logging.Logger来输出日志，在Logger类中也有一个forceJuli字段来判断是否使用，但是这个字段一直没有使用过，也就是说只要不手动改，那forceJuli一直是false。
如果没有做任何配置，那么直接使用Logger来输出日志，那么显示的类名就只会是Play，需要在配置文件中加入application.log.recordCaller=true来将日志输出时显示的类变为调用类名

## 查找文件路径

Play类中记录的路径有5种，分别为根目录路径，java文件路径，模板文件路径，主路由路径，模块路由路径。文件路径的添加很简单，就是将文件路径记录在对应的变量中，这里要提的一点是，conf文件夹是被加入到java文件路径中的，所以写在conf文件夹中的java源码也是可以使用的。~~当然，在conf文件夹里写源码大概会被骂死~~

## 加载模块

Play通过调用loadModules()来加载所有模块，使用的模块在3个地方查找，1是系统环境变量，环境变量MODULES记录的模块将加载，2是配置文件中记录的模块，配置信息为module.开头的模块被加载，3是应用目录下的modules文件夹下的模块被加载。在查找完毕后，会判断是否使用测试模式，是则加入_testrunner模块，并判断是否为dev模式，是则会加入\_docviewer模块。
这里我们来看下Play将模块文件加入应用是如何做的
```java
public static void addModule(String name, File path) {
    VirtualFile root = VirtualFile.open(path);
    modules.put(name, root);
    if (root.child("app").exists()) {
        javaPath.add(root.child("app"));
    }
    if (root.child("app/views").exists()) {
        templatesPath.add(root.child("app/views"));
    }
    if (root.child("conf/routes").exists()) {
        modulesRoutes.put(name, root.child("conf/routes"));
    }
    roots.add(root);
    if (!name.startsWith("_")) {
        Logger.info("Module %s is available (%s)", name, path.getAbsolutePath());
    }
}
```

可以看出引入模块其实就是在各个路径下加入模块路径

## 加载插件

插件(plugins)是Play framework框架非常重要的组成部分，插件作用于Play运行时的方方面面，包括请求前后的处理，更新字节码等等，具体的插件使用说明我们放在之后的插件篇再说，这里就说说加载插件的过程。
插件的加载过程如下：

1. 查找java文件路径下存在的play.plugins文件
1. 读取每一个play.plugins，play.plugins中的记录由2部分组成，一个是插件优先级，一个是类名，逐行读取后根据优先级和类目组成LoadingPluginInfo，并加入List<LoadingPluginInfo>
1. 根据优先级对List<LoadingPluginInfo>排序
1. 根据排序完后的列表依次将插件加入至所有插件列表
1. 初始化插件
1. 更新插件列表

我们先来看看将插件加入所有插件列表的过程
```java
protected boolean addPlugin( PlayPlugin plugin ){
    synchronized( lock ){
        //判断插件列表是否存在插件
        if( !allPlugins.contains(plugin) ){
            allPlugins.add( plugin );
            //根据优先级排序
            Collections.sort(allPlugins);
            //创建只读的插件列表
            allPlugins_readOnlyCopy = createReadonlyCopy( allPlugins);
            //启用插件
            enablePlugin(plugin);
            return true;
        }
    }
    return false;
}
```

这是启用插件的方法
```java
public boolean enablePlugin( PlayPlugin plugin ){
    synchronized( lock ){
        //检查是否存在插件
        if( allPlugins.contains( plugin )){
            //检查插件是否已在启动列表
            if( !enabledPlugins.contains( plugin )){
                //加入启动插件
                enabledPlugins.add( plugin );
                //排序
                Collections.sort( enabledPlugins);
                //创建只读列表
                enabledPlugins_readOnlyCopy = createReadonlyCopy( enabledPlugins);
                //更新插件列表
                updatePlayPluginsList();
                Logger.trace("Plugin " + plugin + " enabled");
                return true;
            }
        }
    }
    return false;
}
```

这里有一个问题是，既然加入插件列表时会进行排序，那对List<LoadingPluginInfo>的排序是否就不需要了呢，其实不是的，因为在加入插件列表时会反射生成PlayPlugin的一个实例，如果实例的静态代码段对插件进行了修改，就会出现问题。
在插件加入完毕后，会循环列表进行插件的初始化

```java
for( PlayPlugin plugin : getEnabledPlugins()){
    if( isEnabled(plugin)){
        initializePlugin(plugin);
    }
}
```

这里为什么要在循环里再判断一遍插件是否启用呢，是为了让高优先级插件可以禁用低优先级插件。

## 预编译

预编译是为了在prod模式下加快加载速度，预编译方法的作用是将已经预编译好的文件读入或对文件进行预编译，对文件预编译包括了java文件的预编译以及模板文件的预编译，我们分别来看看

### java预编译

在细说java预编译过程之前，我觉得有必要先来说一下play framework的类加载机制。  
play.classloading.ApplicationClassloader是play框架的类加载器，所有play框架的代码均由ApplicationClassloader来加载。使用自建的类加载器主要是为了便于处理预编译后的字节码以及方便在dev模式下进行即时的热更新。  
play.classloading.ApplicationClasses类是应用代码的容器，里面存放了所有的class。  
ApplicationClassloader调用通过查找ApplicationClasses来获取对应的类代码（具体来说没有这么简单，因为这边只谈预编译，所以具体的流程暂且不谈，在之后的classloader篇再细说）  
java预编译的入口在ApplicationClassloader.getAllClasses();它的过程如下：  

1. 检查插件是否有编译源码的方法，若有跳至4
1. 扫描之前设定的javaPath下存在的java文件，读取类名，创建ApplicationClass，ApplicationClass类内有类名、java文件、java代码、编译后字节码、增强后字节码等字段，ApplicationClasses中存放的就是一个个ApplicationClass
1. 使用eclipse JDT对源码进行编译，编译后的字节码存到对应的ApplicationClass中
1. 遍历ApplicationClasses中的所有ApplicationClass，判断其是否需要增强，需要增强的类用插件进行增强并写入文件

play的编译器使用的是play.classloading.ApplicationCompiler类，这里对编译过程就不做更多的阐述。
这里有几点值得提一下

1. 上面步骤2中扫描java文件只根据java文件名来判断类名，也就是说在步骤2的时候，ApplicationClasses容器中存放的class还只是单纯的外部类，而内部类与匿名内部类还不包括在内，在步骤3使用jdt编译时，ApplicationCompiler类会将匿名类与内部类都加到ApplicationClasses容器中，所以在第4步遍历的时候也会将内部类与匿名类的字节码文件增强然后加到文件中
1. 由于步骤1会检查是否有插件会编译源码，也就意味着我们可以通过自写插件来覆盖play原有的编译过程。这点我觉得非常不错，因为在使用play做为工程框架时，等到代码量很大时，每次编译过程都会非常漫长，因为play默认会将所有java文件全部重新编译一遍，而这有时候是很不必要的，因为其实只要重新编译更改的文件即可，针对这一点我写了一个根据文件更改时间选择编译的插件，这个放到之后的play应用中再谈。

下面就是java预编译的主要代码
```java
//判断是否有插件会进行编译
if(!Play.pluginCollection.compileSources()) {

    List<ApplicationClass> all = new ArrayList<ApplicationClass>();
    //在javaPath中找所有类
    for (VirtualFile virtualFile : Play.javaPath) {
        all.addAll(getAllClasses(virtualFile));
    }
    List<String> classNames = new ArrayList<String>();
    //将所有类名组成list
    for (int i = 0; i < all.size(); i++) {
            ApplicationClass applicationClass = all.get(i);
        if (applicationClass != null && !applicationClass.compiled && applicationClass.isClass()) {
            classNames.add(all.get(i).name);
        }
    }
    //调用编译器编译
    Play.classes.compiler.compile(classNames.toArray(new String[classNames.size()]));
}

//遍历所有类，添加至allClasses，即ApplicationClasses容器中
for (ApplicationClass applicationClass : Play.classes.all()) {
    //loadApplicationClass方法关键代码如下
    Class clazz = loadApplicationClass(applicationClass.name);
    if (clazz != null) {
        allClasses.add(clazz);
    }
}
```

```java
long start = System.currentTimeMillis();
//从ApplicationClasses容器中找对应的ApplicationClass
ApplicationClass applicationClass = Play.classes.getApplicationClass(name);
if (applicationClass != null) {
    //isDefinable方法就是判断applicationClass是否已经编译且存在对应的javaClass
    if (applicationClass.isDefinable()) {
        return applicationClass.javaClass;
    }
    //查找之前是否存在编译结果
    byte[] bc = BytecodeCache.getBytecode(name, applicationClass.javaSource);

    if (Logger.isTraceEnabled()) {
        Logger.trace("Compiling code for %s", name);
    }
    //判断applicationClass是一个类还是package-info
    if (!applicationClass.isClass()) {
        definePackage(applicationClass.getPackage(), null, null, null, null, null, null, null);
    } else {
        loadPackage(name);
    }
    //如果之前的编译结果存在，就用之前的
    if (bc != null) {
        applicationClass.enhancedByteCode = bc;
        applicationClass.javaClass = defineClass(applicationClass.name, applicationClass.enhancedByteCode, 0, applicationClass.enhancedByteCode.length, protectionDomain);
        resolveClass(applicationClass.javaClass);
        if (!applicationClass.isClass()) {
            applicationClass.javaPackage = applicationClass.javaClass.getPackage();
        }

        if (Logger.isTraceEnabled()) {
            Logger.trace("%sms to load class %s from cache", System.currentTimeMillis() - start, name);
        }

        return applicationClass.javaClass;
    }
    //如果applicationClass编译过了或者编译后有字节码，进行字节码增强
    if (applicationClass.javaByteCode != null || applicationClass.compile() != null) {
        //进行字节码增强
        applicationClass.enhance();
        applicationClass.javaClass = defineClass(applicationClass.name, applicationClass.enhancedByteCode, 0, applicationClass.enhancedByteCode.length, protectionDomain);
        BytecodeCache.cacheBytecode(applicationClass.enhancedByteCode, name, applicationClass.javaSource);
        resolveClass(applicationClass.javaClass);
        if (!applicationClass.isClass()) {
            applicationClass.javaPackage = applicationClass.javaClass.getPackage();
        }

        if (Logger.isTraceEnabled()) {
            Logger.trace("%sms to load class %s", System.currentTimeMillis() - start, name);
        }

        return applicationClass.javaClass;
    }
    Play.classes.classes.remove(name);
}
```

### 模板预编译

不同于java的预编译，模板的预编译结果虽然也会存放在precompile文件夹，但是在运行过程中模板并不是一次性全部加载的，模板的加载主要通过play.templates.TemplateLoader来进行，TemplateLoader中存放了使用过的模板信息。
在模板预编译过程中，主要是步骤如下：

1. 扫描Play.templatesPath目录下所有模板文件
1. 遍历文件，用TemplateLoader.load来加载源文件，并进行模板编译，产生Groovy源码
1. 对编译后的Groovy源码进行编译，并用base64加密写入文件

这里只对模板预编译流程做一个梳理，至于编译的过程放在之后的模板篇再说

# play framework的启动

从上一篇server与servletWrapper中我们可以发现，play framework并不是一定在脚本启动之后便启动服务器，在我们使用dev模式进行开发时也会发现，play总是需要接受到一个请求后才会有真正的启动流程。我们这一节就来看看play的启动过程是怎么样的，这个启动与server或servletWrapper的启动的区别在于，play启动后便真正开始业务处理，而server与servletWrapper的启动仅仅是启动了监听端口，当然要清楚的是servletWrapper启动时也会自动启动play
```java
public static synchronized void start() {
    try {
        //如果已经启动了，先停止，这里是为了dev模式的热更新
        if (started) {
            stop();
        }
        //如果不是独立的server，即如果不是放在servlet容器中运行，注册关闭事件
        if( standalonePlayServer) {
            // Can only register shutdown-hook if running as standalone server
            if (!shutdownHookEnabled) {
                //registers shutdown hook - Now there's a good chance that we can notify
                //our plugins that we're going down when some calls ctrl+c or just kills our process..
                shutdownHookEnabled = true;
                Runtime.getRuntime().addShutdownHook(new Thread() {
                    public void run() {
                        Play.stop();
                    }
                });
            }
        }
        //如果是dev模式启动，重新加载所有class和插件
        if (mode == Mode.DEV) {
            // Need a new classloader
            classloader = new ApplicationClassloader();
            // Put it in the current context for any code that relies on having it there
            Thread.currentThread().setContextClassLoader(classloader);
            // Reload plugins
            pluginCollection.reloadApplicationPlugins();

        }

        // 读取配置文件
        readConfiguration();

        // 配置日志
        String logLevel = configuration.getProperty("application.log", "INFO");
        //only override log-level if Logger was not configured manually
        if( !Logger.configuredManually) {
            Logger.setUp(logLevel);
        }
        Logger.recordCaller = Boolean.parseBoolean(configuration.getProperty("application.log.recordCaller", "false"));

        // 设置语言
        langs = new ArrayList<String>(Arrays.asList(configuration.getProperty("application.langs", "").split(",")));
        if (langs.size() == 1 && langs.get(0).trim().length() == 0) {
            langs = new ArrayList<String>(16);
        }

        // 重新加载模板
        TemplateLoader.cleanCompiledCache();

        // 设置secretKey
        secretKey = configuration.getProperty("application.secret", "").trim();
        if (secretKey.length() == 0) {
            Logger.warn("No secret key defined. Sessions will not be encrypted");
        }

        // 设置默认web encoding
        String _defaultWebEncoding = configuration.getProperty("application.web_encoding");
        if( _defaultWebEncoding != null ) {
            Logger.info("Using custom default web encoding: " + _defaultWebEncoding);
            defaultWebEncoding = _defaultWebEncoding;
            // Must update current response also, since the request/response triggering
            // this configuration-loading in dev-mode have already been
            // set up with the previous encoding
            if( Http.Response.current() != null ) {
                Http.Response.current().encoding = _defaultWebEncoding;
            }
        }

        // 加载所有class
        Play.classloader.getAllClasses();

        // 加载路由
        Router.detectChanges(ctxPath);

        // 初始化缓存
        Cache.init();

        // 运行插件onApplicationStart方法
        try {
            pluginCollection.onApplicationStart();
        } catch (Exception e) {
            if (Play.mode.isProd()) {
                Logger.error(e, "Can't start in PROD mode with errors");
            }
            if (e instanceof RuntimeException) {
                throw (RuntimeException) e;
            }
            throw new UnexpectedException(e);
        }

        if (firstStart) {
            Logger.info("Application '%s' is now started !", configuration.getProperty("application.name", ""));
            firstStart = false;
        }

        // We made it
        started = true;
        startedAt = System.currentTimeMillis();

        // 运行插件afterApplicationStart方法
        pluginCollection.afterApplicationStart();

    } catch (PlayException e) {
        started = false;
        try { Cache.stop(); } catch (Exception ignored) {}
        throw e;
    } catch (Exception e) {
        started = false;
        try { Cache.stop(); } catch (Exception ignored) {}
        throw new UnexpectedException(e);
    }
}
```

可以看出play的启动和play的初始化有很多相同的地方，包括加载配置，加载日志等，启动过程有很多有意思的地方

1. play的启动不会重置模块列表，也就是说如果在dev模式下对配置文件进行修改增加了模块必须要重启服务器，热更新是无效的。
1. 在play的初始化过程中，如果是使用容器打开服务器时就会加载预编译文件，这时候已经读取一遍预编译源码了，所有在启动过程中的getAllClasses就不会再去查找了
1. 插件的onApplicationStart方法一般就是执行些插件初始化的工作，所以遇到错误就会中断启动过程，而afterApplicationStart是在启动完成后在运行的，很有意思的是，用@OnApplicationStart标识的job类是在afterApplicationStart阶段运行的，而不是onApplicationStart阶段。~~那为什么要叫这个名字~~

可以看出路由的解析是在play的启动过程中进行的，具体过程就是读取路径下的路由文件，然后路由的具体解析过程放在模板篇一起讲吧

# 总结

Play类的初始化与启动已经说的差不多了，下一篇我们来看下ActionInvoker与mvc