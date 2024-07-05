+++
title = 'Play framework源码解析 Part1: Play framework 介绍、项目构成及启动脚本解析'
date = 2018-01-03 13:56:21
tags = ["program","backend"]
categories = ["program"] 
+++

<font color=#DC143C >注：本系列文章所用play版本为1.2.6</font>
***

# 介绍

Play framework是个轻量级的RESTful框架，致力于让java程序员实现快速高效开发，它具有以下几个方面的优势：

1. 热加载。在调试模式下，所有修改会及时生效。
1. 抛弃xml配置文件。约定大于配置。
1. 支持异步编程
1. 无状态mvc框架，拓展性良好
1. 简单的路由设置

这里附上Play framework的文档地址，官方有更为详尽的功能叙述。[Play framework文档](https://www.playframework.com/documentation/1.2.x/overview "文档")
***
<!-- more -->

# 项目构成

play framework的初始化非常简单，只要下载了play的软件包后，在命令行中运行play new xxx即可初始化一个项目。
自动生成的项目结构如下：

![play项目结构](https://i.loli.net/2018/01/08/5a53879bec91f.png)

运行play程序也非常简单，在项目目录下使用
```bash
play run
```
即可运行。

***

# 启动脚本解析

## play framework软件包目录

为了更好的了解play framework的运作原理，我们来从play framework的启动脚本开始分析，分析启动脚本有助于我们了解play framework的运行过程。

play的启动脚本是使用python编写的，脚本的入口为play软件包根目录下的play文件,下面我们将从play这个脚本的主入口开始分析。

![play项目结构](https://i.loli.net/2018/01/08/5a5386dfdfdb6.png)


## play脚本解析

play脚本在开头引入了3个类，分别为play.cmdloader,play.application,play.utils，从添加的系统参数中可以看出play启动脚本的存放路径为 framework/pym
cmdloader.py的主要作用为加载framework/pym/commands下的各个脚本文件，用于之后对命令参数的解释运行
application.py的主要作用为解析项目路径下conf/中的配置文件、加载模块、拼接最后运行的java命令

```python
sys.path.append(os.path.join(os.path.dirname(os.path.realpath(sys.argv[0])), 'framework', 'pym'))

from play.cmdloader import CommandLoader
from play.application import PlayApplication
from play.utils import *
```

在脚本的开头，有这么一段代码，意味着只要在play主程序根目录下创建一个名为id的文件，即可设置默认的框架id
```python
play_env["id_file"] = os.path.join(play_env['basedir'], 'id')
if os.path.exists(play_env["id_file"]):
    play_env["id"] = open(play_env["id_file"]).readline().strip()
else:
    play_env["id"] = ''
```

命令参数的分隔由以下代码完成，例如使用了play run --%test,那么参数列表就是
>["xxxx\play","run","--%test"]

这段代码也说明了，play的命令格式为 
>play cmd [app_path] [--options]
```python
application_path = None
remaining_args = []
if len(sys.argv) == 1:
    application_path = os.getcwd()
if len(sys.argv) == 2:
    application_path = os.getcwd()
    remaining_args = sys.argv[2:]
if len(sys.argv) > 2:
    if sys.argv[2].startswith('-'):
        application_path = os.getcwd()
        remaining_args = sys.argv[2:]
    else:
        application_path = os.path.normpath(os.path.abspath(sys.argv[2]))
        remaining_args = sys.argv[3:]
```

在play参数中，有一个ignoreMissing，这个参数的全称其实是ignoreMissingModules，作用就是在当配置文件中配置有模块但是在play目录下并没有找到时是否忽略，如需忽略那么在启动时需要加入`--force`
```python
ignoreMissing = False
if remaining_args.count('--force') == 1:
    remaining_args.remove('--force')
    ignoreMissing = True
```

脚本通过
>play_app = PlayApplication(application_path, play_env, ignoreMissing)

和
>cmdloader = CommandLoader(play_env["basedir"])

来进行PlayApplication类和CommandLoader类的初始化。
在PlayApplication类的初始化过程中，它创建了PlayConfParser类用来解析配置文件，这也就是说play的配置文件解析是在脚本启动阶段进行的，**这也是为什么修改配置文件无法实时生效需要重启的原因**  
在PlayConfParser中有一个常量DEFAULTS储存了默认的http端口和jpda调试端口，分别为9000和8000，需要添加默认值可以修改DEFAULTS，DEFAULTS内的值只有当配置文件中找不到时才会生效。
```python
DEFAULTS = {
    'http.port': '9000',
    'jpda.port': '8000'
}
```

值得一提的是，配置文件中的http.port的优先级是小于命令参数中的http.port的
```python
#env为命令参数
if env.has_key('http.port'):
    self.entries['http.port'] = env['http.port']
```

CommandLoader类的功能很简单，就是遍历framework/pym/commands下的.py文件，依次加载然后读取他的全局变量COMMANDS和MODULE储存用于之后的命令处理和模块加载。

回到play脚本中，在PlayApplication类和CommandLoader类初始化完成之后，play进行"--deps"参数的检查，如果存在--deps，则调用play.deps.DependenciesManager类来进行依赖的检查、更新。DependenciesManager的解析将放到后话详解。
```python
if remaining_args.count('--deps') == 1:
    cmdloader.commands['dependencies'].execute(command='dependencies', app=play_app, args=['--sync'], env=play_env, cmdloader=cmdloader)
    remaining_args.remove('--deps')
```

接下来，play便正式进行启动过程，play按照以下的顺序进行加载：

1. 加载配置文件中记录的模块的命令信息
2. 加载参数中指定的模块的命令信息
3. 运行各模块中的before函数
4. 执行play_command,play_command即为run,test,war等play需要的执行的命令
5. 运行各模块中的after函数
6. 结束脚本

```python
if play_command in cmdloader.commands:
    for name in cmdloader.modules:
        module = cmdloader.modules[name]
        if 'before' in dir(module):
            module.before(command=play_command, app=play_app, args=remaining_args, env=play_env)
    status = cmdloader.commands[play_command].execute(command=play_command, app=play_app, args=remaining_args, env=play_env, cmdloader=cmdloader)
    for name in cmdloader.modules:
        module = cmdloader.modules[name]
        if 'after' in dir(module):
            module.after(command=play_command, app=play_app, args=remaining_args, env=play_env)
    sys.exit(status)
```

下面，我们来看看play常用命令的运行过程...

## Play常用运行命令解析

在本节的一开始，我决定先把play所有的可用命令先列举一下，以便读者可以选择性阅读。

| 命令名称            | 命令所在文件   | 作用                                                                                               |
| ------------------- | -------------- | -------------------------------------------------------------------------------------------------- |
| antify              | ant.py         | 初始化ant构建工具的build.xml文件                                                                   |
| run                 | base.py        | 运行程序                                                                                           |
| new                 | base.py        | 新建play应用                                                                                       |
| clean               | base.py        | 删除临时文件，即清空tmp文件夹                                                                      |
| test                | base.py        | 运行测试程序                                                                                       |
| autotest、auto-test | base.py        | 自动运行所有测试项目                                                                               |
| id                  | base.py        | 设置项目id                                                                                         |
| new,run             | base.py        | 新建play应用并启动                                                                                 |
| clean,run           | base.py        | 删除临时文件并运行                                                                                 |
| modules             | base.py        | 显示项目用到的模块，注：这里显示的模块只是在项目配置文件中引用的模块，命令参数中添加的模块不会显示 |
| check               | check.py       | 检查play更新                                                                                       |
| classpath、cp       | classpath.py   | 显示应用的classpath                                                                                |
| start               | daemon.py      | 在后台运行play程序                                                                                 |
| stop                | daemon.py      | 停止正在运行的程序                                                                                 |
| restart             | daemon.py      | 重启正在运行的程序                                                                                 |
| pid                 | daemon.py      | 显示运行中的程序的pid                                                                              |
| out                 | daemon.py      | 显示输出                                                                                           |
| dependencies、deps  | deps.py        | 运行DependenciesManager更新依赖                                                                    |
| eclipsify、ec       | eclipse.py     | 创建eclipse配置文件                                                                                |
| evolutions          | evolutions.py  | 运行play.db.Evolutions进行数据库演变检查                                                           |
| help                | help.py        | 输出所有play的可用命令                                                                             |
| idealize、idea      | intellij.py    | 生成idea配置文件                                                                                   |
| javadoc             | javadoc.py     | 生成javadoc                                                                                        |
| new-module、nm      | modulesrepo.py | 创建新模块                                                                                         |
| list-modules、lm    | modulesrepo.py | 显示play社区中的模块                                                                               |
| build-module、bm    | modulesrepo.py | 打包模块                                                                                           |
| add                 | modulesrepo.py | 将模块添加至项目                                                                                   |
| install             | modulesrepo.py | 安装模块                                                                                           |
| netbeansify         | netbeans.py    | 生成netbeans配置文件                                                                               |
| precompile          | precompile.py  | 预编译                                                                                             |
| secret              | secret.py      | 生成secret key                                                                                     |
| status              | status.py      | 显示运行中项目的状态                                                                               |
| version             | version.py     | 显示play framework的版本号                                                                         |
| war                 | war.py         | 将项目打包为war文件                                                                                |

### run与start

run应该是我们平时用的最多的命令了，run命令的作用其实很简单，就是根据命令参数拼接java参数，然后调用java来运行play.server.Server，run函数的代码如下:
```python
def run(app, args):
    #app即为play脚本中创建的PlayApplication类
    global process
    #这里检查是否存在conf/routes和conf/application.conf
    app.check()

    print "~ Ctrl+C to stop"
    print "~ "
    java_cmd = app.java_cmd(args)
    try:
        process = subprocess.Popen (java_cmd, env=os.environ)
        signal.signal(signal.SIGTERM, handle_sigterm)
        return_code = process.wait()
    signal.signal(signal.SIGINT, handle_sigint)
        if 0 != return_code:
            sys.exit(return_code)
    except OSError:
        print "Could not execute the java executable, please make sure the JAVA_HOME environment variable is set properly (the java executable should reside at JAVA_HOME/bin/java). "
        sys.exit(-1)
    print
```

app.java_cmd(args)的实现代码如下：
```python
def java_cmd(self, java_args, cp_args=None, className='play.server.Server', args = None):
    if args is None:
        args = ['']
    memory_in_args=False

    #检查java参数中是否有jvm内存设置
    for arg in java_args:
        if arg.startswith('-Xm'):
            memory_in_args=True
    #如果参数中无jvm内存设置，那么在配置文件中找是否有jvm内存设置，若还是没有则在环境变量中找是否有JAVA_OPTS
    #这里其实有个问题，这里假定的是JAVA_OPTS变量里只存了jvm内存设置，如果JAVA_OPTS还存了其他选项，那对运行可能有影响
    if not memory_in_args:
        memory = self.readConf('jvm.memory')
        if memory:
            java_args = java_args + memory.split(' ')
        elif 'JAVA_OPTS' in os.environ:
            java_args = java_args + os.environ['JAVA_OPTS'].split(' ')
    #获取程序的classpath
    if cp_args is None:
        cp_args = self.cp_args()
    #读取配置文件中的jpda端口
    self.jpda_port = self.readConf('jpda.port')
    #读取配置文件中的运行模式
    application_mode = self.readConf('application.mode').lower()
    #如果模式是prod，则用server模式编译
    if application_mode == 'prod':
        java_args.append('-server')
    # JDK 7 compat
    # 使用新class校验器 (不知道作用)
    java_args.append('-XX:-UseSplitVerifier')
    #查找配置文件中是否有java安全配置，如果有则加入java参数中
    java_policy = self.readConf('java.policy')
    if java_policy != '':
        policyFile = os.path.join(self.path, 'conf', java_policy)
        if os.path.exists(policyFile):
            print "~ using policy file \"%s\"" % policyFile
            java_args.append('-Djava.security.manager')
            java_args.append('-Djava.security.policy==%s' % policyFile)
    #加入http端口设置
    if self.play_env.has_key('http.port'):
        args += ["--http.port=%s" % self.play_env['http.port']]
    #加入https端口设置
    if self.play_env.has_key('https.port'):
        args += ["--https.port=%s" % self.play_env['https.port']]

    #设置文件编码
    java_args.append('-Dfile.encoding=utf-8')
    #设置编译命令 (这边使用了jregex/Pretokenizer类的next方法，不知道有什么用)
    java_args.append('-XX:CompileCommand=exclude,jregex/Pretokenizer,next')

    #如果程序模式在dev，则添加jpda调试器参数
    if self.readConf('application.mode').lower() == 'dev':
        if not self.play_env["disable_check_jpda"]: self.check_jpda()
        java_args.append('-Xdebug')
        java_args.append('-Xrunjdwp:transport=dt_socket,address=%s,server=y,suspend=n' % self.jpda_port)
        java_args.append('-Dplay.debug=yes')
    #拼接java参数
    java_cmd = [self.java_path(), '-javaagent:%s' % self.agent_path()] + java_args + ['-classpath', cp_args, '-Dapplication.path=%s' % self.path, '-Dplay.id=%s' % self.play_env["id"], className] + args
    return java_cmd
```

start命令与run命令很类似，执行步骤为：

1. 依次查找play变量pid_file、系统环境变量PLAY_PID_PATH、项目根目录下server.pid，查找是否存在指定pid
1. 若第一步找到pid，查找当前进程列表中是否存在此pid进程，存在则试图关闭进程。(如果此时pid已经不是play的进程呢😅)
1. 在配置文件中找application.log.system.out看是否关闭了系统输出
1. 启动程序，这里与run命令唯一的区别就是他指定了stdout位置，这样就变成了后台程序
1. 将启动后的程序的pid写入server.pid

stop命令即关闭当前进程，这里要提一下，play有个注解叫OnApplicationStop，即会在程序停止时触发，而OnApplicationStop的实现主要是调用了
>Runtime.getRuntime().addShutdownHook();

来完成

### test与autotest

使用play test命令可以让程序进入测试模式，test命令和run命令其实差别不大，唯一的区别就在于使用test命令时，脚本会将play id自动替换为test，当play id为test时会自动引入testrunner模块，testrunner模块主要功能为添加@tests路由并实现了test测试页面，他的具体实现过程后续再谈。
autotest命令的作用是自动测试所有的测试用例，他的执行顺序是这样的：

1. 检查是否存在tmp文件夹，存在即删除
1. 检查是否有程序正在运行，如存在则关闭程序
1. 检查程序是否配置了ssl但是没有指定证书
1. 检查是否存在test-result文件夹，存在即删除
1. 使用test作为id重启程序
1. 调用play.modules.testrunner.FirePhoque来进行自动化测试
1. 关闭程序

autotest的脚本的步骤1和2我觉得是有问题的，应该换下顺序，不然如果程序正在向tmp文件夹插入临时文件，那么tmp文件夹就删除失败了。
关闭程序的代码调用了`http://localhost:%http_port/@kill`进行关闭，@kill的实现方法在play.CorePlugin下，注意，@kill在prod模式下无效 (~~这是废话~~)  
由于使用了python作为启动脚本，无法通过java内部变量值判断程序是否开启，只能查看控制台的输出日志，所以在使用test作为id重启后，脚本使用下面的代码判断程序是否完全启动：
```python
try:
    #这里启动程序
    play_process = subprocess.Popen(java_cmd, env=os.environ, stdout=sout)
    except OSError:
        print "Could not execute the java executable, please make sure the JAVA_HOME environment variable is set properly (the java executable should reside at JAVA_HOME/bin/java). "
        sys.exit(-1)
    #打开日志输出文件
    soutint = open(os.path.join(app.log_path(), 'system.out'), 'r')
    while True:
        if play_process.poll():
            print "~"
            print "~ Oops, application has not started?"
            print "~"
            sys.exit(-1)
        line = soutint.readline().strip()
        if line:
            print line
            #若出现'Server is up and running'则正常启动
            if line.find('Server is up and running') > -1: # This line is written out by Server.java to system.out and is not log file dependent
                soutint.close()
                break
```
FirePhoque类的实现过程我们之后再详解。

### new与clean

我们使用new来创建一个新项目，play new的使用方法为`play new project-name [--with] [--name]`。
with参数为项目所使用的模块。
name为项目名，这里要注意一点，project-name和--name的参数可以设置为不同值，project-name是项目建立的文件夹名，--name的值为项目配置文件中的application.name。  
如果不加--name，脚本会提示你是否使用project-name作为名字，在确认之后，脚本会将resources/application-skel中的所有文件拷贝到{project-name}文件夹下，然后用输入的name替换项目配置文件下的application.name，并生成一个64位的secretKey替换配置文件中的secretKey。  
接着，脚本会查找使用的模块中是否存在dependencies.yml，并将dependencies.yml中的内容加入项目的dependencies.yml中，并调用DependenciesManager检查依赖状态。

new函数的主要代码如下:
```python
print "~ The new application will be created in %s" % os.path.normpath(app.path)
if application_name is None:
    application_name = raw_input("~ What is the application name? [%s] " % os.path.basename(app.path))
if application_name == "":
    application_name = os.path.basename(app.path)
copy_directory(os.path.join(env["basedir"], 'resources/application-skel'), app.path)
os.mkdir(os.path.join(app.path, 'app/models'))
os.mkdir(os.path.join(app.path, 'lib'))
app.check()
replaceAll(os.path.join(app.path, 'conf/application.conf'), r'%APPLICATION_NAME%', application_name)
replaceAll(os.path.join(app.path, 'conf/application.conf'), r'%SECRET_KEY%', secretKey())
print "~"
```

clean命令非常简单，就是删除整个tmp文件夹

### war与precompile

很多时候，我们需要使用tomcat等服务器容器作为服务载体，这时候就需要将play应用打包为war  
war的使用参数是
>play war project-name [-o/--output][filename] [--zip] [--exclude][exclude-directories]

使用-o或--output来指定输出文件夹，使用--zip压缩为war格式，使用--exclude来包含另外需要打包的文件夹  
要注意的是，必须在项目目录外进行操作，不然会失败  

在参数处理完毕后，脚本正式开始打包过程，分为2个步骤：1.预编译。2：打包

预编译即用到了precompile命令，precompile命令与run命令几乎一样，只是在java参数中加入了`precompile=yes`  
让我们来看一下加入了这个java参数对程序的影响。
与预编译有关的代码主要是下面2段：
```java
static boolean preCompile() {
    if (usePrecompiled) {
        if (Play.getFile("precompiled").exists()) {
            classloader.getAllClasses();
            Logger.info("Application is precompiled");
            return true;
        }
        Logger.error("Precompiled classes are missing!!");
        fatalServerErrorOccurred();
        return false;
    }
    //这里开始预编译
    try {
        Logger.info("Precompiling ...");
        Thread.currentThread().setContextClassLoader(Play.classloader);
        long start = System.currentTimeMillis();
        //getAllClasses方法较长，就不贴了，下面一段代码在getAllClasses方法中进入
        classloader.getAllClasses();

        if (Logger.isTraceEnabled()) {
            Logger.trace("%sms to precompile the Java stuff", System.currentTimeMillis() - start);
        }

        if (!lazyLoadTemplates) {
            start = System.currentTimeMillis();
            //编译模板
            TemplateLoader.getAllTemplate();

            if (Logger.isTraceEnabled()) {
                Logger.trace("%sms to precompile the templates", System.currentTimeMillis() - start);
            }
        }
        return true;
    } catch (Throwable e) {
        Logger.error(e, "Cannot start in PROD mode with errors");
        fatalServerErrorOccurred();
        return false;
    }
}
```

```java
public byte[] enhance() {
    this.enhancedByteCode = this.javaByteCode;
    if (isClass()) {

        // before we can start enhancing this class we must make sure it is not a PlayPlugin.
        // PlayPlugins can be included as regular java files in a Play-application.
        // If a PlayPlugin is present in the application, it is loaded when other plugins are loaded.
        // All plugins must be loaded before we can start enhancing.
        // This is a problem when loading PlayPlugins bundled as regular app-class since it uses the same classloader
        // as the other (soon to be) enhanched play-app-classes.
        boolean shouldEnhance = true;
        try {
            CtClass ctClass = enhanceChecker_classPool.makeClass(new ByteArrayInputStream(this.enhancedByteCode));
            if (ctClass.subclassOf(ctPlayPluginClass)) {
                shouldEnhance = false;
            }
        } catch( Exception e) {
            // nop
        }

        if (shouldEnhance) {
            Play.pluginCollection.enhance(this);
        }
    }

    //主要是这一段，他将增强处理后的字节码写入了文件，增强处理在之后会深入展开
    if (System.getProperty("precompile") != null) {
        try {
            // emit bytecode to standard class layout as well
            File f = Play.getFile("precompiled/java/" + (name.replace(".", "/")) + ".class");
            f.getParentFile().mkdirs();
            FileOutputStream fos = new FileOutputStream(f);
            fos.write(this.enhancedByteCode);
            fos.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    return this.enhancedByteCode;
}
```
预编译过程结束后，脚本正式开始打包过程，打包过程比较简单，就是讲预编译后的字节码文件、模板文件、配置文件、使用的类库、使用的模块类库等移动至WEB-INF文件夹中，如果使用了--zip，那么脚本会将生成的文件夹用zip格式打包。

### secret和status

secret命令能生成一个新的secret key
status命令是用于实时显示程序的运行状态，脚本的运作十分简单，步骤如下：

1. 检查是否有--url参数，有则在他之后添加@status
1. 检查是否存在--secret
1. 如果没有--url，则使用`http://localhost:%http_port/@status`;如果没有 --secret，则从配置文件中读取secret key
1. 将secret_key、'@status'使用sha加密，并加入Authorization请求头
1. 发送请求

@status的实现和@kill一样在CorePlugin类中，这在之后再进行详解。

# 总结

Play的启动脚本分析至此就结束了，从脚本的分析过程中我们可以稍微探究下Play在脚本启动阶段有何行为，这对我们进行脚本改造或者启动优化还是非常有帮助的。
下一篇，我们来看看Play的启动类是如何运作的。。