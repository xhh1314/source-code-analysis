DubboInver的inverker方法开始说起。
从上篇文章可看到，在执行request时，是从DubboProtocol的refer()方法里拿到一个`DubboInvoker`实例，然后就从protocol层进入exchanger层。经过上面的分析我们已经看到，client其实是一个包装了很多层并且经过多次代理转发的过程。首先client实例是`ReferenceCountExchangeClient`,直接转发给`HeaderExchangeClient`
```java
@Override
   protected Result doInvoke(final Invocation invocation) throws Throwable {
       RpcInvocation inv = (RpcInvocation) invocation;
       final String methodName = RpcUtils.getMethodName(invocation);
       inv.setAttachment(Constants.PATH_KEY, getUrl().getPath());
       inv.setAttachment(Constants.VERSION_KEY, version);

       ExchangeClient currentClient;
       if (clients.length == 1) {
           currentClient = clients[0];
       } else {
           currentClient = clients[index.getAndIncrement() % clients.length];
       }
       try {
           boolean isAsync = RpcUtils.isAsync(getUrl(), invocation);
           boolean isOneway = RpcUtils.isOneway(getUrl(), invocation);
           int timeout = getUrl().getMethodParameter(methodName, Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);
           if (isOneway) {
               boolean isSent = getUrl().getMethodParameter(methodName, Constants.SENT_KEY, false);
               currentClient.send(inv, isSent);
               RpcContext.getContext().setFuture(null);
               return new RpcResult();
           } else if (isAsync) {
               ResponseFuture future = currentClient.request(inv, timeout);
               RpcContext.getContext().setFuture(new FutureAdapter<Object>(future));
               return new RpcResult();
           } else {
               RpcContext.getContext().setFuture(null);
               //client开始执行的地方
               return (Result) currentClient.request(inv, timeout).get();
           }
       } catch (TimeoutException e) {
           throw new RpcException(RpcException.TIMEOUT_EXCEPTION, "Invoke remote method timeout. method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
       } catch (RemotingException e) {
           throw new RpcException(RpcException.NETWORK_EXCEPTION, "Failed to invoke remote method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
       }
   }
   ```

   `HeaderExchangeClient`是把请求转发给了channel，通过构造方法可以看到，其channel实例是exchanger层的`HeaderExchangeChannel`
   ```java
   public HeaderExchangeClient(Client client, boolean needHeartbeat) {
        if (client == null) {
            throw new IllegalArgumentException("client == null");
        }
        this.client = client;
        //channel实例化 注意是把client传递给了HeaderExchangeChannel
        this.channel = new HeaderExchangeChannel(client);
        String dubbo = client.getUrl().getParameter(Constants.DUBBO_VERSION_KEY);
        this.heartbeat = client.getUrl().getParameter(Constants.HEARTBEAT_KEY, dubbo != null && dubbo.startsWith("1.0.") ? Constants.DEFAULT_HEARTBEAT : 0);
        this.heartbeatTimeout = client.getUrl().getParameter(Constants.HEARTBEAT_TIMEOUT_KEY, heartbeat * 3);
        if (heartbeatTimeout < heartbeat * 2) {
            throw new IllegalStateException("heartbeatTimeout < heartbeatInterval * 2");
        }
        if (needHeartbeat) {
            startHeartbeatTimer();
        }
    }
    ```
再来到`HeaderExchangeChannel`的request方法，发现是又转给了channel，那么我们可以推测，这个channel应该就是transport层的channel了,而这channel就是之前提到的NettyClient，这里把request的抽象，转到了transport层的send方法
```java
public ResponseFuture request(Object request, int timeout) throws RemotingException {
        if (closed) {
            throw new RemotingException(this.getLocalAddress(), null, "Failed to send request " + request + ", cause: The channel " + this + " is closed!");
        }
        // create request.
        Request req = new Request();
        req.setVersion("2.0.0");
        req.setTwoWay(true);
        req.setData(request);
        DefaultFuture future = new DefaultFuture(channel, req, timeout);
        try {
            channel.send(req);
        } catch (RemotingException e) {
            future.cancel();
            throw e;
        }
        return future;
    }
  ```
  `NettyClient`继承的类有点多，就不一一列列举了，从顶级父类`AbstractPeer`的send方法执行到，`AbstractClient`的send方法:
  ```java
  public void send(Object message, boolean sent) throws RemotingException {
       if (send_reconnect && !isConnected()) {
           connect();
       }
       //注意这里有一个getChannel方法，子类需要实现的方法，
       //咋们看下NettyClient的实现就知道什么意思了
       Channel channel = getChannel();
       //TODO Can the value returned by getChannel() be null? need improvement.
       if (channel == null || !channel.isConnected()) {
           throw new RemotingException(this, "message can not send, because channel is closed . url:" + getUrl());
       }
       channel.send(message, sent);
   }
   ```
   `NettyClient`的getChannel()实现如下，感觉挺绕的，到了这里又多出来个NettyChannel
   ```java
   @Override
   protected com.alibaba.dubbo.remoting.Channel getChannel() {
     //这里把connect()方法建立的channel对象赋值过来（SocketChannel）
       Channel c = channel;
       if (c == null || !c.isConnected())
           return null;
       return NettyChannel.getOrAddChannel(c, getUrl(), this);
   }
   ```
   `NettyChannel`的getOrAddChannel如下,这里看到是以Netty原生的channel对象为key，包装了一层自己Channel然后存起来了，这个NettyChannel有其特殊作用的，后面会讲到
   ```java
   static NettyChannel getOrAddChannel(org.jboss.netty.channel.Channel ch, URL url, ChannelHandler handler) {
        if (ch == null) {
            return null;
        }
        NettyChannel ret = channelMap.get(ch);
        if (ret == null) {
            NettyChannel nc = new NettyChannel(ch, url, handler);
            if (ch.isConnected()) {
                ret = channelMap.putIfAbsent(ch, nc);
            }
            if (ret == null) {
                ret = nc;
            }
        }
        return ret;
    }
    public void send(Object message, boolean sent) throws RemotingException {
       super.send(message, sent);

       boolean success = true;
       int timeout = 0;
       try {
         //原生的Netty写事件，把数据写入channel，即开始向远端服务发送请求
           ChannelFuture future = channel.write(message);
           if (sent) {
               timeout = getUrl().getPositiveParameter(Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);
               success = future.await(timeout);
           }
           Throwable cause = future.getCause();
           if (cause != null) {
               throw cause;
           }
       } catch (Throwable e) {
           throw new RemotingException(this, "Failed to send message " + message + " to " + getRemoteAddress() + ", cause: " + e.getMessage(), e);
       }

       if (!success) {
           throw new RemotingException(this, "Failed to send message " + message + " to " + getRemoteAddress()
                   + "in timeout(" + timeout + "ms) limit");
       }
   }
   ```
回到`AbstractClient`的send方法里，拿到channel之后，又执行了channel的send方法，其实就是执行了`NettyChannel`的send方法，看上面的代码可以看到，到这里终于调用了Netty原生的channel把数据写出去了，也就是NettyClient建立连接后获取的channel写出去了

看到这里之后，感觉dubbo发送个请求真的是太绕了，绕晕了有没有.

---

发送完request之后还得处理返回的response，response的处理应该由handler来完成，即需要handler处理receive方法,先回过头再来看下`NettyClient`的初始化化逻辑,看构造方法里有一段调用了父类的`wrapChannelHandler()`方法，通过方法名可以看出，这是在初始化父类的handler实例，并且这个handler被包装了。同时可以看到`NettyHanler`是处理Netty读写事件的handler
```java
public class NettyClient extends AbstractClient {

    private static final Logger logger = LoggerFactory.getLogger(NettyClient.class);

    // ChannelFactory's closure has a DirectMemory leak, using static to avoid
    // https://issues.jboss.org/browse/NETTY-424
    private static final ChannelFactory channelFactory = new NioClientSocketChannelFactory(Executors.newCachedThreadPool(new NamedThreadFactory("NettyClientBoss", true)),
            Executors.newCachedThreadPool(new NamedThreadFactory("NettyClientWorker", true)),
            Constants.DEFAULT_IO_THREADS);
    private ClientBootstrap bootstrap;

    private volatile Channel channel; // volatile, please copy reference to use

    public NettyClient(final URL url, final ChannelHandler handler) throws RemotingException {
      //执行handler包装初始化逻辑
        super(url, wrapChannelHandler(url, handler));
    }

    @Override
    protected void doOpen() throws Throwable {
        NettyHelper.setNettyLoggerFactory();
        bootstrap = new ClientBootstrap(channelFactory);
        // config
        // @see org.jboss.netty.channel.socket.SocketChannelConfig
        bootstrap.setOption("keepAlive", true);
        bootstrap.setOption("tcpNoDelay", true);
        bootstrap.setOption("connectTimeoutMillis", getTimeout());
        //初始化NettyHandler时，把这个NettyClient实例传递了过去
        final NettyHandler nettyHandler = new NettyHandler(getUrl(), this);
        bootstrap.setPipelineFactory(new ChannelPipelineFactory() {
            @Override
            public ChannelPipeline getPipeline() {
                NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec(), getUrl(), NettyClient.this);
                ChannelPipeline pipeline = Channels.pipeline();
                pipeline.addLast("decoder", adapter.getDecoder());
                pipeline.addLast("encoder", adapter.getEncoder());
                //把NettyHandler加入拦截链，由这个handler处理netty的各种读写事件
                pipeline.addLast("handler", nettyHandler);
                return pipeline;
            }
        });
    }

    ....省略掉部分代码
    @Override
    protected org.apache.dubbo.remoting.Channel getChannel() {
        Channel c = channel;
        if (c == null || !c.isConnected())
            return null;
        return NettyChannel.getOrAddChannel(c, getUrl(), this);
    }

}
```
点击`wrapChannelHandler()`方法，最后看到的是这样一段逻辑:
```java
public class ChannelHandlers {

    private static ChannelHandlers INSTANCE = new ChannelHandlers();

    protected ChannelHandlers() {
    }

    public static ChannelHandler wrap(ChannelHandler handler, URL url) {
        return ChannelHandlers.getInstance().wrapInternal(handler, url);
    }

    protected static ChannelHandlers getInstance() {
        return INSTANCE;
    }

    static void setTestingChannelHandlers(ChannelHandlers instance) {
        INSTANCE = instance;
    }

    protected ChannelHandler wrapInternal(ChannelHandler handler, URL url) {
      //初始化代码
        return new MultiMessageHandler(new HeartbeatHandler(ExtensionLoader.getExtensionLoader(Dispatcher.class)
                .getAdaptiveExtension().dispatch(handler, url)));
      }
    }
  ```
可以看到`MultiMessageHandler`包装了`HeartbeatHandler`,`HeartbeatHandler`又把请求代理给了`Dispatcher`,我们看下dispatcher包下的类可以看到实现了很多dispatcher方法，
大概有`AllDispatcher`,`DirectDispatcher`,`ExecutionDispatcher`,`MessageOnlyDispatcher` 几个工厂类，默认使用的是AllDispatcher，即把请求代理给了`AllChannelHandler`，回头再来看`AllChannelHandler`的实现。

先看下`NettyClient`的继承关系：`NettyClient extend AbstractClient extend AbstractEndpoint extend AbstractPeer`，通过查看多个父类，可以看到是`AbstractPeer`的初始逻辑
```java
public abstract class AbstractPeer implements Endpoint, ChannelHandler {

    private final ChannelHandler handler;

    private volatile URL url;

    // closing closed means the process is being closed and close is finished
    private volatile boolean closing;

    private volatile boolean closed;

    public AbstractPeer(URL url, ChannelHandler handler) {
        if (url == null) {
            throw new IllegalArgumentException("url == null");
        }
        if (handler == null) {
            throw new IllegalArgumentException("handler == null");
        }
        this.url = url;
        //赋值包装完的handler
        this.handler = handler;
      }

      @Override
       public void received(Channel ch, Object msg) throws RemotingException {
           if (closed) {
               return;
           }
           //收到消息后，也是直接转给了handler处理
           handler.received(ch, msg);
       }
  ```
把client的handler初始化逻辑说完了，再回到`NettyHandler`处理receive()方法:
```java
@Override
   public void messageReceived(ChannelHandlerContext ctx, MessageEvent e) throws Exception {
       NettyChannel channel = NettyChannel.getOrAddChannel(ctx.getChannel(), url, handler);
       try {
           handler.received(channel, e.getMessage());
       } finally {
           NettyChannel.removeChannelIfDisconnected(ctx.getChannel());
       }
   }
  ```
可以看到在收到Netty的读事件后，是直接把请求又转发给了handler,也就是`NettyClient`的receive()方法去处理。

---
** 执行消息解码的handler处理过程

`AbstractPeer`的handler实例是`MultiMessageHandler`,`MultiMessageHandler extends AbstractChannelHandlerDelegate`，重写了receive方法，看样子是实现组合消息的解码，暂时不知道如何使用，这里直接跳过，进入到`HeartbeatHanler` .
```java
@Override
    public void received(Channel channel, Object message) throws RemotingException {
        if (message instanceof MultiMessage) {
            MultiMessage list = (MultiMessage) message;
            for (Object obj : list) {
                handler.received(channel, obj);
            }
        } else {
            handler.received(channel, message);
        }
    }
```

`HeartbeatHanler extends AbstractChannelHandlerDelegate`， 是判断收到的请求是否与心跳相关，如果是心跳request，则回复消息，如果是心跳response，则消息到此结束，不再回复. handler实例是AllChannelHandler
```java
@Override
   public void received(Channel channel, Object message) throws RemotingException {
       setReadTimestamp(channel);
       if (isHeartbeatRequest(message)) {
           Request req = (Request) message;
           if (req.isTwoWay()) {
             //需要回复心跳的情况，回复消息（注意要把消息id带上）
               Response res = new Response(req.getId(), req.getVersion());
               res.setEvent(Response.HEARTBEAT_EVENT);
               channel.send(res);
               if (logger.isInfoEnabled()) {
                   int heartbeat = channel.getUrl().getParameter(Constants.HEARTBEAT_KEY, 0);
                   if (logger.isDebugEnabled()) {
                       logger.debug("Received heartbeat from remote channel " + channel.getRemoteAddress()
                               + ", cause: The channel has no data-transmission exceeds a heartbeat period"
                               + (heartbeat > 0 ? ": " + heartbeat + "ms" : ""));
                   }
               }
           }
           return;
       }
       if (isHeartbeatResponse(message)) {
           if (logger.isDebugEnabled()) {
               logger.debug("Receive heartbeat response in thread " + Thread.currentThread().getName());
           }
           return;
       }
       //不是心跳 最后接着转发给下一个handler
       handler.received(channel, message);
```
到这里就进入了dispatcher的逻辑了，前面已经介绍过了，Dispatcher由`AllChannelHandler`来实现，`AllChannelHandler` extends `WrappedChannelHandler`,这个handler的作用就是同步转异步，把请求异步转发给下一个handler实例处理(`ChannelEventRunnable`内部实现就是转发handler事件)，如果没有理解错，之前的几个handler处理解是由Netty的线程处理，到这一步Netty的线程就执行完毕空闲了。

```java
public class AllChannelHandler extends WrappedChannelHandler {

    public AllChannelHandler(ChannelHandler handler, URL url) {
        super(handler, url);
    }

    @Override
    public void connected(Channel channel) throws RemotingException {
        ExecutorService cexecutor = getExecutorService();
        try {
            cexecutor.execute(new ChannelEventRunnable(channel, handler, ChannelState.CONNECTED));
        } catch (Throwable t) {
            throw new ExecutionException("connect event", channel, getClass() + " error when process connected event .", t);
        }
    }

    @Override
    public void disconnected(Channel channel) throws RemotingException {
        ExecutorService cexecutor = getExecutorService();
        try {
            cexecutor.execute(new ChannelEventRunnable(channel, handler, ChannelState.DISCONNECTED));
        } catch (Throwable t) {
            throw new ExecutionException("disconnect event", channel, getClass() + " error when process disconnected event .", t);
        }
    }

    @Override
    public void received(Channel channel, Object message) throws RemotingException {
        ExecutorService cexecutor = getExecutorService();
        try {
            cexecutor.execute(new ChannelEventRunnable(channel, handler, ChannelState.RECEIVED, message));
        } catch (Throwable t) {
            //TODO A temporary solution to the problem that the exception information can not be sent to the opposite end after the thread pool is full. Need a refactoring
            //fix The thread pool is full, refuses to call, does not return, and causes the consumer to wait for time out
        	if(message instanceof Request && t instanceof RejectedExecutionException){
        		Request request = (Request)message;
        		if(request.isTwoWay()){
        			String msg = "Server side(" + url.getIp() + "," + url.getPort() + ") threadpool is exhausted ,detail msg:" + t.getMessage();
        			Response response = new Response(request.getId(), request.getVersion());
        			response.setStatus(Response.SERVER_THREADPOOL_EXHAUSTED_ERROR);
        			response.setErrorMessage(msg);
        			channel.send(response);
        			return;
        		}
        	}
            throw new ExecutionException(message, channel, getClass() + " error when process received event .", t);
        }
    }

    @Override
    public void caught(Channel channel, Throwable exception) throws RemotingException {
        ExecutorService cexecutor = getExecutorService();
        try {
            cexecutor.execute(new ChannelEventRunnable(channel, handler, ChannelState.CAUGHT, exception));
        } catch (Throwable t) {
            throw new ExecutionException("caught event", channel, getClass() + " error when process caught event .", t);
        }
    }
}
```
`AllChannelHandler`handler实例是`DecodeHandler` extends `AbstractChannelHandlerDelegate`,`AbstractChannelHandlerDelegate`没有具体的代码实现，就是简单转发请求。
直接看`DecodeHandler`的实现,也是代理了handler请求，加入了解码功能，debug下了收到Response时解码情况，收到response后拿出DecodeableRpcResult进行解码,不过之前已经有解码过了,是在Netty的原生handler链里加入了Code2解码类进行解码，所以这里其实没有再解码，应该是一个扩展功能,另外考虑到这是Netty实现的上层，估计是一个整体的抽象，可能会用到解码代码.
```java
public class DecodeHandler extends AbstractChannelHandlerDelegate {

    private static final Logger log = LoggerFactory.getLogger(DecodeHandler.class);

    public DecodeHandler(ChannelHandler handler) {
        super(handler);
    }

    @Override
    public void received(Channel channel, Object message) throws RemotingException {
        if (message instanceof Decodeable) {
            decode(message);
        }

        if (message instanceof Request) {
            decode(((Request) message).getData());
        }

        if (message instanceof Response) {
          //取出DecodeableRpcResult 进行解码
            decode(((Response) message).getResult());
        }

        handler.received(channel, message);
    }

    private void decode(Object message) {
        if (message != null && message instanceof Decodeable) {
            try {
                ((Decodeable) message).decode();
                if (log.isDebugEnabled()) {
                    log.debug("Decode decodeable message " + message.getClass().getName());
                }
            } catch (Throwable e) {
                if (log.isWarnEnabled()) {
                    log.warn("Call Decodeable.decode failed: " + e.getMessage(), e);
                }
            } // ~ end of catch
        } // ~ end of if
    } // ~ end of method decod
}
```

在上一篇文章`DubboProtocol`分析中已经知道,`DecodeHandler`的handler实例是HeaderExchangeHandler,再看HeaderExchangeHandler的实现,该类实现了Channel的所有事件处理，到这一步数据已经从Transport层流回流到了Exchange层。HeaderExchangeHandler是一个比较重要的类，把其中两个重要的方法贴出来：
```java
public void received(Channel channel, Object message) throws RemotingException {
       channel.setAttribute(KEY_READ_TIMESTAMP, System.currentTimeMillis());
       ExchangeChannel exchangeChannel = HeaderExchangeChannel.getOrAddChannel(channel);
       try {
           if (message instanceof Request) {
               // handle request.
               Request request = (Request) message;
               //事件指的应该就是心跳
               if (request.isEvent()) {
                   handlerEvent(channel, request);
               } else {
                 //双向请求，指的就是需要回复的请求
                   if (request.isTwoWay()) {
                       Response response = handleRequest(exchangeChannel, request);
                       channel.send(response);
                   } else {
                       handler.received(exchangeChannel, request.getData());
                   }
               }
           } else if (message instanceof Response) {
             //处理收到的response
               handleResponse(channel, (Response) message);
           } else if (message instanceof String) {
               if (isClientSide(channel)) {
                   Exception e = new Exception("Dubbo client can not supported string message: " + message + " in channel: " + channel + ", url: " + channel.getUrl());
                   logger.error(e.getMessage(), e);
               } else {
                   String echo = handler.telnet(channel, (String) message);
                   if (echo != null && echo.length() > 0) {
                       channel.send(echo);
                   }
               }
           } else {
               handler.received(exchangeChannel, message);
           }
       } finally {
           HeaderExchangeChannel.removeChannelIfDisconnected(channel);
       }
   }
//处理收到的request请求
   Response handleRequest(ExchangeChannel channel, Request req) throws RemotingException {
     //够着response对象，把request的id赋值上，对请求者来说，这是一个标识
       Response res = new Response(req.getId(), req.getVersion());
       //这个不知道什么情况会跳到这里
       if (req.isBroken()) {
           Object data = req.getData();

           String msg;
           if (data == null) msg = null;
           else if (data instanceof Throwable) msg = StringUtils.toString((Throwable) data);
           else msg = data.toString();
           res.setErrorMessage("Fail to decode request due to: " + msg);
           res.setStatus(Response.BAD_REQUEST);

           return res;
       }
       // find handler by message class.
       Object msg = req.getData();
       try {
           // handle data.
           //开始准备调用业务层的 @service方法，类似springMvc调用controller方法
           //这个handler的实例是DubboProtocol的一个内部类
           Object result = handler.reply(channel, msg);
           res.setStatus(Response.OK);
           res.setResult(result);
       } catch (Throwable e) {
           res.setStatus(Response.SERVICE_ERROR);
           res.setErrorMessage(StringUtils.toString(e));
       }
       return res;
   }

   static void handleResponse(Channel channel, Response response) throws RemotingException {
        if (response != null && !response.isHeartbeat()) {
            DefaultFuture.received(channel, response);
        }
    }

```
`handleResponse()`的代码逻辑要和`HeaderExchangeChannel.receive()`方法联系起来看，因为发送request和处理收到response是不同的线程处理的，这里也是想明白发送request之后，如何拿到response的关键。

 下面单独看下`DefaultFuture`的received()方法:
 ```java
 public static void received(Channel channel, Response response) {
       try {
         //这里是根据消息ID来匹配收到的response属于哪个request
         //所以server端在回复消息时都需要带上request传递过去的id
           DefaultFuture future = FUTURES.remove(response.getId());
           if (future != null) {
               future.doReceived(response);
           } else {
               logger.warn("The timeout response finally returned at "
                       + (new SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS").format(new Date()))
                       + ", response " + response
                       + (channel == null ? "" : ", channel: " + channel.getLocalAddress()
                       + " -> " + channel.getRemoteAddress()));
           }
       } finally {
           CHANNELS.remove(response.getId());
       }
   }
 private void doReceived(Response res) {
       lock.lock();
       try {
         //赋值结果
           response = res;
           if (done != null) {
             //唤醒request等待线程，业务层执行的get()方法即可拿到结果，阻塞结束
               done.signal();
           }
       } finally {
           lock.unlock();
       }
       if (callback != null) {
           invokeCallback(callback);
       }
   }

   @Override
    public Object get(int timeout) throws RemotingException {
        if (timeout <= 0) {
            timeout = Constants.DEFAULT_TIMEOUT;
        }
        if (!isDone()) {
            long start = System.currentTimeMillis();
            lock.lock();
            try {
                while (!isDone()) {
                  //这里是一个while循环一直判断是否拿到结果，如果没有拿到结果就一直等待
                  //如上doReceived里所示，当有结果了这个线程会被唤醒
                    done.await(timeout, TimeUnit.MILLISECONDS);
                    if (isDone() || System.currentTimeMillis() - start > timeout) {
                        break;
                    }
                }
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            } finally {
                lock.unlock();
            }
            if (!isDone()) {
                throw new TimeoutException(sent > 0, channel, getTimeoutMessage(false));
            }
        }
        return returnFromResponse();
    }
 ```

HeaderExchangeHandler 的handler实例是DubboProtocol里的一个`requestHandler`实例
