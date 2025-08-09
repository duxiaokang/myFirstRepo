### Netty 使用文档（基于主分支 4.2 开发版）

> 说明：本文基于 GitHub 主分支（当前 POM 版本为 4.2.x-SNAPSHOT）仓库结构与示例进行梳理，并补充 4.1 稳定版常见用法的差异提示。生产建议使用 4.1 最新稳定版；如需尝鲜 4.2 API，请留意变更与迁移指南。

- **官网**: [netty.io](https://netty.io/)
- **文档门户**: [Wiki](https://netty.io/wiki/)
- **仓库**: [github.com/netty/netty](https://github.com/netty/netty)
- **示例快速运行脚本**: `run-example.sh`

---

## 1. 核心概念速览

- **Channel**: 抽象的网络连接（客户端 `SocketChannel`、服务端 `ServerSocketChannel`）。
- **EventLoopGroup / EventLoop**: 事件循环线程池与单线程事件循环，负责 I/O 事件多路复用、任务队列执行等。
- **Bootstrap / ServerBootstrap**: 客户端/服务端引导器，配置 `group`、`channel`、`option` 与 `handler`（管道初始化）。
- **ChannelPipeline / ChannelHandler**: 入站/出站事件处理流水线；`Handler` 通过 `@Sharable` 可复用，顺序敏感。
- **Future / Promise**: 异步结果容器，`ChannelFuture` 常用于 `bind/connect/writeAndFlush` 等操作链式监听。
- **ByteBuf**: 高性能缓冲区抽象，支持池化（推荐默认池化分配器），零拷贝组合等。

4.2 与 4.1 的主要感知差异：
- 4.2 示例中使用 `MultiThreadIoEventLoopGroup(NioIoHandler.newFactory())`；4.1 中常见为 `NioEventLoopGroup()`。
- 迁移时，可按版本选择相应 API，思想与配置项基本一致。

---

## 2. 依赖与版本管理

推荐使用 BOM 进行版本对齐：

- Maven（4.1 稳定版示例）
```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>io.netty</groupId>
      <artifactId>netty-bom</artifactId>
      <version>4.1.114.Final</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>

<dependencies>
  <dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-transport</artifactId>
  </dependency>
  <dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-handler</artifactId>
  </dependency>
  <dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-codec-http</artifactId>
  </dependency>
</dependencies>
```

- Gradle（Groovy DSL, 4.1 稳定版示例）
```groovy
dependencies {
  implementation platform('io.netty:netty-bom:4.1.114.Final')
  implementation 'io.netty:netty-transport'
  implementation 'io.netty:netty-handler'
  implementation 'io.netty:netty-codec-http'
}
```

- 关于 `netty-all`：4.1 可用但不建议在大型项目中使用，推荐精确依赖模块；4.2 阶段沿用 BOM 精准依赖。

- OpenSSL/ALPN：HTTP/2 或高性能 TLS 建议加入 `netty-tcnative`（或 `netty-tcnative-boringssl-static`）。

```xml
<dependency>
  <groupId>io.netty</groupId>
  <artifactId>netty-tcnative-boringssl-static</artifactId>
  <version>2.0.65.Final</version>
</dependency>
```

---

## 3. 快速开始：Echo 服务/客户端

以下以 4.2 示例为准，并给出 4.1 对应写法对照。

- 4.2 服务端要点（示例源见 `example/echo/EchoServer.java`）：
```java
EventLoopGroup group = new MultiThreadIoEventLoopGroup(NioIoHandler.newFactory());
ServerBootstrap b = new ServerBootstrap();
b.group(group)
 .channel(NioServerSocketChannel.class)
 .option(ChannelOption.SO_BACKLOG, 100)
 .childHandler(new ChannelInitializer<SocketChannel>() {
     @Override
     protected void initChannel(SocketChannel ch) {
         ChannelPipeline p = ch.pipeline();
         p.addLast(new EchoServerHandler());
     }
});
ChannelFuture f = b.bind(8007).sync();
f.channel().closeFuture().sync();
```

- 4.1 对应写法：
```java
EventLoopGroup boss = new NioEventLoopGroup(1);
EventLoopGroup worker = new NioEventLoopGroup();
try {
  ServerBootstrap b = new ServerBootstrap();
  b.group(boss, worker)
   .channel(NioServerSocketChannel.class)
   .childHandler(new ChannelInitializer<SocketChannel>() {
     @Override protected void initChannel(SocketChannel ch) {
       ch.pipeline().addLast(new EchoServerHandler());
     }
   });
  b.bind(8007).sync().channel().closeFuture().sync();
} finally {
  boss.shutdownGracefully();
  worker.shutdownGracefully();
}
```

- 4.2 客户端要点（示例源见 `example/echo/EchoClient.java`）：
```java
EventLoopGroup group = new MultiThreadIoEventLoopGroup(NioIoHandler.newFactory());
Bootstrap b = new Bootstrap();
b.group(group)
 .channel(NioSocketChannel.class)
 .option(ChannelOption.TCP_NODELAY, true)
 .handler(new ChannelInitializer<SocketChannel>() {
     @Override protected void initChannel(SocketChannel ch) {
         ch.pipeline().addLast(new EchoClientHandler());
     }
});
ChannelFuture f = b.connect("127.0.0.1", 8007).sync();
f.channel().closeFuture().sync();
```

- 4.1 对应写法相同，差异主要在 `EventLoopGroup` 类型名。

要点：
- 所有 I/O 操作返回 `ChannelFuture`，务必在需要处 `.sync()` 或添加监听器处理异常。
- Handler 顺序决定解码/编码与业务执行次序。

---

## 4. HTTP 服务器（HTTP/1.1）

参考 `example/http/helloworld`：

- 启动入口：
```java
ServerBootstrap b = new ServerBootstrap();
b.group(new MultiThreadIoEventLoopGroup(NioIoHandler.newFactory()))
 .channel(NioServerSocketChannel.class)
 .handler(new LoggingHandler(LogLevel.INFO))
 .childHandler(new HttpHelloWorldServerInitializer(sslCtx));
```

- 管道初始化：
```java
p.addLast(new HttpServerCodec());
p.addLast(new HttpContentCompressor((CompressionOptions[]) null));
p.addLast(new HttpServerExpectContinueHandler());
p.addLast(new HttpHelloWorldServerHandler());
```

- 业务处理：
```java
boolean keepAlive = HttpUtil.isKeepAlive(req);
FullHttpResponse resp = new DefaultFullHttpResponse(req.protocolVersion(), OK,
    Unpooled.wrappedBuffer("Hello World".getBytes(StandardCharsets.UTF_8)));
resp.headers().set(CONTENT_TYPE, TEXT_PLAIN)
             .setInt(CONTENT_LENGTH, resp.content().readableBytes());
ChannelFuture f = ctx.write(resp);
if (!keepAlive) { f.addListener(ChannelFutureListener.CLOSE); }
```

提示：
- 压缩建议使用 `HttpContentCompressor`（Brotli 需运行时引入 `brotli4j`）。
- 静态文件可参考 `example/http/file`。

---

## 5. HTTP/2 入门

参考 `example/http2/helloworld`：
- 需要 TLS/ALPN，推荐 `netty-tcnative`，否则退回 JDK provider。
- 服务端核心：
```java
SslContext sslCtx = SslContextBuilder.forServer(kmf)
  .sslProvider(OpenSsl.isAlpnSupported() ? SslProvider.OPENSSL : SslProvider.JDK)
  .ciphers(Http2SecurityUtil.CIPHERS, SupportedCipherSuiteFilter.INSTANCE)
  .applicationProtocolConfig(new ApplicationProtocolConfig(
    Protocol.ALPN,
    SelectorFailureBehavior.NO_ADVERTISE,
    SelectedListenerFailureBehavior.ACCEPT,
    ApplicationProtocolNames.HTTP_2,
    ApplicationProtocolNames.HTTP_1_1))
  .build();
```
- 客户端需信任（示例中 `InsecureTrustManagerFactory` 仅限本地测试）。

---

## 6. 传输与平台优化

- NIO（跨平台）：`io.netty:netty-transport` + `NioServerSocketChannel/NioSocketChannel`。
- Linux
  - `epoll`：`io.netty:netty-transport-native-epoll`（classifier `linux-x86_64` 等），`EpollEventLoopGroup`、`EpollServerSocketChannel`。
  - `io_uring`：`io.netty:netty-transport-native-io_uring`，`IOUringEventLoopGroup` 等（内核 5.10+，需要相应本地依赖）。
- macOS：`kqueue`：`io.netty:netty-transport-native-kqueue`，`KQueueEventLoopGroup` 等。

使用姿势（以 epoll 为例）：
```xml
<dependency>
  <groupId>io.netty</groupId>
  <artifactId>netty-transport-native-epoll</artifactId>
  <classifier>linux-x86_64</classifier>
</dependency>
```
```java
EventLoopGroup group = new EpollEventLoopGroup();
ServerBootstrap b = new ServerBootstrap()
  .group(group)
  .channel(EpollServerSocketChannel.class);
```

---

## 7. 常用编解码器与模块

- `netty-codec-http`：HTTP/1.1，聚合/分块、压缩、CORS、WebSocket（`websocketx`）。
- `netty-codec-http2`：HTTP/2 编解码、升级、流控。
- `netty-handler`：`LoggingHandler`、`SslHandler`、超时、流量整形等。
- `netty-resolver` / `resolver-dns`：异步 DNS 解析。
- 其他：`codec-redis`、`codec-memcache`、`codec-mqtt`、`codec-stomp`、`codec-protobuf`、`codec-dns` 等。

---

## 8. ByteBuf 实战要点

- 默认使用池化分配器，减少 GC：`PooledByteBufAllocator.DEFAULT`。
- 零拷贝：`CompositeByteBuf`、`slice()/duplicate()`、`FileRegion`/`DefaultFileRegion` 发送文件。
- 引用计数与泄漏检测：开启 `-Dio.netty.leakDetection.level=paranoid` 仅限开发/排查。

---

## 9. 背压与可写性

- 写水位线：`ChannelOption.WRITE_BUFFER_WATER_MARK` 控制高/低水位线。
- 监听可写性：覆盖 `channelWritabilityChanged`，在 `!ctx.channel().isWritable()` 时暂停写入。
- 拉模式：`ChannelConfig.setAutoRead(false)` + `ctx.read()` 主动拉取下一批数据。

---

## 10. 性能与最佳实践

- 避免在 I/O 线程里做阻塞/重 CPU 任务；使用业务线程池通过 `ctx.executor().execute()` 或自建 `EventExecutorGroup`。
- 合理批量：`write()` 多次 + `flush()` 一次；仅在必要时 `writeAndFlush()`。
- 充分利用本地传输（`epoll/kqueue/io_uring`）、`tcnative`。
- 绑定 CPU 亲和、`SO_REUSEPORT`（平台相关）、`TCP_NODELAY`/`SO_KEEPALIVE` 等按需调优。

---

## 11. 调试与观测

- `LoggingHandler(LogLevel.DEBUG/TRACE)` 挂到 pipeline 开头定位编解码问题。
- 资源泄漏检测：`-Dio.netty.leakDetection.level=advanced|paranoid`。
- JVM 参数：直接内存大小、压缩 OOPS、GC 日志等按场景配置。

---

## 12. TLS 与证书

- 开发环境可使用自签证书（仓库示例通过 `pkitesting` 构造）。
- 生产建议：OpenSSL（`netty-tcnative`）+ 合规套件。
- SNI/OCSP：可用 `netty-handler-ssl-ocsp` 与相关 API。

---

## 13. 运行仓库自带示例

根目录提供 `run-example.sh`，支持参数透传和示例名选择：

```bash
# 列出可选示例
./run-example.sh

# 运行 Echo 服务
./run-example.sh echo-server

# 运行 Echo 客户端（指定主机与端口）
./run-example.sh -Dhost=127.0.0.1 -Dport=8007 echo-client

# 运行 HTTP HelloWorld（支持 -Dssl -Dport）
./run-example.sh -Dport=8080 http-helloworld-server
```

可选示例涵盖：Echo、Telnet、SecureChat、HTTP/1.1、HTTP/2、WebSocket、Redis/Memcache 客户端、Socks 代理、SCTP/UDT、DNS 等（详见脚本映射表）。

---

## 14. 常见问题（FAQ）

- 是否还应使用 `netty-all`？
  - 小型 Demo 可以；生产建议使用 BOM 精准依赖所需模块。
- 我需要选择哪种传输？
  - 生产优先选本地传输（Linux `epoll`/`io_uring`、macOS `kqueue`）。跨平台/开发环境可用 NIO。
- TLS 性能不足？
  - 尝试 `netty-tcnative-boringssl-static`；同时调整会话复用、ALPN、ciphers。
- HTTP/2 配置为何失败？
  - 检查 ALPN 与 cipher；若 JDK provider 不支持，切换到 `OPENSSL`。

---

## 15. 参考与扩展阅读

- 开发指南: [Developer Guide](https://netty.io/wiki/developer-guide.html)
- 原生传输与依赖: [Native Transports](https://netty.io/wiki/native-transports.html)
- 模块化（JPMS）: 项目内 `testsuite-jpms/README.md`
- 进阶示例: 项目内 `example/` 各模块

---

如需将本指南裁剪为 4.1 稳定版专用，请说明目标 JDK/平台与协议需求（HTTP/1、HTTP/2、WebSocket、自定义协议），我可以输出对应最小可运行模板与依赖清单。