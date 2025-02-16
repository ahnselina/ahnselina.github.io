---
title: 基于brpc的stream方式实现双向通信
date: 2022-3-12 23:30:30
categories:
- 知识积累
tags: [brpc, RPC, 通信]
---

![](https://brpc.incubator.apache.org/images/docs/logo.png)

<!-- more -->

本文先简单介绍一下brpc，然后是brpc官方对stream方式使用的介绍，再看brpc官方的stream方式的使用示例。不过brpc官方示例是client通过stream一直向server发消息，而笔者希望能通过stream进行双向通信，所以对示例进行了修改，以实现client和server的通过stream方式的双向通信。

## bRPC简介

bRPC基础介绍。

### 什么是RPC?

互联网上的机器大都通过TCP/IP协议相互访问，但TCP/IP只是往远端发送了一段二进制数据，为了建立服务还有很多问题需要抽象：

* 数据以什么格式传输？不同机器间，网络间可能是不同的字节序，直接传输内存数据显然是不合适的；随着业务变化，数据字段往往要增加或删减，怎么兼容前后不同版本的格式？

* 一个TCP连接可以被多个请求复用以减少开销么？多个请求可以同时发往一个TCP连接么?

* 如何管理和访问很多机器？

* 连接断开时应该干什么？

* 万一server不发送回复怎么办？
  …
  RPC可以解决这些问题，它把网络交互类比为“client访问server上的函数”：client向server发送request后开始等待，直到server收到、处理、回复client后，client又再度恢复并根据response做出反应。
  ![](https://brpc.incubator.apache.org/images/docs/rpc.png)
  我们来看看上面的一些问题是如何解决的：

* 数据需要序列化，[protobuf](https://github.com/protocolbuffers/protobuf)在这方面做的不错。用户填写protobuf::Message类型的request，RPC结束后，从同为protobuf::Message类型的response中取出结果。protobuf有较好的前后兼容性，方便业务调整字段。http广泛使用json作为序列化方法。

* 用户无需关心连接如何建立，但可以选择不同的[连接方式](https://brpc.incubator.apache.org/docs/client/basics/#%E8%BF%9E%E6%8E%A5%E6%96%B9%E5%BC%8F)：短连接，连接池，单连接。

* 大量机器一般通过命名服务被发现，可基于DNS, ZooKeeper, etcd等实现。在百度内，我们使用BNS (Baidu Naming Service)。brpc也提供“list://“和"file://"。用户可以指定负载均衡算法，让RPC每次选出一台机器发送请求，包括: round-robin, randomized, consistent-hashing(murmurhash3 or md5)和 [locality-aware](https://brpc.incubator.apache.org/docs/rpc-in-depth/locality-aware/).

* 连接断开时可以重试。

* 如果server没有在给定时间内回复，client会返回超时错误。

### 哪里可以使用RPC?

几乎所有的网络交互。

RPC不是万能的抽象，否则我们也不需要TCP/IP这一层了。但是在我们绝大部分的网络交互中，RPC既能解决问题，又能隔离更底层的网络问题。

对于RPC常见的质疑有：

* 我的数据非常大，用protobuf序列化太慢了。首先这可能是个伪命题，你得用profiler证明慢了才是真的慢，其次很多协议支持携带二进制数据以绕过序列化。
* 我传输的是流数据，RPC表达不了。事实上brpc中很多协议支持传递流式数据，包括http中的ProgressiveReader, h2的streams, streaming rpc, 和专门的流式协议RTMP。
* 我的场景不需要回复。简单推理可知，你的场景中请求可丢可不丢，可处理也可不处理，因为client总是无法感知，你真的确认这是OK的？即使场景真的不需要，我们仍然建议用最小的结构体回复，因为这不大会是瓶颈，并且追查复杂bug时可能是很有价值的线索。

### 什么是brpc?

百度内最常使用的工业级RPC框架, 有1,000,000+个实例(不包含client)和上千种服务, 在百度内叫做”baidu-rpc”. 目前只开源C++版本。

你可以使用它：

* 搭建能在一个端口支持多协议的服务, 或访问各种服务
* Server能同步或异步处理请求。
* Client支持同步、异步、半同步，或使用组合channels简化复杂的分库或并发访问。
* 通过http界面调试服务, 使用cpu, heap, contention profilers.
* 获得更好的延时和吞吐.
* 把你组织中使用的协议快速地加入brpc，或定制各类组件, 包括命名服务 (dns, zk, etcd), 负载均衡 (rr, random, consistent hashing)

## 流式(stream)RPC概述

在一些应用场景中， client或server需要向对面发送大量数据，这些数据非常大或者持续地在产生以至于无法放在一个RPC的附件中。比如一个分布式系统的不同节点间传递replica或snapshot。client/server之间虽然可以通过多次RPC把数据切分后传输过去，但存在如下问题：

* 如果这些RPC是并行的，无法保证接收端有序地收到数据，拼接数据的逻辑相当复杂。
* 如果这些RPC是串行的，每次传递都得等待一次网络RTT+处理数据的延时，特别是后者的延时可能是难以预估的。
  为了让大块数据以流水线的方式在client/server之间传递， 我们提供了Streaming RPC这种交互模型。Streaming RPC让用户能够在client/service之间建立用户态连接，称为Stream, 同一个TCP连接之上能同时存在多个Stream。 Stream的传输数据以消息为基本单位， 输入端可以源源不断的往Stream中写入消息， 接收端会按输入端写入顺序收到消息。
  Streaming RPC保证：
* 有消息边界。
* 接收消息的顺序和发送消息的顺序严格一致。
* 全双工。
* 支持流控。
* 提供超时提醒
  目前的实现还没有自动切割过大的消息，同一个tcp连接上的多个Stream之间可能有[Head-of-line blocking](https://en.wikipedia.org/wiki/Head-of-line_blocking)问题，请尽量避免过大的单个消息，实现自动切割后我们会告知并更新文档。
  例子见[example/streaming_echo_c++](https://github.com/apache/incubator-brpc/tree/master/example/streaming_echo_c++)。

## 建立Stream

目前Stream都由Client端建立。Client先在本地创建一个Stream，再通过一次RPC（必须使用baidu_std协议）与指定的Service建立一个Stream，如果Service在收到请求之后选择接受这个Stream， 那在response返回Client后Stream就会建立成功。过程中的任何错误都把RPC标记为失败，同时也意味着Stream创建失败。用linux下建立连接的过程打比方，Client先创建socket（创建Stream），再调用connect尝试与远端建立连接（通过RPC建立Stream），远端accept后连接就建立了（service接受后创建成功）。
如果Client尝试向不支持Streaming RPC的老Server建立Stream，将总是失败。
程序中我们用StreamId代表一个Stream，对Stream的读写，关闭操作都将作用在这个Id上。

```cpp
struct StreamOptions
    // The max size of unconsumed data allowed at remote side.
    // If |max_buf_size| <= 0, there's no limit of buf size
    // default: 2097152 (2M)
    int max_buf_size;

    // Notify user when there's no data for at least |idle_timeout_ms|
    // milliseconds since the last time that on_received_messages or on_idle_timeout
    // finished.
    // default: -1
    long idle_timeout_ms;

    // Maximum messages in batch passed to handler->on_received_messages
    // default: 128
    size_t messages_in_batch;

    // Handle input message, if handler is NULL, the remote side is not allowd to
    // write any message, who will get EBADF on writting
    // default: NULL
    StreamInputHandler* handler;
};

// [Called at the client side]
// Create a Stream at client-side along with the |cntl|, which will be connected
// when receiving the response with a Stream from server-side. If |options| is
// NULL, the Stream will be created with default options
// Return 0 on success, -1 otherwise
int StreamCreate(StreamId* request_stream, Controller &cntl, const StreamOptions* options);
```

## 读取Stream

在建立或者接受一个Stream的时候， 用户可以继承StreamInputHandler并把这个handler填入StreamOptions中. 通过这个handler，用户可以处理对端的写入数据，连接关闭以及idle timeout

```cpp
class StreamInputHandler {
public:
    // 当接收到消息后被调用
    virtual int on_received_messages(StreamId id, butil::IOBuf *const messages[], size_t size) = 0;

    // 当Stream上长时间没有数据交互后被调用
    virtual void on_idle_timeout(StreamId id) = 0;

    // 当Stream被关闭时被调用
    virtual void on_closed(StreamId id) = 0;
};
```

> 第一次收到请求的时间
> 在client端，如果建立过程是一次同步RPC， 那在等待的线程被唤醒之后，on_received_message就可能会被调用到。 如果是异步RPC请求， 那等到这次请求的done->Run() 执行完毕之后， on_received_message就可能会被调用。
> 在server端， 当框架传入的done->Run()被调用完之后， on_received_message就可能会被调用。

## 写入Stream

```cpp
// Write |message| into |stream_id|. The remote-side handler will received the
// message by the written order
// Returns 0 on success, errno otherwise
// Errno:
//  - EAGAIN: |stream_id| is created with positive |max_buf_size| and buf size
//            which the remote side hasn't consumed yet excceeds the number.
//  - EINVAL: |stream_id| is invalied or has been closed
int StreamWrite(StreamId stream_id, const butil::IOBuf &message);
```

## 流控

当存在较多已发送但未接收的数据时，发送端的Write操作会立即失败(返回EAGAIN）， 这时候可以通过同步或异步的方式等待对端消费掉数据。

```cpp
// Wait util the pending buffer size is less than |max_buf_size| or error occurs
// Returns 0 on success, errno otherwise
// Errno:
//  - ETIMEDOUT: when |due_time| is not NULL and time expired this
//  - EINVAL: the Stream was close during waiting
int StreamWait(StreamId stream_id, const timespec* due_time);

// Async wait
void StreamWait(StreamId stream_id, const timespec *due_time,
                void (*on_writable)(StreamId stream_id, void* arg, int error_code),
                void *arg);
```

## 关闭Stream

```cpp
// Close |stream_id|, after this function is called:
//  - All the following |StreamWrite| would fail
//  - |StreamWait| wakes up immediately.
//  - Both sides |on_closed| would be notifed after all the pending buffers have
//    been received
// This function could be called multiple times without side-effects
int StreamClose(StreamId stream_id);
```

## 对原始例子的修改

brpc stream使用的官方原始例子为[https://github.com/apache/incubator-brpc/tree/master/example/streaming_echo_c++](https://github.com/apache/incubator-brpc/tree/master/example/streaming_echo_c++)。
不过官方的原始例子只是streaming_echo_client端一直往streaming_echo_server端发stream流消息。
client.cpp代码为：[streaming_echo_client端代码](https://github.com/apache/incubator-brpc/blob/master/example/streaming_echo_c%2B%2B/client.cpp)
server.cpp代码为：[https://github.com/apache/incubator-brpc/blob/master/example/streaming_echo_c%2B%2B/server.cpp](https://github.com/apache/incubator-brpc/blob/master/example/streaming_echo_c%2B%2B/server.cpp)

修改后client.cpp代码为：

```cpp
#include <gflags/gflags.h>
#include <butil/logging.h>
#include <brpc/channel.h>
#include <brpc/stream.h>
#include "echo.pb.h"

DEFINE_bool(send_attachment, true, "Carry attachment along with requests");
DEFINE_string(connection_type, "", "Connection type. Available values: single, pooled, short");
DEFINE_string(server, "0.0.0.0:8001", "IP Address of server");
DEFINE_int32(timeout_ms, 100, "RPC timeout in milliseconds");
DEFINE_int32(max_retry, 3, "Max retries(not including the first RPC)");

class ClientStreamReceiver : public brpc::StreamInputHandler {
public:
    virtual int on_received_messages(brpc::StreamId id,
                                     butil::IOBuf *const messages[],
                                     size_t size) {
        std::ostringstream os;
        for (size_t i = 0; i < size; ++i) {
            os << "msg[" << i << "]=" << *messages[i];
        }
        LOG(INFO) << "Received from Server Stream=" << id << ": " << os.str();
        return 0;
    }
    virtual void on_idle_timeout(brpc::StreamId id) {
        LOG(INFO) << "Server Stream=" << id << " has no data transmission for a while";
    }
    virtual void on_closed(brpc::StreamId id) {
        LOG(INFO) << "Server Stream=" << id << " is closed";
    }

};


int main(int argc, char* argv[]) {
    // Parse gflags. We recommend you to use gflags as well.
    GFLAGS_NS::ParseCommandLineFlags(&argc, &argv, true);

    // A Channel represents a communication line to a Server. Notice that
    // Channel is thread-safe and can be shared by all threads in your program.
    brpc::Channel channel;

    // Initialize the channel, NULL means using default options.
    brpc::ChannelOptions options;
    options.protocol = brpc::PROTOCOL_BAIDU_STD;
    options.connection_type = FLAGS_connection_type;
    options.timeout_ms = FLAGS_timeout_ms/*milliseconds*/;
    options.max_retry = FLAGS_max_retry;
    if (channel.Init(FLAGS_server.c_str(), NULL) != 0) {
        LOG(ERROR) << "Fail to initialize channel";
        return -1;
    }

    // Normally, you should not call a Channel directly, but instead construct
    // a stub Service wrapping it. stub can be shared by all threads as well.
    example::EchoService_Stub stub(&channel);
    brpc::Controller cntl;
    brpc::StreamId stream;
    ClientStreamReceiver _receiver;
    brpc::StreamOptions stream_options;
    stream_options.handler = &_receiver;
    if (brpc::StreamCreate(&stream, cntl, &stream_options) != 0) {
        LOG(ERROR) << "Fail to create stream";
        return -1;
    }
    LOG(INFO) << "Created Stream=" << stream;
    example::EchoRequest request;
    example::EchoResponse response;
    request.set_message("I'm a RPC to connect stream");
    stub.Echo(&cntl, &request, &response, NULL);
    if (cntl.Failed()) {
        LOG(ERROR) << "Fail to connect stream, " << cntl.ErrorText();
        return -1;
    }

    while (!brpc::IsAskedToQuit()) {
        butil::IOBuf msg1;
        msg1.append("abcdefghijklmnopqrstuvwxyz");
        CHECK_EQ(0, brpc::StreamWrite(stream, msg1));
        butil::IOBuf msg2;
        msg2.append("0123456789");
        CHECK_EQ(0, brpc::StreamWrite(stream, msg2));
        sleep(1);
    }

    CHECK_EQ(0, brpc::StreamClose(stream));
    LOG(INFO) << "EchoClient is going to quit";
    return 0;
}
```

修改后server.cpp代码为：

```cpp
#include <gflags/gflags.h>
#include <butil/logging.h>
#include <brpc/server.h>
#include "echo.pb.h"
#include <brpc/stream.h>

DEFINE_bool(send_attachment, true, "Carry attachment along with response");
DEFINE_int32(port, 8001, "TCP Port of this server");
DEFINE_int32(idle_timeout_s, -1, "Connection will be closed if there is no "
             "read/write operations during the last `idle_timeout_s'");
DEFINE_int32(logoff_ms, 2000, "Maximum duration of server's LOGOFF state "
             "(waiting for client to close connection before server stops)");

class StreamReceiver : public brpc::StreamInputHandler {
public:
    virtual int on_received_messages(brpc::StreamId id,
                                     butil::IOBuf *const messages[],
                                     size_t size) {
        std::ostringstream os;
        for (size_t i = 0; i < size; ++i) {
            os << "msg[" << i << "]=" << *messages[i];
        }
        LOG(INFO) << "Received from client Stream=" << id << ": " << os.str();

        butil::IOBuf msg1;
        msg1.append("server ===== send to client by stream....");
        CHECK_EQ(0, brpc::StreamWrite(id, msg1));
        butil::IOBuf msg2;
        msg2.append("server send to client by stream  0123456789");
        CHECK_EQ(0, brpc::StreamWrite(id, msg2));
        //sleep(1);

        return 0;
    }
    virtual void on_idle_timeout(brpc::StreamId id) {
        LOG(INFO) << "Stream=" << id << " has no data transmission for a while";
    }
    virtual void on_closed(brpc::StreamId id) {
        LOG(INFO) << "Stream=" << id << " is closed";
    }

};

// Your implementation of example::EchoService
class StreamingEchoService : public example::EchoService {
public:
    StreamingEchoService() : _sd(brpc::INVALID_STREAM_ID) {};
    virtual ~StreamingEchoService() {
        brpc::StreamClose(_sd);
    };
    virtual void Echo(google::protobuf::RpcController* controller,
                      const example::EchoRequest* /*request*/,
                      example::EchoResponse* response,
                      google::protobuf::Closure* done) {
        // This object helps you to call done->Run() in RAII style. If you need
        // to process the request asynchronously, pass done_guard.release().
        brpc::ClosureGuard done_guard(done);

        brpc::Controller* cntl =
            static_cast<brpc::Controller*>(controller);
        brpc::StreamOptions stream_options;
        stream_options.handler = &_receiver;
        if (brpc::StreamAccept(&_sd, *cntl, &stream_options) != 0) {
            cntl->SetFailed("Fail to accept stream");
            return;
        }
        response->set_message("Accepted stream");
    }

private:
    StreamReceiver _receiver;
    brpc::StreamId _sd;
};

int main(int argc, char* argv[]) {
    // Parse gflags. We recommend you to use gflags as well.
    GFLAGS_NS::ParseCommandLineFlags(&argc, &argv, true);

    // Generally you only need one Server.
    brpc::Server server;

    // Instance of your service.
    StreamingEchoService echo_service_impl;

    // Add the service into server. Notice the second parameter, because the
    // service is put on stack, we don't want server to delete it, otherwise
    // use brpc::SERVER_OWNS_SERVICE.
    if (server.AddService(&echo_service_impl,
                          brpc::SERVER_DOESNT_OWN_SERVICE) != 0) {
        LOG(ERROR) << "Fail to add service";
        return -1;
    }

    // Start the server.
    brpc::ServerOptions options;
    options.idle_timeout_sec = FLAGS_idle_timeout_s;
    if (server.Start(FLAGS_port, &options) != 0) {
        LOG(ERROR) << "Fail to start EchoServer";
        return -1;
    }

    // Wait until Ctrl-C is pressed, then Stop() and Join() the server.
    server.RunUntilAskedToQuit();
    return 0;
}
```

关键的是在**在建立（StreamCreate方法）或者接受（brpc::StreamAccept(&_sd, *cntl, &stream_options)）一个Stream的时候**， 把**继承StreamInputHandler的具体实现的handler填入StreamOptions中**，然后放入到**StreamCreate方法和StreamAccept方法**。
修改后运行streaming_echo_server端打印信息：

![](https://github.com/ahnselina/temp/blob/main/client.png?raw=true)

[如看不见图片，请直接点击](https://github.com/ahnselina/temp/blob/main/client.png)

修改后运行streaming_echo_client端打印信息：
![](https://github.com/ahnselina/temp/blob/main/server.png?raw=true)

[如看不见图片，请直接点击](https://github.com/ahnselina/temp/blob/main/server.png)

通过如上的修改，就达到了client和server端通过stream双向通信的目的 ：)

## 参考资料

[brpc简介](https://brpc.incubator.apache.org/docs/overview/)
[brpc stream 官方资料](https://github.com/apache/incubator-brpc/blob/master/docs/cn/streaming_rpc.md)