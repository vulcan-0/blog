---
title: 简单聊聊Tomcat
subtitle: 作为Web服务器的Tomcat
date: 2020-11-06 12:35:11
tags:
    - 服务器
    - Tomcat
---

## 从一个简单的Web服务器说起

Tomcat作为一个Web服务器，需要对浏览器的请求进行解析和响应。如果要基于Java语言实现一个Web服务器，我们需要做些什么事情呢？

Web服务器最基本的功能就是作为一个HTTP协议的服务端实现，HTTP协议是基于TCP的一个应用层协议，而JDK正好提供了对TCP进行封装的Socket和ServerSocket，我们可以基于这两个类去实现一个简单的Web服务器，大致的思路如下：

```java
ServerSocket serverSocket = new ServerSocket(8080);
while (true) {
    Socket socket = serverSocket.accept();
    BufferedReader reader = new BufferedReader(
            new InputStreamReader(socket.getInputStream()));
    StringBuffer buffer = new StringBuffer();
    List<String> lines = new ArrayList<String>();
    String line = "";
    for (char c = (char) reader.read(); !line.equals("\n");
            c = (char) reader.read()) {
        if (c != '\r') {
            buffer.append(c);
        } else {
            line = buffer.toString();
            lines.add(line);
            buffer.delete(0, buffer.length());
        }
    }
    PrintWriter out = new PrintWriter(socket.getOutputStream());
    out.println("HTTP/1.1 200 OK");
    out.println("Content-Type: text/html");
    out.println();
    for (String str : lines) {
        out.println("<p>" + str + "</p>");
    }
    out.flush();
    socket.close();
}
```

上述代码展示了如何监听8080端口、接收浏览器请求、并原样返回浏览器的请求报文。由此，我们可以知道Web服务器最核心的内容是`建立HTTP连接和处理HTTP报文`。

## Tomcat的架构

我们先看下Tomcat的架构图：

![](/images/Tomcat-Architechture.jpg "from https://howtodoinjava.com/tomcat/tomcats-architecture-and-server-xml-configuration-tutorial/")

再看下Tomcat server.xml的默认配置：

```xml
<?xml version='1.0' encoding='utf-8'?>
<Server port="8005" shutdown="SHUTDOWN">
   <Listener className="org.apache.catalina.core.AprLifecycleListener" SSLEngine="on" />
   <Listener className="org.apache.catalina.core.JasperListener" />
   <Listener className="org.apache.catalina.core.JreMemoryLeakPreventionListener" />
   <Listener className="org.apache.catalina.mbeans.GlobalResourcesLifecycleListener" />
   <Listener className="org.apache.catalina.core.ThreadLocalLeakPreventionListener" />
   <GlobalNamingResources>
     <Resource name="UserDatabase" auth="Container"
               type="org.apache.catalina.UserDatabase"
               description="User database that can be updated and saved"
               factory="org.apache.catalina.users.MemoryUserDatabaseFactory"
               pathname="conf/tomcat-users.xml" />
   </GlobalNamingResources>
   <Service name="Catalina">
     <Connector port="8080" protocol="HTTP/1.1"
                connectionTimeout="20000"
                redirectPort="8443" />
     <Connector port="8009" protocol="AJP/1.3" redirectPort="8443" />
     <Engine name="Catalina" defaultHost="localhost">
       <Realm className="org.apache.catalina.realm.LockOutRealm">
         <Realm className="org.apache.catalina.realm.UserDatabaseRealm"
                resourceName="UserDatabase"/>
       </Realm>
       <Host name="localhost"  appBase="webapps"
             unpackWARs="true" autoDeploy="true">
         <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
                prefix="localhost_access_log." suffix=".txt"
                pattern="%h %l %u %t &quot;%r&quot; %s %b" />
       </Host>
     </Engine>
   </Service>
</Server>
```

对照着看，可以看出来，一个Server包含一到多个Service，一个Service包含一到多个Connector和一个Engine，一个Engine包含一到多个Host，一个Host包含一到多个Context，它们分别都代表什么意思呢？我们可以从[Tomcat的官网](https://tomcat.apache.org/tomcat-5.5-doc/architecture/overview.html)中找到答案：

- Server：代表的是整个Tomcat应用。

- Service：主要作用是将一到多个Connector和一个Engine绑定起来。

- Connector：处理和客户端的通讯，Tomcat对应不同的协议会建立不同的Connector，常见的协议包括HTTP、AJP和HTTPS。

- Engine：最外层的容器，处理客户端请求，并返回结果，包含了Tomcat对session集群的支持。

- Host：Engine的子容器，可以理解为一个服务站点，不同的域名绑定到不同的Host。

- Context：Host的子容器，代表的是一个Web应用，我们平时开发的Web应用，放到Tomcat中就对应到一个Context。

- Wrapper（上面没有提到，但非常重要的一个元素）：Context的子容器，一个Wrapper代表的是一个Servlet，对应到浏览器的一个请求。

我们想一下Tomcat为什么要作出这样的抽象？Tomcat是一个Web服务器产品，对于用户，它就是一个服务器，因此，将Tomcat抽象为一个Server。服务器之上，可以提供一系列的服务，因此，Server下面会有Service。Tomcat借助两个核心组件：Connector和Container，得以提供Web服务，这跟我们前面的简单服务器示例提到的服务器核心是一致的：即`建立HTTP连接和处理HTTP报文`（对于HTTP协议如此，可以类比到其他协议）。其中，Container包含四个层级：Engine代表集群、Host代表站点、Context代表Web应用、Wrapper代表单个请求。这样理解，Tomcat的整体架构就非常清晰了。

## Tomcat的容器