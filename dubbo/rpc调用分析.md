InvokerInvocationHandler 实现了jdk动态代理接口
默认使用FailoverClusterInvoker执行调用，即失败可重试的策略
执行到DubboInvoker，调用remote模块代码执行远程调用，起始代码在ReferenceCountExchangeClient.request()
NettyChannel封装了netty的实现
DefaultFuture存储了Channel的实例，远程调用结果从这个对象中获取数据

DecodeHandler的received方法是谁调用的？
HeaderExchangeHandler
ChannelEventRunnable
AllChannelHandler
以上几个类待仔细分析

HeaderExchangeChannel 的channel是NettyClient(继承自AbstractPeer)

`NettyClient extend AbstractClient extend AbstractEndpoint extend AbstractPeer`

NettyHanler 的handler实例是NettyClient，receive到请求后，跳转到AbstracPeer的receive方法，AbstractPeer的handler实例是MultiMessageHandler

`MultiMessageHandler extends AbstractChannelHandlerDelegate`，重写了receive方法，看样子是实现组合消息的解码，handler实例是`HeartbeatHanler` .
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
             //回复消息时把id带上
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
       handler.received(channel, message);
   }
```


`AllChannelHandler`继承自`WrappedChannelHandler`（handler实例是DecodeHandler，继承自AbstractChannelHandlerDelegate),负责把请求异步转发给handler实例处理(`ChannelEventRunnable`内部实现就是转发handler事件)，异步执行后的代码，往下看会发现，handler最后会传到HeaderExchangeHandler
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
`AbstractChannelHandlerDelegate`没有具体的代码实现，就是简单转发请求，看`DecodeHandler`也是代理了handler请求，加入了解码功能，debug下了收到Response时解码情况，收到response后拿出DecodeableRpcResult进行解码,不过之前已经有解码过了（Code2解码），所以这里其实没有再解码，应该是一个扩展功能,另外考虑到这是netty实现的上层，估计是一个整体的抽象，可能会用到解码代码.
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
    } // ~ end of method decode

}
```


AbstractChannelHandlerDelegate 的handler实例是HeaderExchangeHandler,再看HeaderExchangeHandler的实现,该类实现了Channel的所有事件处理，到这一步数据已经从Transport层流回流到了Exchange层。HeaderExchangeHandler是一个比较重要的类，把其中两个重要的方法贴出来：
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
                 //双向请求，应该指的就是需要回复的请求
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
//处理收到的request请求
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

```
单独看下handleResponse方法，这个里的代码逻辑要和`HeaderExchangeChannel.receive()`方法联系起来看，这里也是整理发送请求再到收到请求结果思路的关键。收到的response请求最终到这里结束

```java
static void handleResponse(Channel channel, Response response) throws RemotingException {
     if (response != null && !response.isHeartbeat()) {
         DefaultFuture.received(channel, response);
     }
 }
 ```
 `DefaultFuture`的方法:
 ```java
 private void doReceived(Response res) {
       lock.lock();
       try {
         //赋值结果
           response = res;
           if (done != null) {
             //唤醒等待线程，业务层执行的get()方法即可拿到结果，阻塞结束
               done.signal();
           }
       } finally {
           lock.unlock();
       }
       if (callback != null) {
           invokeCallback(callback);
       }
   }
 ```

HeaderExchangeHandler 的handler实例是DubboProtocol里的一个`requestHandler`实例,看下其reply方法
```java
private ExchangeHandler requestHandler = new ExchangeHandlerAdapter() {
  public Object reply(ExchangeChannel channel, Object message) throws RemotingException {
            if (message instanceof Invocation) {
                Invocation inv = (Invocation) message;
                Invoker<?> invoker = getInvoker(channel, inv);
                // need to consider backward-compatibility if it's a callback
                if (Boolean.TRUE.toString().equals(inv.getAttachments().get(IS_CALLBACK_SERVICE_INVOKE))) {
                    String methodsStr = invoker.getUrl().getParameters().get("methods");
                    boolean hasMethod = false;
                    if (methodsStr == null || methodsStr.indexOf(",") == -1) {
                        hasMethod = inv.getMethodName().equals(methodsStr);
                    } else {
                        String[] methods = methodsStr.split(",");
                        for (String method : methods) {
                            if (inv.getMethodName().equals(method)) {
                                hasMethod = true;
                                break;
                            }
                        }
                    }
                    if (!hasMethod) {
                        logger.warn(new IllegalStateException("The methodName " + inv.getMethodName()
                                + " not found in callback service interface ,invoke will be ignored."
                                + " please update the api interface. url is:"
                                + invoker.getUrl()) + " ,invocation is :" + inv);
                        return null;
                    }
                }
                RpcContext.getContext().setRemoteAddress(channel.getRemoteAddress());
                return invoker.invoke(inv);
            }
            throw new RemotingException(channel, "Unsupported request: "
                    + (message == null ? null : (message.getClass().getName() + ": " + message))
                    + ", channel: consumer: " + channel.getRemoteAddress() + " --> provider: " + channel.getLocalAddress());
        }
```
代码执行到这里，其实dubbo的数据流就已经进入到了protocol层

NettyHandler 

DubboProtocol 没有实现send() ,receive()是当message instance Invocation 调用reply,感觉像是心跳之类的情况才会执行到receive




11月15日看代码总结：
客户端发送请求，肯定应该从client开始，而client就是NettyClient,不过dubbo不是直接把请求转给nettyClient的，而是从HeaderExchangeClient 转到HeaderExchangeChannel,再转发给NettyClient.
HeaderExchangeChannel的request()方法里，初始化了DefaultFuture对象，即把request请求的结果future初始化了，待收到response后，再把这个future里的response初始化，这个DefaultFuture里存储了request的id,返回的response里也有id字段，最后要根据这个进行匹配,把response的值赋值到DefaultFuture里

NettyClient 继承自AbstractClient ，AbsClient定义了发起连接请求，重连的方法，子类只需实现其doConnect等方法即可；再看AbstractEndpoint,这个类定义了Codec的初始化逻辑，实现了getCodec()方法，子类NettyClient在初始化netty handler 的时候会用到这个逻辑，加入handler拦截链进行编码解码，如果再看NettyServer会发现也继承了AbstractEndpoint；这个类又继承自AbstractPeer，这个父抽象类里定义好了send() receive()委派给谁的实现,最后的发送和接收请求都委派给MultiMessageHandler
