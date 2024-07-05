+++
title = 'Play framework源码解析 Part2:Server与ServletWrapper'
date = 2018-01-11 15:16:54
tags = ["program","backend"]
categories = ["program"] 
+++

在上一节中我们剖析了Play framework的启动原理，很容易就能发现Play framework的启动主入口在play.server.Server中，在本节，我们来一起看看Server类中主要发生了什么。

# Server类

既然是程序运行的主入口，那么必然是由main方法进入的，Server类中的main方法十分简单。源码如下:

<!-- more -->

```java
public static void main(String[] args) throws Exception {
    File root = new File(System.getProperty("application.path"));
    //获取参数中的precompiled
    if (System.getProperty("precompiled", "false").equals("true")) {
        Play.usePrecompiled = true;
    }
    //获取参数中的writepid
    if (System.getProperty("writepid", "false").equals("true")) {
        //这个方法的作用是检查当前目录下是否存在server.pid文件，若存在表明当前已有程序在运行
        writePID(root);
    }
    //Play类的初始化
    Play.init(root, System.getProperty("play.id", ""));
    if (System.getProperty("precompile") == null) {
        //Server类初始化
        new Server(args);
    } else {
        Logger.info("Done.");
    }
}
```

main方法执行的操作很简单:

1. 获取程序路径
1. 检查是否存在precompiled参数
1. 检查是否存在writepid参数，若存在则检查是否存在server.pid文件，若存在则表明已有程序在运行，不存在则将当前程序pid写入server.pid
1. play类初始化
1. 检查是否存在precompile参数项，若存在表示是个预编译行为，结束运行，若没有则启动服务

这其中最重要的便是Play类的初始化以及Server类的初始化
这里我们先来看Server类的初始化过程，现在可以先简单的将Play类的初始化理解为Play框架中一些常量的初始化以及日志、配置文件、路由信息等配置的读取。
这里贴一下Server类的初始化过程：
```java
public Server(String[] args) {
    //设置文件编码为UTF-8
    System.setProperty("file.encoding", "utf-8");
    //p为Play类初始化过程中读取的配置文件信息
    final Properties p = Play.configuration;
    //获取参数中的http与https端口信息，若不存在则用配置文件中的http与https端口信息
    httpPort = Integer.parseInt(getOpt(args, "http.port", p.getProperty("http.port", "-1")));
    httpsPort = Integer.parseInt(getOpt(args, "https.port", p.getProperty("https.port", "-1")));
    //若没有配置则设置默认端口为9000
    if (httpPort == -1 && httpsPort == -1) {
        httpPort = 9000;
    }
    //http与https端口不能相同
    if (httpPort == httpsPort) {
        Logger.error("Could not bind on https and http on the same port " + httpPort);
        Play.fatalServerErrorOccurred();
    }

    InetAddress address = null;
    InetAddress secureAddress = null;
    try {
        //获取配置文件中的默认http地址，若不存在则在系统参数中查找
        //之前还是参数配置大于配置文件，这里不知道为什么又变成了配置文件的优先级高于参数配置，很迷
        if (p.getProperty("http.address") != null) {
            address = InetAddress.getByName(p.getProperty("http.address"));
        } else if (System.getProperties().containsKey("http.address")) {
            address = InetAddress.getByName(System.getProperty("http.address"));
        }

    } catch (Exception e) {
        Logger.error(e, "Could not understand http.address");
        Play.fatalServerErrorOccurred();
    }
    try {
        //同上，获取https地址
        if (p.getProperty("https.address") != null) {
            secureAddress = InetAddress.getByName(p.getProperty("https.address"));
        } else if (System.getProperties().containsKey("https.address")) {
            secureAddress = InetAddress.getByName(System.getProperty("https.address"));
        }
    } catch (Exception e) {
        Logger.error(e, "Could not understand https.address");
        Play.fatalServerErrorOccurred();
    }
    //netty服务器启动类初始化，使用nio服务器，无限制线程池
    //这里的线程池是netty的主线程池与工作线程池，是处理连接的线程池，而Play实际执行业务操作的线程池在另一个地方配置
    ServerBootstrap bootstrap = new ServerBootstrap(new NioServerSocketChannelFactory(
            Executors.newCachedThreadPool(), Executors.newCachedThreadPool())
    );
    try {
        //初始化http端口
        if (httpPort != -1) {
            //设置管道工厂类
            bootstrap.setPipelineFactory(new HttpServerPipelineFactory());
            //绑定端口
            bootstrap.bind(new InetSocketAddress(address, httpPort));
            bootstrap.setOption("child.tcpNoDelay", true);

            if (Play.mode == Mode.DEV) {
                if (address == null) {
                    Logger.info("Listening for HTTP on port %s (Waiting a first request to start) ...", httpPort);
                } else {
                    Logger.info("Listening for HTTP at %2$s:%1$s (Waiting a first request to start) ...", httpPort, address);
                }
            } else {
                if (address == null) {
                    Logger.info("Listening for HTTP on port %s ...", httpPort);
                } else {
                    Logger.info("Listening for HTTP at %2$s:%1$s  ...", httpPort, address);
                }
            }

        }

    } catch (ChannelException e) {
        Logger.error("Could not bind on port " + httpPort, e);
        Play.fatalServerErrorOccurred();
    }
    //下面是https端口服务器的启动过程，和http一致
    bootstrap = new ServerBootstrap(new NioServerSocketChannelFactory(
            Executors.newCachedThreadPool(), Executors.newCachedThreadPool())
    );

    try {
        if (httpsPort != -1) {
            //这里的管道工厂类变成了SslHttpServerPipelineFactory
            bootstrap.setPipelineFactory(new SslHttpServerPipelineFactory());
            bootstrap.bind(new InetSocketAddress(secureAddress, httpsPort));
            bootstrap.setOption("child.tcpNoDelay", true);

            if (Play.mode == Mode.DEV) {
                if (secureAddress == null) {
                    Logger.info("Listening for HTTPS on port %s (Waiting a first request to start) ...", httpsPort);
                } else {
                    Logger.info("Listening for HTTPS at %2$s:%1$s (Waiting a first request to start) ...", httpsPort, secureAddress);
                }
            } else {
                if (secureAddress == null) {
                    Logger.info("Listening for HTTPS on port %s ...", httpsPort);
                } else {
                    Logger.info("Listening for HTTPS at %2$s:%1$s  ...", httpsPort, secureAddress);
                }
            }

        }

    } catch (ChannelException e) {
        Logger.error("Could not bind on port " + httpsPort, e);
        Play.fatalServerErrorOccurred();
    }
    if (Play.mode == Mode.DEV) {
        // print this line to STDOUT - not using logger, so auto test runner will not block if logger is misconfigured (see #1222)
        //输出启动成功，以便进行自动化测试
        System.out.println("~ Server is up and running");
    }
}
```

server类的初始化没什么好说的，重点就在于那2个管道工厂类，HttpServerPipelineFactory与SslHttpServerPipelineFactory

## HttpServerPipelineFactory

HttpServerPipelineFactory类作用就是为netty服务器增加了各个ChannelHandler。
```java
public class HttpServerPipelineFactory implements ChannelPipelineFactory {

    public ChannelPipeline getPipeline() throws Exception {

        Integer max = Integer.valueOf(Play.configuration.getProperty("play.netty.maxContentLength", "-1"));

        ChannelPipeline pipeline = pipeline();
        PlayHandler playHandler = new PlayHandler();

        pipeline.addLast("flashPolicy", new FlashPolicyHandler()); 
        pipeline.addLast("decoder", new HttpRequestDecoder());
        pipeline.addLast("aggregator", new StreamChunkAggregator(max));
        pipeline.addLast("encoder", new HttpResponseEncoder());
        pipeline.addLast("chunkedWriter", playHandler.chunkedWriteHandler);
        pipeline.addLast("handler", playHandler);

        return pipeline;
    }
}
```

pipeline依次添加了flash处理器，http request解码器，流区块聚合器，http response编码器，区块写入器，play处理器  
flash处理器和流区块聚合器这里暂且不作详细解析，读者可以理解为这2者的作用分别为传输flash流和合并请求头中有Transfer-Encoding:chunked的请求数据。  
PlayHandler类是Play将netty接收到的request转换为play的request并真正开始业务处理的入口类，我们先暂时放下PlayHandler，先来看看SslHttpServerPipelineFactory做了什么

## SslHttpServerPipelineFactory

SslHttpServerPipelineFactory是https使用的管道工厂类，他除了添加了一下处理器外，最重要的是根据配置创建了https的连接方式。
```java
    public ChannelPipeline getPipeline() throws Exception {

        Integer max = Integer.valueOf(Play.configuration.getProperty("play.netty.maxContentLength", "-1"));
        String mode = Play.configuration.getProperty("play.netty.clientAuth", "none");

        ChannelPipeline pipeline = pipeline();

        // Add SSL handler first to encrypt and decrypt everything.
        SSLEngine engine = SslHttpServerContextFactory.getServerContext().createSSLEngine();
        engine.setUseClientMode(false);

        if ("want".equalsIgnoreCase(mode)) {
            engine.setWantClientAuth(true);
        } else if ("need".equalsIgnoreCase(mode)) {
            engine.setNeedClientAuth(true);
        }

        engine.setEnableSessionCreation(true);

        pipeline.addLast("flashPolicy", new FlashPolicyHandler());
        pipeline.addLast("ssl", new SslHandler(engine));
        pipeline.addLast("decoder", new HttpRequestDecoder());
        pipeline.addLast("aggregator", new StreamChunkAggregator(max));
        pipeline.addLast("encoder", new HttpResponseEncoder());
        pipeline.addLast("chunkedWriter", new ChunkedWriteHandler());

        pipeline.addLast("handler", new SslPlayHandler());

        return pipeline;
    }
```

在设置完SSLEngine后，SslHttpServerPipelineFactory在decoder处理器前加了一个SslHandler来处理连接，这里不是使用playHandler作为处理器，而是使用SslPlayHandler，SslPlayHandler是PlayHandler的一个子类，就是重写了连接和出错时的处理，这里不多做阐述。

## PlayHandler

PlayHandler的作用是解析http request的类型，然后根据不同类型来进行不同的处理，由于PlayHandler内处理的东西较多，这里先附上一张大体流程图以供读者参考

![PlayHandle流程图](https://s1.ax1x.com/2018/01/16/pdfssg.jpg)

### messageReceived

messageReceived是一个Override方法，作用就是在netty消息传递至该handle时进行相应操作,方法代码如下
```java
@Override
public void messageReceived(final ChannelHandlerContext ctx, final MessageEvent messageEvent) throws Exception {
    if (Logger.isTraceEnabled()) {
        Logger.trace("messageReceived: begin");
    }

    final Object msg = messageEvent.getMessage();

    //判断是否为HttpRequest的信息，不是则不进行处理
    if (msg instanceof HttpRequest) {

        final HttpRequest nettyRequest = (HttpRequest) msg;

        // 若为websocket的初次握手请求，则进行websocket握手过程
        if (HttpHeaders.Values.WEBSOCKET.equalsIgnoreCase(nettyRequest.getHeader(HttpHeaders.Names.UPGRADE))) {
            websocketHandshake(ctx, nettyRequest, messageEvent);
            return;
        }

        try {
            //将netty的request转为Play的request
            final Request request = parseRequest(ctx, nettyRequest, messageEvent);

            final Response response = new Response();
            //Response.current为ThreadLocal<Response>类型，保证每个线程拥有单独Response
            Http.Response.current.set(response);

            // Buffered in memory output
            response.out = new ByteArrayOutputStream();

            // Direct output (will be set later)
            response.direct = null;

            // Streamed output (using response.writeChunk)
            response.onWriteChunk(new Action<Object>() {

                public void invoke(Object result) {
                    writeChunk(request, response, ctx, nettyRequest, result);
                }
            });

            /*
                查找Play插件(即继承了PlayPlugin抽象类的类)中是否有该request的实现方法，若存在，则用Play插件的实现
                Play自带的插件实现有：
                    CorePlugin实现了/@kill和/@status路径的访问流程
                    DBPlugin实现了/@db路径的访问流程
                    Evolutions实现了/@evolutions路径的访问流程
                这里将插件的访问与其他访问分开最主要原因应该是插件访问不需要启动Play，而其他命令必须先调用Play.start来启动
            */
            boolean raw = Play.pluginCollection.rawInvocation(request, response);
            if (raw) {
                copyResponse(ctx, request, response, nettyRequest);
            } else {
                /*
                    Invoker.invoke就是向Invoker类中的ScheduledThreadPoolExecutor提交任务并执行
                    使用ScheduledThreadPoolExecutor是因为可以定时，便于执行异步任务
                    Invoker类可以理解为是各个不同容器任务转向Play任务的一个转换类，他下面有2个静态内部类，分别为InvocationContext和Invocation
                    InvocationContext内存放了当前request映射的controller类的annotation
                    Invocation则为Play任务的基类，他的实现类有：
                        Job：Play的异步任务
                        NettyInvocation：netty请求的实现类
                        ServletInvocation：Servlet容器中的实现类
                        WebSocketInvocation：websocket的实现类
                */
                Invoker.invoke(new NettyInvocation(request, response, ctx, nettyRequest, messageEvent));
            }

        } catch (Exception ex) {
            //处理服务器错误
            serve500(ex, ctx, nettyRequest);
        }
    }

    // 如果msg是websocket，则进行websocket接收
    if (msg instanceof WebSocketFrame) {
        WebSocketFrame frame = (WebSocketFrame) msg;
        websocketFrameReceived(ctx, frame);
    }

    if (Logger.isTraceEnabled()) {
        Logger.trace("messageReceived: end");
    }
}
```

可以很清楚的发现PlayHandler处理请求就是依靠了各个不同的Invocation实现来完成，那我们就来看看Invocation他是如何一步步处理请求的。

### Invocation

Invocation是一个抽象类，继承了Runnable接口，它的run方法是这个类的核心。
```java
public void run() {
    //waitInQueue是一个监视器，监视了这个invocation在队列中等待的时间
    if (waitInQueue != null) {
        waitInQueue.stop();
    }
    try {
        //预初始化，是对语言文件的清空
        preInit();
        //初始化
        if (init()) {
            //运行执行前插件任务
            before();
            //处理request
            execute();
            //运行执行后插件任务
            after();
            //成功执行后的处理
            onSuccess();
        }
    } catch (Suspend e) {
        //Suspend类让请求暂停
        suspend(e);
        after();
    } catch (Throwable e) {
        onException(e);
    } finally {
        _finally();
    }
}
```

这里面的preInit()、before()、after()是没有实现类的，用的就是Invocation类中的方法。我们先来看看Invocation类中对这些方法的实现，然后再来看看他的实现类对这些方法的修改

+ init

```java
public boolean init() {
    //设置当前线程的classloader
    Thread.currentThread().setContextClassLoader(Play.classloader);
    //检查是否为dev模式，是则检查文件是否修改，以进行热加载
    Play.detectChanges();
    //若Play未启动，启动
    if (!Play.started) {
        if (Play.mode == Mode.PROD) {
            throw new UnexpectedException("Application is not started");
        }
        Play.start();
    }
    //InvocationContext中添加当前映射方法的annotation
    InvocationContext.current.set(getInvocationContext());
    return true;
}
```

*Play.start可以先理解为启动Play服务，具体展开将在之后mvc章节详解*

+ execute

execute在Invocation类中是一个抽象方法，需要实现类完成自己的实现

+ onSuccess

```java
public void onSuccess() throws Exception {
    Play.pluginCollection.onInvocationSuccess();
}
```

执行play插件中的执行成功方法

+ onException

```java
public void onException(Throwable e) {
    Play.pluginCollection.onInvocationException(e);
    if (e instanceof PlayException) {
        throw (PlayException) e;
    }
    throw new UnexpectedException(e);
}
```

执行play插件中的执行异常方法，然后抛出异常

+ suspend

```java
public void suspend(Suspend suspendRequest) {
    if (suspendRequest.task != null) {
        WaitForTasksCompletion.waitFor(suspendRequest.task, this);
    } else {
        Invoker.invoke(this, suspendRequest.timeout);
    }
}
```

若暂停的请求有任务，那么调用WaitForTasksCompletion.waitFor进行等待，直到任务完成再继续运行请求
若没有任务，那就将请求通过ScheduledThreadPoolExecutor.schedule延时一段时间后执行

+ _finally

```java
public void _finally() {
    Play.pluginCollection.invocationFinally();
    InvocationContext.current.remove();
}
```

执行play插件中的invocationFinally方法

#### NettyInvocation

NettyInvocation是netty请求使用的实现类，我们来依次看看他对init,execute,onSuccess等方法的实现上相比Invocation改变了什么

+ run

```java
@Override
public void run() {
    try {
        if (Logger.isTraceEnabled()) {
            Logger.trace("run: begin");
        }
        super.run();
    } catch (Exception e) {
        serve500(e, ctx, nettyRequest);
    }
    if (Logger.isTraceEnabled()) {
        Logger.trace("run: end");
    }
}
```

NettyInvocation的run方法和基类唯一的差别就是包了一层try catch来处理服务器错误，这里有一点要注意，可以发现，NettyInvocation的run有一层try catch来处理500错误，PlayHandle中的messageReceived也有一层try catch来处理500错误，我的理解是前者是用来处理业务流程中的错误的，后者是为了防止意外错误引发整个服务器挂掉所以又套了一层做保险

+ init

```java
@Override
public boolean init() {
    //设置当前线程的classloader
    Thread.currentThread().setContextClassLoader(Play.classloader);
    if (Logger.isTraceEnabled()) {
        Logger.trace("init: begin");
    }
    //设置当前线程的request和response
    Request.current.set(request);
    Response.current.set(response);
    try {
        //如果是dev模式检查路径更新
        if (Play.mode == Play.Mode.DEV) {
            Router.detectChanges(Play.ctxPath);
        }
        //如果是prod模式且静态路径缓存有当前请求信息，则从缓存中获取
        if (Play.mode == Play.Mode.PROD && staticPathsCache.containsKey(request.domain + " " + request.method + " " + request.path)) {
            RenderStatic rs = null;
            synchronized (staticPathsCache) {
                rs = staticPathsCache.get(request.domain + " " + request.method + " " + request.path);
            }
            //使用静态资源
            serveStatic(rs, ctx, request, response, nettyRequest, event);
            if (Logger.isTraceEnabled()) {
                Logger.trace("init: end false");
            }
            return false;
        }
        //检查是否为静态资源
        Router.routeOnlyStatic(request);
        super.init();
    } catch (NotFound nf) {
        //返回404
        serve404(nf, ctx, request, nettyRequest);
        if (Logger.isTraceEnabled()) {
            Logger.trace("init: end false");
        }
        return false;
    } catch (RenderStatic rs) {
        //使用静态资源
        if (Play.mode == Play.Mode.PROD) {
            synchronized (staticPathsCache) {
                staticPathsCache.put(request.domain + " " + request.method + " " + request.path, rs);
            }
        }
        serveStatic(rs, ctx, request, response, nettyRequest, this.event);
        if (Logger.isTraceEnabled()) {
            Logger.trace("init: end false");
        }
        return false;
    }

    if (Logger.isTraceEnabled()) {
        Logger.trace("init: end true");
    }
    return true;
}
```

可以看出，NettyInvocation的初始化就是在基类初始化之前判断request的路径，处理静态资源以及处理路径不存在的资源，这里也就说明了访问路由设置中的静态资源是不需要启动play服务的，~~是否意味着可以通过play搭建一个静态资源服务器？~~

+ execute

```java
@Override
public void execute() throws Exception {
    //检查连接是否中断
    if (!ctx.getChannel().isConnected()) {
        try {
            ctx.getChannel().close();
        } catch (Throwable e) {
            // Ignore
        }
        return;
    }
    //渲染之前检查长度
    saveExceededSizeError(nettyRequest, request, response);
    //运行
    ActionInvoker.invoke(request, response);
}
```

ActionInvoker.invoke是play mvc的执行入口方法，这个在之后会进行详解

+ success

```java
@Override
public void onSuccess() throws Exception {
    super.onSuccess();
    if (response.chunked) {
        closeChunked(request, response, ctx, nettyRequest);
    } else {
        copyResponse(ctx, request, response, nettyRequest);
    }
    if (Logger.isTraceEnabled()) {
        Logger.trace("execute: end");
    }
}
```

成功之后就判断是否是chunked类型的request，若是则关闭区块流并返回response，若不是则返回正常的response

#### Job

Job是Play任务的基类，用来处理异步任务，Job虽然继承了Invocation，但他并不会加入至Play的主线程池执行，Job有自己单独的线程池进行处理。因为Job可能需要一个返回值，所以它同时继承了Callable接口来提供返回值。
既然job能提供返回值，那真正起作用的方法就是call()而不是run(),我们来看一下他的call方法

```java
public V call() {
    Monitor monitor = null;
    try {
        if (init()) {
            before();
            V result = null;

            try {
                lastException = null;
                lastRun = System.currentTimeMillis();
                monitor = MonitorFactory.start(getClass().getName()+".doJob()");
                //执行任务并返回结果
                result = doJobWithResult();
                monitor.stop();
                monitor = null;
                wasError = false;
            } catch (PlayException e) {
                throw e;
            } catch (Exception e) {
                StackTraceElement element = PlayException.getInterestingStrackTraceElement(e);
                if (element != null) {
                    throw new JavaExecutionException(Play.classes.getApplicationClass(element.getClassName()), element.getLineNumber(), e);
                }
                throw e;
            }
            after();
            return result;
        }
    } catch (Throwable e) {
        onException(e);
    } finally {
        if(monitor != null) {
            monitor.stop();
        }
        _finally();
    }
    return null;
}
```

可以看出，job的call方法和Invocation的run方法其实差不多，中间执行job的代码其实就是execute方法的实现，但是有一点不同的是，job方法完成后不会调用onSuccess()方法。
这里只是提一下Job与Invocation类的关系，job任务的处理放在之后的插件篇再谈

#### WebSocketInvocation

WebSocketInvocation是websocket请求的实现类，我们来具体看下他对Invocation的方法做了怎么样的复写

+ init

```java
@Override
public boolean init() {
    Http.Request.current.set(request);
    Http.Inbound.current.set(inbound);
    Http.Outbound.current.set(outbound);
    return super.init();
}
```

没什么大的变动，就是在当前线程下添加了出入站的通道信息

+ execute

```java
@Override
public void execute() throws Exception {
    WebSocketInvoker.invoke(request, inbound, outbound);
}
```

execute方法就是将request请求，出入站通道传入WebSocketInvoker.invoke进行处理，WebSocketInvoker.invoke的代码如下，可以看出WebSocketInvoker.invoke方法也就是调用了ActionInvoker.invoke来对请求进行处理。

```java
public static void invoke(Http.Request request, Http.Inbound inbound, Http.Outbound outbound) {
    try {
        if (Play.mode == Play.Mode.DEV) {
            WebSocketController.class.getDeclaredField("inbound").set(null, Http.Inbound.current());
            WebSocketController.class.getDeclaredField("outbound").set(null, Http.Outbound.current());
            WebSocketController.class.getDeclaredField("params").set(null, Scope.Params.current());
            WebSocketController.class.getDeclaredField("request").set(null, Http.Request.current());
            WebSocketController.class.getDeclaredField("session").set(null, Scope.Session.current());
            WebSocketController.class.getDeclaredField("validation").set(null, Validation.current());
        }
        //websocket是没有response的
        ActionInvoker.invoke(request, null);
    }catch (PlayException e) {
        throw e;
    } catch (Exception e) {
        throw new UnexpectedException(e);
    }
}
```

+ onSuccess

```java
@Override
public void onSuccess() throws Exception {
    outbound.close();
    super.onSuccess();
}
```

websocket只有当连接关闭时才会触发onSuccess方法，所以onSuccess相比Invocation也就多了一个关闭出站通道的操作

#### ServletInvocation

ServletInvocation是当使用Servlet容器时的实现类，也就是将程序用war打包后放在Servlet容器后运行的类，ServletInvocation不直接基础于Invocation，而且继承于DirectInvocation，DirectInvocation继承了Invocation，DirectInvocation类添加了一个Suspend字段，用来处理线程暂停或等待操作。
因为在Servlet规范中请求线程的管理交由Servlet容器处理，所以ServletInvocation不是使用Invoker的ScheduledThreadPoolExecutor来执行的，那么在配置文件中设置play.pool数量对于使用Servlet容器的程序是无效的。
既然ServletInvocation不是使用Invoker中的线程池运行的，那他是从什么地方初始化这个类并运行的呢，这点将在本篇的ServletWrapper中再详解

我们来看看ServletInvocation对Invocation中的方法进行了怎样的覆写

+ init

```java
@Override
public boolean init() {
    try {
        return super.init();
    } catch (NotFound e) {
        serve404(httpServletRequest, httpServletResponse, e);
        return false;
    } catch (RenderStatic r) {
        try {
            serveStatic(httpServletResponse, httpServletRequest, r);
        } catch (IOException e) {
            throw new UnexpectedException(e);
        }
        return false;
    }
}
```

很简单的初始化，和NettyInvocation相比就是静态资源的缓存交由Servlet容器处理

+ run

```java
@Override
public void run() {
    try {
        super.run();
    } catch (Exception e) {
        serve500(e, httpServletRequest, httpServletResponse);
        return;
    }
}
```

与NettyInvocation一样

+ execute

```java
@Override
public void execute() throws Exception {
    ActionInvoker.invoke(request, response);
    copyResponse(request, response, httpServletRequest, httpServletResponse);
}
```

ServletInvocation在execute结束后就将结果返回，这里不知道为什么不和NettyInvocation统一一下，都在execute阶段返回结果或者都在onSuccess阶段返回结果

**使用netty的流程我们已经理清楚了，简单来说就是讲netty的请求处理后交由ActionInvoker.invoke来执行，ActionInvoker.invoke也是之后要研究的重点。**

# ServletWrapper

这篇的一开始我们以Server类为入口阐述了使用java命令直接运行时的运行过程，那么如果程序用war打包后放在Servlet容器中是如何运行的呢?
我们可以打开play程序包下resources/war的web.xml来看看默认的web.xml中是怎么配置的。

```xml
  <listener>
      <listener-class>play.server.ServletWrapper</listener-class>
  </listener>
  
  <servlet>
    <servlet-name>play</servlet-name>
    <servlet-class>play.server.ServletWrapper</servlet-class>	
  </servlet>
```

可以看出监听器和servlet入口都是ServletWrapper，那我们从程序的创建开始一点点剖析play的运行过程

## contextInitialized

contextInitialized是在启动时自动加载，主要完成一些初始化的工作，代码如下

```java
public void contextInitialized(ServletContextEvent e) {
    //standalonePlayServer表示这是否是一个独立服务器
    Play.standalonePlayServer = false;

    ClassLoader oldClassLoader = Thread.currentThread().getContextClassLoader();
    //这是程序根目录
    String appDir = e.getServletContext().getRealPath("/WEB-INF/application");
    File root = new File(appDir);

    //play id在web.xml中配置
    final String playId = System.getProperty("play.id", e.getServletContext().getInitParameter("play.id"));
    if (StringUtils.isEmpty(playId)) {
        throw new UnexpectedException("Please define a play.id parameter in your web.xml file. Without that parameter, play! cannot start your application. Please add a context-param into the WEB-INF/web.xml file.");
    }

    Play.frameworkPath = root.getParentFile();
    //使用预编译的文件
    Play.usePrecompiled = true;
    //初始化,初始化过程中会将运行模式强制改为prod
    Play.init(root, playId);
    //检查配置文件中模式是不是dev，是dev提示自动切换为prod
    Play.Mode mode = Play.Mode.valueOf(Play.configuration.getProperty("application.mode", "DEV").toUpperCase());
    if (mode.isDev()) {
        Logger.info("Forcing PROD mode because deploying as a war file.");
    }

    // Servlet 2.4手动加载路径
    // Servlet 2.4 does not allow you to get the context path from the servletcontext...
    if (isGreaterThan(e.getServletContext(), 2, 4)) {
        loadRouter(e.getServletContext().getContextPath());
    }

    Thread.currentThread().setContextClassLoader(oldClassLoader);
}
```

初始化过程有2个地方需要注意

1. 不管配置文件中使用的是什么模式，放在Servlet容器中会统一改为prod
1. 默认使用预编译模式加载

## service

service是servlet处理request的方法，我们先来看一下他的实现代码
```java
@Override
protected void service(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse) throws ServletException, IOException {
    //路由还没初始化时初始化路由
    if (!routerInitializedWithContext) {
        loadRouter(httpServletRequest.getContextPath());
    }

    if (Logger.isTraceEnabled()) {
        Logger.trace("ServletWrapper>service " + httpServletRequest.getRequestURI());
    }

    Request request = null;
    try {
        Response response = new Response();
        response.out = new ByteArrayOutputStream();
        Response.current.set(response);
        //httpServletRequest转为play的request
        request = parseRequest(httpServletRequest);

        if (Logger.isTraceEnabled()) {
            Logger.trace("ServletWrapper>service, request: " + request);
        }
        //检查插件中是否有实现
        boolean raw = Play.pluginCollection.rawInvocation(request, response);
        if (raw) {
            copyResponse(Request.current(), Response.current(), httpServletRequest, httpServletResponse);
        } else {
            //这里不是Invoker.invoke()方法，是invokeInThread
            Invoker.invokeInThread(new ServletInvocation(request, response, httpServletRequest, httpServletResponse));
        }
    } catch (NotFound e) {
        //处理404
        if (Logger.isTraceEnabled()) {
            Logger.trace("ServletWrapper>service, NotFound: " + e);
        }
        serve404(httpServletRequest, httpServletResponse, e);
        return;
    } catch (RenderStatic e) {
        //处理静态资源
        if (Logger.isTraceEnabled()) {
            Logger.trace("ServletWrapper>service, RenderStatic: " + e);
        }
        serveStatic(httpServletResponse, httpServletRequest, e);
        return;
    } catch(URISyntaxException e) {
        //处理404
        serve404(httpServletRequest, httpServletResponse, new NotFound(e.toString()));
        return;
    } catch (Throwable e) {
        throw new ServletException(e);
    } finally {
        //因为servlet的线程会复用，所以要手动删除当前线程内的值
        Request.current.remove();
        Response.current.remove();
        Scope.Session.current.remove();
        Scope.Params.current.remove();
        Scope.Flash.current.remove();
        Scope.RenderArgs.current.remove();
        Scope.RouteArgs.current.remove();
        CachedBoundActionMethodArgs.clear();
    }
}
```

看的出和PlayHandle其实就是一个处理方式，先转为play的request再扔到Invoker类中进行处理，那么我们来看看Invoker.invokeInThread的具体实现过程

```java
public static void invokeInThread(DirectInvocation invocation) {
    boolean retry = true;
    while (retry) {
        invocation.run();
        if (invocation.retry == null) {
            retry = false;
        } else {
            try {
                if (invocation.retry.task != null) {
                    invocation.retry.task.get();
                } else {
                    Thread.sleep(invocation.retry.timeout);
                }
            } catch (Exception e) {
                throw new UnexpectedException(e);
            }
            retry = true;
        }
    }
}
```

invokeInThread就是在当前线程下运行invocation，如果遇到暂停或者等待任务完成就循环直到完成。  
**要注意的一点是，Play 1.2.6未完成servlet模式下的websocket实现，所以如果要用websocket请用netty模式**

# 总结

至此，play的2种正常运行的启动逻辑已经分析完毕，play的启动过程还是在与不同的服务器运行容器打交道，如何将转化后的请求交由ActionInvoker来处理。
下一篇，我们先不看ActionInvoker的具体实现，先看看play对配置文件及路由的分析处理过程。