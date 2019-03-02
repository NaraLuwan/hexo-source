---
title: jetty.xml解析
date: 2017-03-07 12:36:00
tags: Jetty
categories: 
- Java
- Web
sticky: 0
---
jetty有一种启动方式是在jetty的根目录中运行命令行：java -jar start.jar，这个命令会调用apache的XmlConfiguration工具类作为启动类，这个类会默认读取/etc/jetty.xml文件，加载jetty启动所必须的配置。接下来分段研究：
```
<Configure id="Server" class="org.eclipse.jetty.server.Server">
```
这段配置是整个配置文件的root元素，它其实是调用org.eclipse.jetty.server类的默认构造函数来创建一个server对象，对应源码：
```
public Server(){
    this((ThreadPool)null);
}
```
创建了一个空的server，那么接下来就是为这个server设置各种对象及属性了。继续看：<!-- more -->
```
<Set name="ThreadPool">
      <New class="org.eclipse.jetty.util.thread.QueuedThreadPool">
        <Set name="minThreads">100</Set>
        <Set name="maxThreads">1000</Set>
        <Set name="SpawnOrShrinkAt">2</Set>
      </New>
</Set>
```
这段配置是为当前server设置线程池，从xml元素也可以看出来它其实是调用server的SetThreadPool方法：
```
server.setThreadPool(threadPool);
```
然后new了一个org.eclipse.jetty.util.thread.QueuedThreadPool对象，并设置它的最小线程数、最大线程数以及最多有多少task可以暂时放到队列里待会执行，还有其他很多参数具体场景可能需要不同的配置。再继续看：
```
<Call name="addConnector">
      <Arg>
          <New class="org.eclipse.jetty.server.nio.SelectChannelConnector">
            <Set name="host"><Property name="jetty.host" /></Set>
            <Set name="port"><Property name="jetty.port" default="8080"/></Set>
            <Set name="maxIdleTime">3000</Set>
            <Set name="Acceptors">2</Set>
            <Set name="statsOn">false</Set>
            <Set name="confidentialPort">9043</Set>
            <Set name="lowResourcesConnections">20000</Set>
            <Set name="lowResourcesMaxIdleTime">10000</Set>
          </New>
     </Arg>
</Call>
```
接下来这一段配置是调用addConnector方法，为当前的server对象添加一个connector，即监听器，jetty默认情况下创建的就是SelectChannelConnector，也就是所谓的nioConnector，关于bio,nio和nio的区别网上资料很多可自行百度。这个方法传入了很多参数，host、端口port、连接最大的空闲时间maxIdleTime、创建selector个数Acceptors等等，后面这些参数实际项目配置的都不同，但是都会有一个经验值。关于SelectChannelConnector的详细介绍参考这里： http://blog.csdn.net/kobejayandy/article/details/20166215 再继续看：
```
<Set name="handler">
     <New id="Handlers" class="org.eclipse.jetty.server.handler.HandlerCollection">
        <Set name="handlers">
         <Array type="org.eclipse.jetty.server.Handler">
           <Item>
             <New id="Contexts" class="org.eclipse.jetty.server.handler.ContextHandlerCollection"/>
           </Item>
           <Item>
             <New id="DefaultHandler" class="org.eclipse.jetty.server.handler.DefaultHandler"/>
           </Item>
         </Array>
        </Set>
     </New>
</Set>
```
这段配置是去调用server的setHandler方法为当前的server设置handler属性，这里默认创建的是org.mortbay.jetty.handler.HandlerCollection，接着给这个handlerCollection创建了两个handler分别是ContextHandlerCollection和DefaultHandler。我们知道当jetty收到一个http请求之后，会调用server来处理这个请求，而server对象是调用内部的handler来处理，所以关键还是看HandlerCollection是怎么处理的吧，也就是它的handle方法，源码：
```
@Override
public void handle(String target, Request baseRequest, HttpServletRequest request, HttpServletResponse response)
        throws IOException, ServletException
    {
        if (_handlers!=null && isStarted())
        {
            MultiException mex=null;

            for (int i=0;i<_handlers.length;i++)
            {
                try
                {
                    _handlers[i].handle(target,baseRequest, request, response);
                }
                catch(IOException e)
                {
                    throw e;
                }
                catch(RuntimeException e)
                {
                    throw e;
                }
                catch(Exception e)
                {
                    if (mex==null)
                        mex=new MultiException();
                    mex.add(e);
                }
            }
            if (mex!=null)
            {
                if (mex.size()==1)
                    throw new ServletException(mex.getThrowable(0));
                else
                    throw new ServletException(mex);
            }

        }
}
```
代码其实很简单，主要是那个for循环遍历当前collection里所有的handler，依次调用它们的handle方法。所以我们平常项目里有些初始化操作会写在handler类里边在jetty启动的时候就会执行初始化。

好了，到这里整个jetty.xml的文件最重要的一些内容就配置完了，接下来还有一些其他的配置，一般情况下用默认值就好了。

<hr />

 