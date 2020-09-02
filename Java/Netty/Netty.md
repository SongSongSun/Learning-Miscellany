## 初识Netty

#### 1.Netty简介

Netty是基于NIO实现的**异步事件驱动的网络应用程序框架**
用于快速开发可维护的高性能协议服务器和客户端.

Netty 是最流行的 NIO 框架，它已经得到成百上千的商业、商用项目验证，许多框架和开源组件的底层 rpc 都是使用的 Netty，如 Dubbo、Elasticsearch 等等。下面是官网给出的一些 Netty 的特性：

##### 设计

- 适用于各种传输类型的统一API-阻塞和非阻塞套接字
- 基于灵活且可扩展的事件模型，可将关注点明确分离
- 高度可定制的线程模型-单线程，一个或多个线程池，例如SEDA
- 真正的无连接数据报套接字支持

##### 使用方便

- 记录良好的Javadoc，用户指南和示例
- 没有其他依赖关系，JDK 5（Netty 3.x）或6（Netty 4.x）就足够了

##### 性能

- 更高的吞吐量，更低的延迟
- 减少资源消耗
- 减少不必要的内存复制

##### 安全

- 完整的SSL / TLS和StartTLS支持



这些特性可以先有一个简单的印象即可

#### 2.搭建简单的HTTP服务器

我这边用到的Netty版本是：

```xml
<dependency>
            <groupId>io.netty</groupId>
            <artifactId>netty-all</artifactId>
            <version>4.1.48.Final</version>
</dependency>
```

编写一个简单的HTTP服务器，在浏览器上输入访问的地址来访问我们的服务，然后返回响应。

服务端代码如下

```java
@Slf4j
public class Main {
    public static void main(String[] args) {
        log.info("=========<<<<<<<启动Netty中>>>>>>=========");

        //构造两个线程组，bossGroup用来处理连接，workGroup用来处理请求
        NioEventLoopGroup bossGroup = new NioEventLoopGroup();
        NioEventLoopGroup workGroup = new NioEventLoopGroup();
        try {
            //服务端启动辅助类
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workGroup)
                    .channel(NioServerSocketChannel.class)
                	//设置用于处理Channel请求的ChannelHandler.
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel socketChannel) throws Exception {
                            //处理Http消息的编码
                            socketChannel.pipeline().addLast("httpServerCodec", new HttpServerCodec());
                            //因为自定义的ChannelHandler中使用到了FullRequest
                            socketChannel.pipeline().addLast("aggregator", new HttpObjectAggregator(65536));
                            //添加自定义的ChannelHandler
                            socketChannel.pipeline().addLast("httpServerHandler", new NettyServerHandler());
                        }
                    });
            ChannelFuture sync = b.bind(8888).sync();
            log.info("NettyServer已启动，访问地址为：127.0.0.1:8888");
            //等待服务端口关闭
            sync.channel().closeFuture().addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture channelFuture) throws Exception {
                    //优雅退出，释放线程池资源
                    bossGroup.shutdownGracefully();
                    workGroup.shutdownGracefully();
                    log.info("退出netty");
                }
            });
        } catch (Exception e) {
            log.error("NettyServer启动失败", e);
        }
    }
}
```

在编写Netty程序时，一般都会创建两个NioEventLoopGroup的实例bossGroup,workGroup. 先简单的这么理解这两个实例,它们实际上是两个线程池，线程的数量默认为CPU核心数*2. 其中bossGroup 用来处理连接，workGroup用来处理请求。

ServerBootstrap用来为 Netty 程序的启动组装配置一些必须要组件，例如上面的创建的两个线程组。channel 方法用于指定服务器端监听套接字通道 NioServerSocketChannel。

.childHandler()方法主要是设置用于处理Channel请求的ChannelHandler，可以简单的理解为当请求了Netty服务之后，将会运行里面的channelhandler。这一块将在后续详细介绍。

接着bind 方法将服务绑定到 8888 端口上，bind 方法内部会执行端口绑定等一系列操做，使得前面的配置都各就各位各司其职，sync 方法用于阻塞当前 Thread，一直到端口绑定操作完成。接下来一句是应用程序将会阻塞等待直到服务器的 Channel 关闭。

把shutdownGracefully函数写在cf.channel().closeFuture().addListener中,这样main线程会主动释放无需在阻塞等待了.并且管道关闭的时候,会自动的进行优雅关闭.

```java
new ChannelInitializer<SocketChannel>() {
    @Override
    protected void initChannel(SocketChannel socketChannel) throws Exception {
        //处理Http消息的编码
        socketChannel.pipeline().addLast("httpServerCodec", new HttpServerCodec());
        //因为自定义的ChannelHandler中使用到了FullRequest
        socketChannel.pipeline().addLast("aggregator", new HttpObjectAggregator(65536));
        //添加自定义的ChannelHandler
        socketChannel.pipeline().addLast("httpServerHandler", new NettyServerHandler());
    }                       
}
```

这段代码主要是为了添加后续处理中需要的编码解码器，以及自定义的channelhandler用来实现自己的业务。

Netty 是一个高性能网络通信框架，同时它也是比较底层的框架，想要 Netty 支持 Http（超文本传输协议），必须要给它提供相应的编解码器。

我们这里使用 Netty 自带的 Http 编解码组件 HttpServerCodec 对通信数据进行编解码，HttpServerCodec 是 HttpRequestDecoder 和 HttpResponseEncoder 的组合，因为在处理 Http 请求时这两个类是经常使用的，所以 Netty 直接将他们合并在一起更加方便使用。

HttpObjectAggregator提供了将请求合并为FullHttpRequest的功能



NettyServerHandler就是我们自己需要实现的类了

代码如下：

```java
/**
 * @Author Song
 * @Date 2020/7/21 15:10
 * @Version 1.0
 * @Description Netty 的设计中把 Http 请求分为了 HttpRequest 和 HttpContent 两个部分
 * HttpRequest 主要包含请求头、请求方法等信息
 * HttpContent 主要包含请求体的信息
 * <p>
 * Netty 又提供了另一个类 FullHttpRequest，FullHttpRequest 包含请求的所有信息，
 * 它是一个接口，直接或者间接继承了 HttpRequest 和 HttpContent，它的实现类是 DefalutFullHttpRequest。
 * 故在此处可以使用FullHttpRequest。
 * 注意使用了FullHttpRequest，需要在ChannelPipeline中加入HttpObjectAggregator 的实例
 * HttpObjectAggregator 作用为将请求转换为单一的 FullHttpReques
 */
@Slf4j
public class NettyServerHandler extends SimpleChannelInboundHandler<FullHttpRequest> {

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, FullHttpRequest request) throws Exception {
        log.info("请求类型:" + request.method().name());
        log.info("uri:" + request.uri());
        ByteBuf buf = request.content();
        String requestContent = buf.toString(CharsetUtil.UTF_8);
        log.info("请求体为:" + requestContent);
        //设置响应内容
        ByteBuf byteBuf = Unpooled.copiedBuffer("hello world" + requestContent, CharsetUtil.UTF_8);
        FullHttpResponse response = new DefaultFullHttpResponse(HttpVersion.HTTP_1_1, HttpResponseStatus.OK, byteBuf);
        //添加响应头
        response.headers().add(HttpHeaderNames.CONTENT_TYPE, "text/plain");
        response.headers().add(HttpHeaderNames.CONTENT_LENGTH, byteBuf.readableBytes());
        //响应客户端
        ctx.writeAndFlush(response);
    }
}
```

至此一个简单的HTTP服务就编写完成了，现在运行main方法，然后用postman测试一下。

![image-20200723175007151](D:\MyStudy\学习杂记\Java\Netty\Netty.assets\image-20200723175007151.png)

![image-20200723175218530](D:\MyStudy\学习杂记\Java\Netty\Netty.assets\image-20200723175218530.png)

程序将会返回我们hello world 加上请求的内容。正确响应

现在就先写这么多啦。请听下回分解！



#### 3.编写一个客户端