---
title: 聊聊Tomcat
subtitle: 作为Web服务器的Tomcat
date: 2020-11-06 12:35:11
tags:
    - 服务器
    - Tomcat
---

## 从一个简单的Web服务器说起

Tomcat作为一个Web服务器，需要对浏览器的请求进行解析和响应。如果要基于Java语言实现一个Web服务器，我们需要做哪些事情呢？

Web服务器最基本的功能就是作为一个`HTTP协议的服务端实现`，我们知道HTTP协议是基于TCP的一个应用层协议，在JDK中，有两个对TCP进行了封装的类：Socket和ServerSocket，我们可以基于这两个类去实现一个简单的Web服务器，大致的思路如下：

```java
ServerSocket serverSocket = new ServerSocket(8080);
while (true) {
    Socket socket = serverSocket.accept();
    InputStream input = socket.getInputStream();
    byte[] bytes = new byte[2048];
    int i = input.read(bytes);
    StringBuffer buffer = new StringBuffer();
    for (int j = 0; j < i; j++) {
        buffer.append((char) bytes[j]);
    }
    System.out.println(buffer.toString());

    PrintWriter out = new PrintWriter(socket.getOutputStream());
    out.println("HTTP/1.1 200 OK");
    out.println("Content-Type: text/html");
    out.println();
    out.println("<h1>Hello!</h1>");
    out.flush();
    socket.close();
}
```

上述代码展示了如何监听8080端口、接收浏览器请求、并给浏览器返回请求成功的报文。由此，我们可以知道Web服务器最核心的内容是`建立HTTP连接和处理HTTP报文`。

## Tomcat的架构

我们先看下Tomcat的架构图：

![](/images/Tomcat-Architechture.jpg "from https://howtodoinjava.com/tomcat/tomcats-architecture-and-server-xml-configuration-tutorial/")

再看下Tomcat server.xml的默认配置（下面提到的Tomcat配置均来自于【Tomcat 5.5.23】）：

```xml
<Server port="8005" shutdown="SHUTDOWN">
  <Listener className="org.apache.catalina.core.AprLifecycleListener" />
  <Listener className="org.apache.catalina.mbeans.ServerLifecycleListener" />
  <Listener className="org.apache.catalina.mbeans.GlobalResourcesLifecycleListener" />
  <Listener className="org.apache.catalina.storeconfig.StoreConfigLifecycleListener" />

  <Service name="Catalina">
    <Connector port="8080" maxHttpHeaderSize="8192"
        maxThreads="150" minSpareThreads="25" maxSpareThreads="75"
        enableLookups="false" redirectPort="8443" acceptCount="100"
        connectionTimeout="20000" disableUploadTimeout="true" 
        URIEncoding="UTF-8" useBodyEncodingForURI="true" />
    <Connector port="8009" 
        enableLookups="false" redirectPort="8443" protocol="AJP/1.3" />

    <Engine name="Catalina" defaultHost="localhost">
      <Host name="localhost" appBase="webapps"
          unpackWARs="true" autoDeploy="true"
          xmlValidation="false" xmlNamespaceAware="false">
        <Valve className="org.apache.catalina.valves.AccessLogValve"
            directory="logs"  prefix="localhost_access_log." suffix=".txt"
            pattern="common" resolveHosts="false" />
        <Valve className="org.apache.catalina.valves.FastCommonAccessLogValve"
            directory="logs"  prefix="localhost_access_log." suffix=".txt"
            pattern="common" resolveHosts="false" />
      </Host>
    </Engine>
  </Service>
</Server>
```

对照着看，可以看出来，一个Server包含一到多个Service，一个Service包含一到多个Connector和一个Engine，一个Engine包含一到多个Host，一个Host包含一到多个Context，它们分别都代表什么意思呢？我们可以从[Tomcat的官网](https://tomcat.apache.org/tomcat-5.5-doc/architecture/overview.html)中找到答案：

- Server：代表的是整个Tomcat服务器。Server默认监听8005端口，在收到SHUTDOWN命令时，优雅停止服务器。

- Service：主要作用是将一到多个Connector和一个Engine绑定起来。

- Connector：处理与客户端之间的通讯，Tomcat对应不同的协议会建立不同的Connector，目前支持的协议包括HTTP和AJP等。

- Engine：最外层的容器，处理客户端请求，并返回结果。Engine提供了对session集群的支持。

- Host：Engine的子容器，可以理解为一个服务站点，不同的域名绑定到不同的Host。

- Context：Host的子容器，代表的是一个Web应用，我们平时开发的Web应用，放到Tomcat中就对应到一个Context。

- Wrapper（上面没有提到，但非常重要的一个元素）：Context的子容器，一个Wrapper代表的是一个Servlet，对应到浏览器的一个请求。

我们想一下Tomcat为什么要作出这样的抽象？Tomcat是一个Web服务器，所以，将Tomcat抽象为一个Server是合理的。服务器之上，可以提供一系列的服务，因此，Server下面会有Service。去掉Service这层抽象，直接通过Server将Connector和Container绑定起来可不可以？实际上也是可以的，但是这样显然会比中间多一层Service的扩展性差。我们前面提到Web服务器的两个核心工作是，`建立HTTP连接和处理HTTP报文`（对于AJP协议，是建立AJP连接和处理AJP报文），Tomcat抽象出Connector和Container分别来处理这两个任务。其中，Container包含四个层级：Engine（代表集群）、Host（代表站点）、Context（代表Web应用）、Wrapper（代表单个请求）。

### Connector

我们一起看下Connector的核心工作包含哪些方面。

#### 线程池

我们再次看下上面提到的简单Web服务器示例代码：

```java
ServerSocket serverSocket = new ServerSocket(8080);
while (true) {
    Socket socket = serverSocket.accept();
    ......
}
```

我们会发现这个服务器同一时间只能处理一个请求，因为它是单线程的。我们应该引入多线程来让服务器能够在同一时间处理多个请求，引入了线程就意味着我们必须对线程池进行管理。我们看下下面的例子：

```java
Executor executor = new ThreadPoolExecutor(Constants.DEFAULT_CORE_POOL_SIZE,
        Constants.DEFAULT_MAX_POOL_SIZE,
        Constants.DEFAULT_THREAD_KEEP_ALIVE_TIME,
        TimeUnit.MILLISECONDS,
        new LinkedBlockingQueue<Runnable>());
......

Socket socket = serverSocket.accept();
......
// connection代表对HTTP连接的封装
connection.setSocket(socket);
executor.execute(connection);
```

上述代码展示了多线程的引入，我们看下Tomcat server.xml中的connector配置：

```xml
<Connector port="8080" maxHttpHeaderSize="8192"
    maxThreads="150" minSpareThreads="25" maxSpareThreads="75"
    enableLookups="false" redirectPort="8443" acceptCount="100"
    connectionTimeout="20000" disableUploadTimeout="true" 
    URIEncoding="UTF-8" useBodyEncodingForURI="true" />
```

其中就包含了一些线程池的管理参数，如：maxThreads、minSpareThreads、maxSpareThreads等，由此可见，Tomcat也是通过多线程技术来让服务器具备同一时间处理多个请求的能力的。

#### HTTP 1.1

在HTTP 1.1之后，服务器与浏览器是可以保持长连接的，Tomcat是如何进行处理的呢？我们看下下面的例子：

```java
do {
    Thread.sleep(Constants.DEFAULT_KEEP_ALIVE_INTERVAL);
    keepAliveTime -= Constants.DEFAULT_KEEP_ALIVE_INTERVAL;

    // if 心跳检查
    // keepIt = true
    // else 
    // keepIt = false
    // keepAliveTime = Constants.DEFAULT_KEEP_ALIVE_TIME;
    // 处理请求
    ......

    boolean requestAlive = request != null ? request.isAlive() : false;
    keepAlive = (keepIt || requestAlive) && keepAliveTime > 0;
} while (keepAlive);

......
socket.colse();
```

可以看到，只要keepAlive是true，就会不断处理新的请求，直到keepAlive变为false，才会断开连接。keepAlive由三个变量的值：keepIt、requestAlive、keepAliveTime共同决定。requestAlive表示请求是否还存活，具体是通过判断请求头中的Connection字段来确定的，示例代码如下：

```java
public boolean isAlive() {
    List<String> values = headers.get(Constants.CONNECTION.toLowerCase());
    if (values != null && !values.isEmpty()) {
        if (Constants.KEEP_ALIVE.equals(values.get(values.size() - 1))) {
            return true;
        }
    }
    return false;
}
```

示例代码中的keepIt表示什么意思呢？实现了HTTP 1.1协议的浏览器，会定期向服务器发起心跳检查，如果该请求是一个心跳检查，那么keepIt就为true。

最后还会判断下连接的保持时间是否超过了keepAliveTime，如果超过了则断开连接。当然每次接收到新的请求的时候，是会重置keepAliveTime的，如下：

```java
keepAliveTime = Constants.DEFAULT_KEEP_ALIVE_TIME;
```

[comment]: <> (关于AJP、HTTPS、NIO的内容，请听下回分解)

#### Request & Response

Connector在收到HTTP请求后，会创建Request和Response对象，并把对象传递给Container，Container再将对象传递给servlet，这样servlet程序员就可以拿到这两个对象，从而实现Web接口。

```java
request = new HttpRequest(input);
......

response = new HttpResponse(output);
......

HttpRequestFacade requestFacade = new HttpRequestFacade(request);
HttpResponseFacade responseFacade = new HttpResponseFacade(response);
......
container.invoke(requestFacade, responseFacade);
```

相关类图如下：

![](/images/Tomcat-Request-Response.jpg)

可以看到，上述示例使用了[外观模式](https://www.runoob.com/design-pattern/facade-pattern.html)对Request和Response对象进行封装，为什么要使用外观模式呢？我们知道外观模式主要解决的问题是`降低访问复杂系统的内部子系统时的复杂度，简化客户端与之的接口`，而这里的封装也是为了对servlet程序员屏蔽HttpRequest和HttpResponse的特有方法，只给用户暴露HttpServletRequest和HttpServletResponse接口定义的方法。

### Container

我们下面看下Container的Pipeline模式和Lifecycle接口。

#### Pipeline

Container采用了Pipeline模式，它们的类关系图如下：

![](/images/Tomcat-Container-Pipeline-class.jpg)

Container的invoke执行过程如下：

![](/images/Tomcat-Container-Pipeline-invoke.jpg)

可以看到，Connector在创建了Request和Response对象后，再经过StandardEngine、StandardHost、StandardContext、StandardWrapper等的层层调用后，消息才最终到达了FilterChain。这样做有什么好处呢？我们可以发现，经过这样的调用链封装，容器的每一层都可以拿到Request和Response对象，从而可以自由执行自己那部分的逻辑了。结合上面的类图，我们发现每一个Container都是一个Pipeline，我们可以往每个Pipeline里面添加定制化的Value，如此，Container就拥有了更好的扩展性。

[comment]: <> (关于filter的内容，请听下回分解)

#### Lifecycle

Container实现了Lifecycle接口，以提供生命周期管理的功能，其中，类的关系图如下（所有的Container都实现了Lifecycle接口，此处使用StantardContext作为示例）：

![](/images/Tomcat-Container-Lifecycle.jpg)

StantardContext的示例代码如下：

```java
public void start() {
    lifecycleSupport.fireLifecycleEvent(LifecycleEvent.BEFORE_START_EVENT, "");
    ......

    lifecycleSupport.fireLifecycleEvent(LifecycleEvent.START_EVENT, "");
    Container[] containers = findChildren();
    for (Container container : containers) {
        container.start();
    }

    ......
    lifecycleSupport.fireLifecycleEvent(LifecycleEvent.AFTER_START_EVENT, "");
}

public void stop() {
    lifecycleSupport.fireLifecycleEvent(LifecycleEvent.BEFORE_STOP_EVENT, "");
    ......

    lifecycleSupport.fireLifecycleEvent(LifecycleEvent.STOP_EVENT, "");
    Container[] containers = findChildren();
    for (Container container : containers) {
        container.stop();
    }

    ......
    lifecycleSupport.fireLifecycleEvent(LifecycleEvent.AFTER_STOP_EVENT, "");
}

public void addLifecycleListener(LifecycleListener lifecycleListener) {
    lifecycleSupport.addLifecycleListener(lifecycleListener);
}

public void removeLifecycleListener(LifecycleListener lifecycleListener) {
    lifecycleSupport.removeLifecycleListener(lifecycleListener);
}
```

LifecycleSupport的示例代码如下：

```java
public void fireLifecycleEvent(String type, Object data) {
    LifecycleEvent lifecycleEvent = new LifecycleEvent(lifecycle, type, data);
    List<LifecycleListener> currentListeners;
    synchronized (listeners) {
        currentListeners = new ArrayList<LifecycleListener>(listeners);
    }
    for (LifecycleListener lifecycleListener : currentListeners) {
        lifecycleListener.lifecycleEvent(lifecycleEvent);
    }
}

public void addLifecycleListener(LifecycleListener lifecycleListener) {
        listeners.add(lifecycleListener);
}

public void removeLifecycleListener(LifecycleListener lifecycleListener) {
    listeners.remove(lifecycleListener);
}
```

Container实现了Lifecycle接口的start()和stop()方法，在启动或关闭的时候会调用子容器的start()和stop()方法，并发出START_EVENT、STOP_EVENT等生命周期事件。

我们在文章前面提到的Tomcat配置的Listener，其实就是生命周期监听器：

```xml
<Server port="8005" shutdown="SHUTDOWN">
  <Listener className="org.apache.catalina.core.AprLifecycleListener" />
  <Listener className="org.apache.catalina.mbeans.ServerLifecycleListener" />
  <Listener className="org.apache.catalina.mbeans.GlobalResourcesLifecycleListener" />
  <Listener className="org.apache.catalina.storeconfig.StoreConfigLifecycleListener" />
  ......
</Server>
```

#### Filter

在编写Web应用程序的时候，我们经常需要编写Filter，Filter的实现是怎么样的呢？我们上面其实已经有说明，Request和Response对象从Connector产生，一直传递到了FilterChain，FilterChain接下来会做什么处理呢？我们看下下面的类关系图：

![](/images/Tomcat-Container-Filter-class.jpg)

再看下调用关系：

![](/images/Tomcat-Container-Filter-doFilter.jpg)

通过观察可以发现，Filter其实也使用了Pipeline模式，ApplicationFilterChain在接收到请求时，会逐个获取并调用Filter，等所有Filter都调用完后，再去调用servlet。

至此，Tomcat的核心架构和核心调用链路就清晰了。

[comment]: <> (关于Logger、Loader、Session、Realm、Digester、Manager、JMX的内容，请自行学习)

> 声明：以上提到的示例代码并非Tomcat的源码，而是为了便于理解而构建的。我根据《深入剖析Tomcat》一书，编写了一个简单的Servlet容器，有兴趣的可以到[这里](https://github.com/vulcan-0/simple-tomcat)查看。