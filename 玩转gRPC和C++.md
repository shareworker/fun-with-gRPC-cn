# 概述

博客源码地址：[fun-with-gRPC](https://github.com/jgaa/fun-with-gRPC/tree/main)

## gRPC 是什么？

什么是 gRPC 呢？在我看来，它是谷歌的一个内部项目，后来以开源形式发布给公众，但缺乏适用于高级用法的完整文档。用 C++ 或 Python（最近我使用 gRPC 的两种语言）创建一个概念验证项目相对简单，但当你深入到“异步请求的深海”时，根本不知道该如何操作。需要明确的是：使用 gRPC 和 C++ 来编写生产级代码是很难的。理解谷歌工程师如何设想生成的服务端接口和客户端桩的使用方式需要时间，正确设置一个用 CMake 构建的项目以生成 C++ 文件也需要时间。如果你需要自己自动构建 protobuf 和 gRPC 库以及工具，那更是要付出大量精力。在本系列博客中，我不会涉及后者，而是使用我所用的 Linux 发行版（Debian 和 Kubuntu）中提供的库和工具。

从 C++ 的角度来看，异步 API 提供了一个坚实的基础，用来实现强大的服务器来处理 gRPC 客户端。谷歌在高性能上优先于类型安全性和 C++ 最佳实践，这可以理解。我假设他们在其庞大的服务器基础设施中使用 gRPC。电力成本昂贵，节省服务器应用中非常关键路径上的指令会带来显著的节约。然而，使用异步接口并不容易。尤其是大多数人仍在使用的传统异步接口。每个 RPC（远程过程调用）都需要大量的样板代码。对于谷歌来说，这种方式是合理的，因为他们有大量程序员，可能也有很多 gRPC 专家。如果他们有内部工具可以生成大部分甚至全部的样板代码，我不会感到惊讶。

## 传统异步接口的优缺点是什么？

**优点**：

- gRPC 在我看来比我熟悉的任何在 C++ 中的HTTP REST解决方案都更快、更稳定。
- 可以将服务器扩展到处理非常大量的每秒请求数。（我还没有探索其在各种消息大小和客户端数量下的每秒请求极限。）
- 传输层被抽象到一个你几乎不会考虑的程度。
- 你可以使用双向流，这在 HTTP REST 中是不可用的。（一些流行服务使用 WebSocket 作为一种替代方案。个人而言，我坚信 WebSocket 是撒旦在心情特别糟糕的一天里创造的。）
- 你无需像使用 JSON 或 XML 那样，为每个对象通过网络发送属性名称的开销。 gRPC 是 protobuf 的扩展——效率更高。
- gRPC 接口易于通过添加新方法和扩展数据类型进行演进。 做得好，可以允许旧应用和新应用共存，这对于高可用性云应用至关重要。
- gRPC 自身提供了出色的性能，非常适合在 C++ 应用中使用。
- 你完全控制线程。如果需要处理大量的同时进行的长时间流式客户端，可以将客户端划分到不同的线程上——例如，每个线程处理几千个客户端。

**缺点：**

- 高效使用较为复杂。这可能对资金充足且经验丰富的大团队无关紧要，但对小团队和独立开发者有影响。
- 不遵循C++最佳实践。这使得编写无bug和安全的代码更加困难。
- 很难找到关于高级用法的文档。
- 每个RPC的实现都需要大量的时间。（如果需要维护 Swagger 文档并正确处理安全性，HTTP REST 请求也会有类似的问题。）
- gRPC 库和生成的代码会带来一些膨胀。 我能够创建的最小服务器 X64 Linux 二进制文件——只实现了一个一元 RPC 调用，使用 g++ 13 编译为发布版并进行剥离，最终大小为 457 KB。最小的客户端二进制文件为 372 KB。这使得 gRPC 在 Kubernetes 之类环境下运行大量微型微服务时不太理想。

## 新的异步回调接口的优缺点

**优点:**

1. 继承了旧异步接口的所有优点（除了线程控制）: 继承了旧异步接口的优势，除了无法直接控制线程。
2. 使用起来更加简单: 比旧的异步接口简单得多，更加友好。
3. 抽象了传输层和执行层: 开发者可以专注于处理实际相关的事件，而不是处理底层细节。
4. 性能处理优化: gRPC 库自动管理多个线程和队列，使项目达到合理的性能，而不需要开发者自己考虑。
5. 项目添加 gRPC 接口的“上市时间”缩短了大约 80%: 项目开发时间大幅缩短。
6. 用户只需理解限制条件: 用户不需要成为 gRPC 专家就可以使用它。

**缺点:**

1. 不支持 C++ 协程: 回调接口看起来和 Java 接口非常相似。
2. 容易出错: 为开发者提供了过多导致资源泄漏、竞争条件和性能问题的机会。正确使用接口仍然很难。
3. 不一致性: 在使用指针的地方，C++ 的最佳实践通常建议使用引用。

### 工具和要求

我将使用 CMake 和 C++ 20。我在 g++ 12 和 13 以及 clang 15 上测试我的代码。除了 protobuf 和 gRPC，我还会使用新版的 boost 库。对于日志记录，我使用一个 header-only 的日志库 logfault，它通过 CMake 自动处理。

### CMake 和 gRPC

CMake 可以处理从 .proto 文件生成代码，这对于在项目中使用 gRPC 是必需的。我发现为 protobuf 和 gRPC 相关的代码创建一个静态库非常有用，然后将其添加到其他库和/或可执行文件的依赖中。示例：

```cmake
project(proto LANGUAGES CXX)

INCLUDE(FindProtobuf)
FIND_PACKAGE(Protobuf REQUIRED)
find_package(gRPC CONFIG REQUIRED)
set(GRPC_LIB gRPC::grpc gRPC::grpc++)

file(GLOB PROTO_FILES "${PROJECT_SOURCE_DIR}/*.proto")
message("   using PROTO_FILES: ${PROTO_FILES}")

add_library(${PROJECT_NAME} STATIC ${PROTO_FILES})
target_link_libraries(${PROJECT_NAME}
    PUBLIC
        $<BUILD_INTERFACE:protobuf::libprotobuf>
        $<BUILD_INTERFACE:${GRPC_LIB}>
)

target_include_directories(proto PUBLIC ${CMAKE_CURRENT_BINARY_DIR})
get_target_property(grpc_cpp_plugin_location gRPC::grpc_cpp_plugin LOCATION)
protobuf_generate(TARGET ${PROJECT_NAME} LANGUAGE cpp)
protobuf_generate(TARGET ${PROJECT_NAME} LANGUAGE grpc GENERATE_EXTENSIONS .grpc.pb.h .grpc.pb.cc PLUGIN "protoc-gen-grpc=${grpc_cpp_plugin_location}")
```

然后，当你创建一个使用 gRPC 的项目时，只需在该项目的 CMakeLists.txt 中提到 proto：

```cmake
add_dependencies(${PROJECT_NAME}
    proto
    ...
    )
```

# 实现一个一元异步服务器

## 异步模型

我的理解是，“异步”模型是最早支持 gRPC 调用异步处理的方法。这适用于服务器端和客户端。

这种异步模型与您可能熟悉的其他“异步”模式完全不同，例如 boost.asio 的异步操作、C++ 20 协程甚至 Node.js 的 futures。

在 gRPC 异步模型中，你有一个队列（Queue）。这个队列有一个 `Next()` 方法，当你准备处理某个操作的结果时——任何操作——这是你要调用的方法。当 `Next()` 返回时，意味着某个异步操作已完成（或失败）。

工作流程如下：你启动一个或多个异步操作。每个操作都直接或间接地获得一个指向你的队列的指针。然后在稍后的某个时间点，你调用队列的 `Next()` 方法来等待任意一个操作完成。当操作完成时，你处理这个操作，例如处理一个服务器接收到的 RPC 请求。如果合适的话，你再启动另一个异步操作，然后你的事件循环会最终回到调用 `Next()`。

换句话说，这个模型要求我们实现一个事件循环，并提供一个线程来运行这个循环。

当你初始化一个异步操作时，你调用一个由 `rpcgen` 代码生成器为你的具体用例实现的接口方法。你的用例是在 `.proto` 文件中描述的。初始化操作告诉 gRPC 你要处理什么操作，以及你想使用哪个“队列”。具体细节取决于操作的方向（服务器端或客户端），以及操作是否涉及流。

当你调用队列的 `Next()` 方法时，如果没有异步操作完成，它将无限期地阻塞。或者，你可以使用 `AsyncNext()`，它允许你设置一个超时时间。当 `Next()` 返回时，你会得到三个状态信息：`Next()` 调用的状态、此操作的布尔状态，以及一个指针（gRPC 称之为 tag）。基于这三种状态，以及你对由该 tag 标识的对象的先前情况的了解，你必须决定如何处理这个事件。正如你所见，这不仅仅是一个事件循环，基本上是一个循环状态机。（或者更糟糕，你可以将其视为一堆没有语言或库支持的原始协程，你需要自己处理所有脏活。）

我在“互联网上”看到的一些示例试图在这个事件循环中处理所有事情。我不建议这样做。由于我们通常需要处理多个 RPC 方法，并且可能有大量同时进行的“会话”或正在进行的请求，我发现将异步接口实现为一个简单的内部循环，并将每个单独的 RPC 操作实现为一个外部状态机会更简单。

现在，让我们退一步看看队列和这三种状态。首先，队列有一些严格的限制。它可以处理大量的项目，但对于具有特定标签的项目，它一次只能排队一个。标签是 `void *` 的值。队列对你的具体用例、标签代表的内容或你定义的 gRPC 方法不感兴趣。它相当原始。而且，请记住——它只允许你为由某个标签标识的任何操作添加一个请求。你可以使用一个对象地址的指针，也可以使用一个唯一的数字，比如 `const auto tag = reinterpret_cast<void *>(1);`。

当你启动一个异步操作时，你需要提供一个标签。操作完成后，你最终会在事件循环中通过 `Queue::Next()` 得到返回的标签。第一个事件的发生取决于操作的类型以及你是在客户端还是服务器端。例如，在服务器端，你可能需要接收一个请求。因此，你可以通过调用由 `rpcgen` 生成的类来处理特定的 RPC 调用，从而添加一个操作。当客户端发送请求时，你会看到第一个事件。如果请求的参数是普通的原型消息，而不是流，你可以立即查看请求并准备一个回复。然后，关于该对象的下一个事件将是回复已发送或发送失败。如果请求是消息流，那么你的下一步操作将是读取第一条消息，而该对象的下一个事件将是第一条消息已发送或出现错误。

当从简单的请求/回复工作流（使用“普通”消息）转向请求流/回复流时，状态机的复杂性增加了。

例如，我的 DNS 服务器将通过 gRPC 处理 RocksDB 数据库中一些列族的复制操作，跟随者将连接到主服务器。主服务器上的所有事务随后会通过双向流传递给它们。跟随者也会将消息流回主服务器，但不是立即发送。每 200 毫秒，如果最后提交的事务 ID 发生了变化，它会发送该 ID。这意味着，当一个跟随者连接到主服务器时，它会接收到大量事务，直到追赶上最新状态。之后，它会随着新事务在主服务器上提交而接收它们。而主服务器在向发起客户端确认变更完成之前，会等待所有最新的跟随者确认该事务。因此，这个架构是完全异步的双向流。在跟随者发出关于其最后提交的事务 ID 的初始消息后（该 ID 可能是 0），两条流之间只存在微弱的关联。每一方在消息可用或计时器到期时推送消息。

## 一个例子

让我们从最简单的用例开始；一个传统的 RPC（远程过程调用）请求，带有一个请求参数和一个单一回复。我们唯一需要处理的复杂性是多个同时到达的请求。因此，创建一个对象来表示单个请求是合理的。我们可以使用该对象实例的指针作为标签。

proto 文件：

```protobuf
service RouteGuide {
  rpc GetFeature(Point) returns (Feature) {}
}

message Point {
  int32 latitude = 1;
  int32 longitude = 2;
}

message Feature {
  string name = 1;
  Point location = 2;
}
```

## 服务器，简单的一元请求/响应消息

我选择使用 C++ 类来实现异步服务器。

初始化非常简单：

```c++
class SimpleReqRespSvc {
public:

    void init() {
        grpc::ServerBuilder builder;

        // Tell gRPC what TCP address/port to listen to and how to handle TLS.
        // grpc::InsecureServerCredentials() will use HTTP 2.0 without encryption.
        builder.AddListeningPort(config_.address, grpc::InsecureServerCredentials());

        // Tell gRPC what rpc methods we support.
        // The code for the class exposed by `service_` is generated from our proto-file.
        builder.RegisterService( service_);

        // Get a queue for our async events
        cq_ = builder.AddCompletionQueue();

        // Finally assemble the server.
        server_ = builder.BuildAndStart();

        LOG_INFO
            // Fancy way to print the class-name
            // Useful when I copy/paste this code around ;)
            << boost::typeindex::type_id_runtime(*this).pretty_name()

            // The useful information
            << " listening on " << config_.address;
    }

private:
     // An instance of our service, compiled from code generated by protoc
    ::routeguide::RouteGuide::AsyncService service_;

    // This is the Queue. It's shared for all the requests.
    std::unique_ptr<grpc::ServerCompletionQueue> cq_;

    // A gRPC server object
    std::unique_ptr<grpc::Server> server_;

    const Config  config_;
}
```

用例输出：

```
async-server
2023-07-17 15:16:56.699 EEST INFO 17607 async-server starting up.
2023-07-17 15:16:56.709 EEST INFO 17608 SimpleReqRespSvc listening on 127.0.0.1:10123
```

事件循环也很简单。它等待队列中的下一个事件可用，并处理超时和关闭。如果有一个标签，我们将其转换为指向请求类实例的指针，并调用它的 `proceed()` 方法（我们很快会讲到这一点）。这将使对象的状态机进入下一个状态。

```c++
    while(true) {
        bool ok = true;
        void *tag = {};

        // FIXME: This is crazy. Figure out how to use stable clock!
        const auto deadline = std::chrono::system_clock::now()
                                + std::chrono::milliseconds(1000);

        // Get any IO operation that is ready.
        const auto status = cq_->AsyncNext( tag,  ok, deadline);

        // So, here we deal with the first of the three states: The status from Next().
        switch(status) {
        case grpc::CompletionQueue::NextStatus::TIMEOUT:
            LOG_DEBUG << "AsyncNext() timed out.";
            continue;

        case grpc::CompletionQueue::NextStatus::GOT_EVENT:
            LOG_DEBUG << "AsyncNext() returned an event. The status is "
                        << (ok ? "OK" : "FAILED");

            // Use a scope to allow a new variable inside a case statement.
            {
                auto request = static_cast<OneRequest *>(tag);

                // Now, let the OneRequest state-machine deal with the event.
                // We could have done it here, but that code would smell really nasty.
                request->proceed(ok);
            }
            break;

        case grpc::CompletionQueue::NextStatus::SHUTDOWN:
            LOG_INFO << "SHUTDOWN. Tearing down the gRPC connection(s) ";
            return;
        } // switch
    } // loop
```

有几件事我不喜欢 `AsyncNext()`。它使用 `void *` 完全抹去了标签的类型。这在 C++ 中是反模式，因为它阻止了编译器和常见工具捕捉与类型相关的错误。我猜他们这样做是为了最大化性能和灵活性。然而，我认为他们应该提供一个类型安全的替代方案。

`Next*()` 使用指针而不是对象引用。这通常是 C++ 中的一种模式，当参数是可选时会使用这种方式。然而，这里显然不是这种情况，至少 `ok` 参数不是。此外，`ok` 参数的含义根据上下文会有所不同，这使得使用它更加困难。使用状态枚举可能会是更好的选择。在客户端中，除了 `ok` 参数之外，还有一个状态类，这使得接口更难理解和正确实现。此外，使用截止时间使用的是未来的绝对时间点，允许 `std::chrono::system_clock::time_point`，但不允许 `std::chrono::steady_clock::time_point`。这太疯狂了！更好的方法是直接接受 `std::chrono::duration` 类型，例如：`std::chrono::milliseconds(123)` 或仅使用表示毫秒的整数。使用系统时钟无法提供准确的超时，因为操作系统会进行时间偏移和时间校正。

但是我们开发者已经学会了利用现有的资源来工作。这也是软件工程有趣的一大部分 😉

我创建了一个 `OneRequest` 类，用于处理我们 RPC 处理程序的简单状态机。

该类在创建时做的第一件事就是调用 `service_.RequestGetFeature()`，其中 `service_` 是由 `protoc` 根据我们的 `.proto` 文件生成的异步服务实例。

```c++
class OneRequest {
public:
    enum class State {
        CREATED,
        REPLIED,
        DONE
    };

    OneRequest(::routeguide::RouteGuide::AsyncService  service,
                ::grpc::ServerCompletionQueue  cq)
        : service_{service}, cq_{cq} {

        // Register this instance with the event-queue and the service.
        // The first event received over the queue is that we have a request.
        service_.RequestGetFeature( ctx_,  req_,  resp_,  cq_,  cq_, this);
    }
    ...
private:
    // We need many variables to handle this one RPC call...
    ::routeguide::RouteGuide::AsyncService  service_;
    ::grpc::ServerCompletionQueue  cq_;
    ::routeguide::Point req_;
    ::grpc::ServerContext ctx_;
    ::routeguide::Feature reply_;
    ::grpc::ServerAsyncResponseWriter<::routeguide::Feature> resp_{ ctx_};
    State state_ = State::CREATED;
```

这个语句让 gRPC 得到了一个可以处理 `GetFeature` 请求的实例。它只会使用一次。因此，在状态机被调用时，我们首先做的事情之一就是创建一个新实例，以便 gRPC 有另一个地方可以调度对这个 RPC 方法的下一个调用。

对于一元 RPC 调用（即方法使用单个 protobuf 消息作为请求，单个 protobuf 消息作为回复），状态机相对简单。

我们在构造函数中通过 `RequestGetFeature` 启动了操作。 当客户端进行 RPC 调用时，我们在状态 `CREATED` 中被调用。 我们启动回复操作并将控制权返回给事件循环。 当回复发送完毕时，我们在状态 `REPLIED` 中再次被调用。 注意，这个状态是我们自己定义的，gRPC 对此一无所知。

```c++
 // State-machine to deal with a single request
    // This works almost like a co-routine, where we work our way down for each
    // time we are called. The State_ could just as well have been an integer/counter;
    void proceed(bool ok) {
        switch(state_) {
        case State::CREATED:
            if (!ok) [[unlikely]] {
                // The operation failed.
                // Let's end it here.
                LOG_WARN << "The request-operation failed. Assuming we are shutting down";
                return done();
            }

            // Before we do anything else, we must create a new instance of
            // OneRequest, so the service can handle a new request from a client.
            createNew(service_, cq_);

            // This is where we have the request, and may formulate an answer.
            // If this was code for a framework, this is where we would have called
            // the `onRpcRequestGetFeature()` method, or unblocked the next statement
            // in a co-routine waiting for the next request.
            //
            // In our case, let's just return something.
            reply_.set_name("whatever");
            reply_.mutable_location()->CopyFrom(req_);

            // Initiate our next async operation.
            // That will complete when we have sent the reply, or replying failed.
            resp_.Finish(reply_, ::grpc::Status::OK, this);

            // This instance is now active.
            state_ = State::REPLIED;
            // Now, we wait for a new event...
            break;

        case State::REPLIED:
            if (!ok) [[unlikely]] {
                // The operation failed.
                LOG_WARN << "The reply-operation failed.";
            }

            state_ = State::DONE; // Not required, but may be useful if we investigate a crash.

            // We are done. There will be no further events for this instance.
            return done();

        default:
            LOG_ERROR << "Logic error / unexpected state in proceed()!";
            assert(false);
        } // switch
    }

    void done() {
        // Ugly, ugly, ugly
        LOG_TRACE << "If the program crash now, it was a bad idea to delete this ;)";
        delete this;
    }
```

# 实现一个一元异步客户端

在服务器中，我们使用了两个线程，一个运行服务器的事件循环，另一个处理信号。我这样做是因为服务器设计为一直运行，直到被停止。另一方面，客户端在完成工作后将退出。对于某些客户端，比如命令行工具，这很合理。你可能会编写一个作为服务器一部分的客户端，需要随时准备处理 RPC。在这种情况下，只要在工作完成后不退出事件循环，就可以了 😉

使用此代码的测试程序有三个输入：服务器地址、要执行的请求总数，以及要运行的并行请求数。

客户端的初始化甚至比服务器更简单：

```c++
class SimpleReqResClient {
public:


     // Run the event-loop.
    // Returns when there are no more requests to send
    void run() {

        LOG_INFO << "Connecting to gRPC service at: " << config_.address;
        channel_ = grpc::CreateChannel(config_.address, grpc::InsecureChannelCredentials());

        stub_ = ::routeguide::RouteGuide::NewStub(channel_);

        ...
    }

private:
    // This is the Queue. It's shared for all the requests.
    ::grpc::CompletionQueue cq_;

    // This is a connection to the gRPC server
    std::shared_ptr<grpc::Channel> channel_;

    // An instance of the client that was generated from our .proto file.
    std::unique_ptr<::routeguide::RouteGuide::Stub> stub_;

    const Config  config_;
    std::atomic_size_t pending_requests_{0};
    std::atomic_size_t request_count{0};
```

由于我们的代码只是用于测试，我们将在进入事件循环之前在 `run()` 中添加一些工作。这些请求将被添加到队列中，并按 gRPC 认为合适的顺序执行。

```c++
// Add request(s)
for(auto i = 0; i < config_.parallel_requests;  ++i) {
    createRequest();
}

        ...
```

事件循环本身与服务器中的事件循环相同，除了 `while()` 条件。

```c++
 while(pending_requests_) {
        // FIXME: This is crazy. Figure out how to use stable clock!
        const auto deadline = std::chrono::system_clock::now()
                                + std::chrono::milliseconds(500);

        // Get any IO operation that is ready.
        void * tag = {};
        bool ok = true;

        // Wait for the next event to complete in the queue
        const auto status = cq_.AsyncNext( tag,  ok, deadline);

        // So, here we deal with the first of the three states: The status of Next().
        switch(status) {
        case grpc::CompletionQueue::NextStatus::TIMEOUT:
            LOG_DEBUG << "AsyncNext() timed out.";
            continue;

        case grpc::CompletionQueue::NextStatus::GOT_EVENT:
            LOG_TRACE << "AsyncNext() returned an event. The boolean status is "
                        << (ok ? "OK" : "FAILED");

            // Use a scope to allow a new variable inside a case statement.
            {
                auto request = static_cast<OneRequest *>(tag);

                // Now, let the OneRequest state-machine deal with the event.
                // We could have done it here, but that code would smell really nasty.
                request->proceed(ok);
            }
            break;

        case grpc::CompletionQueue::NextStatus::SHUTDOWN:
            LOG_INFO << "SHUTDOWN. Tearing down the gRPC connection(s) ";
            return;
        } // switch
```

我们还为每个 RPC 请求创建了一个类。

```c++
class OneRequest {
    public:
        OneRequest(SimpleReqResClient  parent)
                : parent_{parent} {

            // Initiate the async request.
            rpc_ = parent_.stub_->AsyncGetFeature( ctx_, req_,  parent_.cq_);
            assert(rpc_);

            // Add the operation to the queue, so we get notified when
            // the request is completed.
            // Note that we use `this` as tag.
            rpc_->Finish( reply_,  status_, this);

            // Reference-counting of instances of requests in flight
            parent.incCounter();
        }
    ...
private:
    SimpleReqResClient  parent_;

    // We need quite a few variables to perform our single RPC call.
    ::routeguide::Point req_;
    ::routeguide::Feature reply_;
    ::grpc::Status status_;
    std::unique_ptr< ::grpc::ClientAsyncResponseReader< ::routeguide::Feature>> rpc_;
    ::grpc::ClientContext ctx_;
}
```

在构造函数中，我们调用了 `AsyncGetFeature()`，这是由 `rpcgen` 根据我们的 proto 文件为我们生成的桩函数。注意，我们没有在那里添加标签。相反，我们调用返回对象的 `Finish()` 方法，并在其中提供我们的标签。在这种情况下，我们只会收到一个事件：要么从服务器收到成功的回复，要么收到失败的消息。所以，这次的状态机相对简单。别担心，当我们开始处理流时，事情会变得更复杂 😉

`AsyncGetFeature()` 的参数包括指向 `ctx_` 的指针，它是 gRPC 的客户端上下文——这是 C 编程中的常见模式。然后是请求参数，这是我们发送到服务器的请求或消息。请求实例必须在请求被发送到网络上（或更长时间）期间保持存活，因此我们使用类变量 `req_` 来存储它。最后一个参数是指向我们队列的指针。如前所述，我对在 C++ 中使用指针作为必需参数并不是很满意。

注意，我们为 `Finish()` 提供了一个指向 `reply_` 的指针，这是一个 protobuf 消息实例，用于存储此 RPC 请求的返回类型。gRPC 将在其中存储服务器的回复。我们还提供了一个指向 `status_` 的指针，它是 `::grpc::Status` 的实例，是一个封装枚举的类，可以识别一些常见的错误情况。在服务器中，我们在 `Finish()` 调用中提供了一个 `::grpc::Status`。因此，我的理解是，服务器端的代码可以使用此状态向客户端报告一些常见问题。然而，一些可用的错误代码，如 `UNAVAILABLE` 和 `CANCELLED`，表明 gRPC 本身可能会向客户端返回错误状态。所以我不会去猜测 `status_` 错误的来源。我会尽可能妥善处理它们。

这就是 RPC 请求状态机的样子：

```c++
    void proceed(bool ok) {
    if (!ok) [[unlikely]] {
        LOG_WARN << "OneRequest: The request failed.";
        return done();
    }

    // Initiate a new request
    parent_.createRequest();

    if (status_.ok()) {
        LOG_TRACE << "Request successful. Message: " << reply_.name();
    } else {
        LOG_WARN << "OneRequest: The request failed with error-message: " << status_.error_message();
    }

    // The reply is a single message, so at this time we are done.
    done();
}
```

请注意，我们必须处理两个不同的潜在错误状态：`ok` 变量和 `_status`。只有当两者都正常时，我们才能期望回复中包含对我们有用或有效的信息。

# 实现一个带有单条消息和一个流的异步服务器。

我们在前面的部分已经涵盖了一些 RPC 框架所能提供的所有功能。gRPC 的一个很酷的功能是，除了处理请求和回复之外，它还可以处理消息流。在本文中，我们将介绍一个实现了三个 RPC 调用的异步服务器：

- `GetFeature()` 和之前一样。

- `ListFeatures()` 接收一个请求并返回一条消息流。 
- `RecordRoute()` 接收一条消息流，并在流中的最后一条消息收到后返回一个回复。

 因此，在第一次迭代中，新增的内容是如何处理多种请求类型以及单向流。

proto 文件看起来是这样的：

```protobuf
syntax = "proto3";
package routeguide;

// Interface exported by the server.
service RouteGuide {
  rpc GetFeature(Point) returns (Feature) {}
  rpc ListFeatures(Rectangle) returns (stream Feature) {}
  rpc RecordRoute(stream Point) returns (RouteSummary) {}
}

message Point {
  int32 latitude = 1;
  int32 longitude = 2;
}

message Rectangle {
  Point lo = 1;
  Point hi = 2;
}

// A feature names something at a given point.
message Feature {
  string name = 1;
  Point location = 2;
}

message RouteSummary {
  int32 point_count = 1;
  int32 feature_count = 2;
  int32 distance = 3;
  int32 elapsed_time = 4;
}
```

现在，我们首先需要做的是进行一些泛化。我们将尽量保持事件循环的简洁，并尽量避免通过复制粘贴代码来处理各种请求。

为了使用请求实现的 `this` 指针作为标签，并允许事件循环轻松调用 `proceed()`，我们将首先创建一个请求的基类。我还添加了对服务器类的引用，这样我们可以轻松地将信息从命令行传递到请求处理程序的实例。

```c++
    /*! Base class for requests
    *
    *  In order to use `this` as a tag and avoid any special processing in the
    *  event-loop, the simplest approacch in C++ is to let the request implementations
    *  inherit form a base-class that contains the shared code they all need, and
    *  a pure virtual method for the state-machine.
    */
class RequestBase {
public:
    RequestBase(UnaryAndSingleStreamSvc  parent,
                ::routeguide::RouteGuide::AsyncService  service,
                ::grpc::ServerCompletionQueue  cq)
        : parent_{parent}, service_{service}, cq_{cq} {}

    virtual ~RequestBase() = default;

    // The state-machine
    virtual void proceed(bool ok) = 0;

    void done() {
        // Ugly, ugly, ugly
        LOG_TRACE << "If the program crash now, it was a bad idea to delete this ;)";
        delete this;
    }


protected:
    // The state required for all requests
    UnaryAndSingleStreamSvc  parent_;
    ::routeguide::RouteGuide::AsyncService  service_;
    ::grpc::ServerCompletionQueue  cq_;
    ::grpc::ServerContext ctx_;
};
```

如你所见，这非常简单。我们只需要在派生类中实现构造函数和 `proceed()` 方法即可。

## GetFeature

`GetFeature` RPC 调用的实现几乎与我们在第一次迭代中的实现完全相同。

我们从基类派生，声明我们的 `State` 枚举，然后在构造函数中像之前一样进行 gRPC 初始化。

```c++
    /*! Implementation for the `GetFeature()` RPC call.
    */
class GetFeatureRequest : public RequestBase {
public:
    enum class State {
        CREATED,
        REPLIED,
        DONE
    };

    GetFeatureRequest(UnaryAndSingleStreamSvc  parent,
                        ::routeguide::RouteGuide::AsyncService  service,
                        ::grpc::ServerCompletionQueue  cq)
        : RequestBase(parent, service, cq) {

        // Register this instance with the event-queue and the service.
        // The first event received over the queue is that we have a request.
        service_.RequestGetFeature( ctx_,  req_,  resp_,  cq_,  cq_, this);
    }

    ...

private:
    ::routeguide::Point req_;
    ::routeguide::Feature reply_;
    ::grpc::ServerAsyncResponseWriter<::routeguide::Feature> resp_{ ctx_};
    State state_ = State::CREATED;
```

`proceed()` 中唯一显著的变化是调用了创建新实例以处理下一个请求的函数。由于我们有多种类型，我将 `createNew()` 重构为服务器类中的模板方法。我们还在重新使用 `reply_` 变量之前调用了 `Clear()`。这是为了避免在处理新的流消息时意外泄露之前流消息中的信息。

```c++
    ...
    // Before we do anything else, we must create a new instance
    // so the service can handle a new request from a client.
    createNew<ListFeaturesRequest>(parent_, service_, cq_);
    ...

    // Prepare the reply-object to be re-used.
    // This is usually cheaper than creating a new one for each write operation.
    reply_.Clear();
```

我们的新服务器实现类是 `UnaryAndSingleStreamSvc`。它与我们第一个服务器迭代中的 `SimpleReqRespSvc` 类非常相似。事件循环完全相同。

## ListFeatures

让我们来看看 `ListFeatures()` 的请求处理程序 `ListFeaturesRequest` 是如何实现的。

```c++
 /*! Implementation for the `ListFeatures()` RPC call.
     *
     *  This is a bit more advanced. We receive a normal request message,
     *  but the reply is a stream of messages.
     */
    class ListFeaturesRequest : public RequestBase {
    public:
        enum class State {
            CREATED,
            REPLYING,
            FINISHING,
            DONE
        };

        ListFeaturesRequest(UnaryAndSingleStreamSvc  parent,
                            ::routeguide::RouteGuide::AsyncService  service,
                            ::grpc::ServerCompletionQueue  cq)
            : RequestBase(parent, service, cq) {

            // Register this instance with the event-queue and the service.
            // The first event received over the queue is that we have a request.
            service_.RequestListFeatures( ctx_,  req_,  resp_,  cq_,  cq_, this);
        }

        ...
private:
        ::routeguide::Rectangle req_;
        ::routeguide::Feature reply_;
        ::grpc::ServerAsyncWriter<::routeguide::Feature> resp_{ ctx_};
        State state_ = State::CREATED;
        size_t replies_ = 0;
```

正如你可能注意到的，相比于简单的 `GetFeatureRequest` 类，它有一个额外的状态。这是因为它可能会在响应中一次发送多条消息，直到发送完成消息。它还使用了不同类型的回复——`ServerAsyncWriter`，而不是 `ServerAsyncResponseWriter`。这使我们可以通过回复流向客户端发送任意数量的消息。一些服务器，例如“Breaking News”服务，可能永远不会进入 `Finish` 状态。

正如我之前承诺的，流的引入增加了状态机的复杂性。以下是 `ListFeaturesRequest` 的 `proceed()` 实现。

```c++
  // State-machine to deal with a single request
    // This works almost like a co-routine, where we work our way down for each
    // time we are called. The State_ could just as well have been an integer/counter;
    void proceed(bool ok) override {
        switch(state_) {
        case State::CREATED:
            if (!ok) [[unlikely]] {
                // The operation failed.
                // Let's end it here.
                LOG_WARN << "The request-operation failed. Assuming we are shutting down";
                return done();
            }

            // Before we do anything else, we must create a new instance
            // so the service can handle a new request from a client.
            createNew<ListFeaturesRequest>(parent_, service_, cq_);

            state_ = State::REPLYING;
            //fallthrough

        case State::REPLYING:
            if (!ok) [[unlikely]] {
                // The operation failed.
                LOG_WARN << "The reply-operation failed.";
            }

            if (++replies_ > parent_.config_.num_stream_messages) {
                // We have reached the desired number of replies
                state_ = State::FINISHING;

                // *Finish* will relay the event that the write is completed on the queue, using *this* as tag.
                resp_.Finish(::grpc::Status::OK, this);

                // Now, wait for the client to be aware of use finishing.
                break;
            }

            // This is where we have the request, and may formulate another answer.
            // If this was code for a framework, this is where we would have called
            // the `onRpcRequestListFeaturesOnceAgain()` method, or unblocked the next statement
            // in a co-routine awaiting the next state-change.
            //
            // In our case, let's just return something.

            // Prepare the reply-object to be re-used.
            // This is usually cheaper than creating a new one for each write operation.
            reply_.Clear();

            // Since it's a stream, it make sense to return different data for each message.
            reply_.set_name(std::string{"stream-reply #"} + std::to_string(replies_));

            // *Write* will relay the event that the write is completed on the queue, using *this* as tag.
            resp_.Write(reply_, this);

            // Now, we wait for the write to complete
            break;

        case State::FINISHING:
            if (!ok) [[unlikely]] {
                // The operation failed.
                LOG_WARN << "The finish-operation failed.";
            }

            state_ = State::DONE; // Not required, but may be useful if we investigate a crash.

            // We are done. There will be no further events for this instance.
            return done();

        default:
            LOG_ERROR << "Logic error / unexpected state in proceed()!";
        } // switch
    }
```

阅读上面的代码时，请记住，`Write()` 和 `Finish()` 的调用只是启动了一个异步操作。这些方法会立即返回，结果将在完成时作为新事件到达队列。

## RecordRoute

最后，在本文中，我们将看看一个带有输入流和（一个）普通回复的请求的实现。该请求的 proto 定义如下：

```c++
rpc RecordRoute(stream Point) returns (RouteSummary) {}
```

正如你从实现中看到的，它与之前的实现非常相似。我们使用 `::grpc::ServerAsyncReader` 而不是 `::grpc::ServerAsyncWriter`，并调用 `RequestRecordRoute()` 来启动请求流程。

```c++
/*! Implementation for the `RecordRoute()` RPC call.
    *
    *  This is a bit more advanced. We receive a normal request message,
    *  but the reply is a stream of messages.
    */
class RecordRouteRequest : public RequestBase {
public:
    enum class State {
        CREATED,
        READING,
        FINISHING,
        DONE
    };

    RecordRouteRequest(UnaryAndSingleStreamSvc  parent,
                        ::routeguide::RouteGuide::AsyncService  service,
                        ::grpc::ServerCompletionQueue  cq)
        : RequestBase(parent, service, cq) {

        // Register this instance with the event-queue and the service.
        // The first event received over the queue is that we have a request.
        service_.RequestRecordRoute( ctx_,  reader_,  cq_,  cq_, this);
    }

    ...

private:
    ::routeguide::Point req_;
    ::routeguide::RouteSummary reply_;
    ::grpc::ServerAsyncReader< ::routeguide::RouteSummary, ::routeguide::Point> reader_{ ctx_};
    State state_ = State::CREATED;
```

状态机与之前的例子也类似。最大的区别在于我们不知道要读取多少消息，才能确定客户端完成并进行回复。

我们在构造函数中启动了 `RecordRoute` 请求。然后，当客户端调用此方法并且状态机第一次被调用（处于 `CREATED` 状态）时，我们启动了第一次 `Read()` 操作。接下来，基本上每次状态机调用且 `ok == true` 时，我们都会启动一个新的读取操作。当 `ok != true` 时，我们启动 `Finish()` 操作，而下次 `proceed()` 被调用时，我们调用 `done()` 来删除该请求对象的实例。

```c++
 void proceed(bool ok) override {
        switch(state_) {
        case State::CREATED:
            if (!ok) [[unlikely]] {
                // The operation failed.
                // Let's end it here.
                LOG_WARN << "The request-operation failed. Assuming we are shutting down";
                return done();
            }

            // Before we do anything else, we must create a new instance
            // so the service can handle a new request from a client.
            createNew<RecordRouteRequest>(parent_, service_, cq_);

            // Initiate the first read operation
            reader_.Read( req_, this);
            state_ = State::READING;
            break;

        case State::READING:
            if (!ok) [[unlikely]] {
                // The operation failed.
                // This is normal on an incoming stream, when there are no more messages.
                // As far as I know, there is no way at this point to deduce if the false status is
                // because the client is done sending messages, or because we encountered
                // an error.
                LOG_TRACE << "The read-operation failed. It's probably not an error :)";

                // Initiate the finish operation

                // This is where we have received the request, with all it's parts,
                // and may formulate another answer.
                // If this was code for a framework, this is where we would have called
                // the `onRpcRequestRecordRouteDone()` method, or unblocked the next statement
                // in a co-routine awaiting the next state-change.
                //
                // In our case, let's just return something.

                reply_.set_distance(100);
                reply_.set_distance(300);
                reader_.Finish(reply_, ::grpc::Status::OK, this);
                state_ = State::FINISHING;
                break;
            }

            // This is where we have read a message from the request.
            // If this was code for a framework, this is where we would have called
            // the `onRpcRequestRecordRouteGotMessage()` method, or unblocked the next statement
            // in a co-routine awaiting the next state-change.
            //
            // In our case, let's just log it.
            LOG_TRACE << "Got message: longitude=" << req_.longitude()
                        << ", latitude=" << req_.latitude();

            // Prepare the reply-object to be re-used.
            // This is usually cheaper than creating a new one for each read operation.
            req_.Clear();

            // *Read* will relay the event that the write is completed on the queue, using *this* as tag.
            // Initiate the first read operation
            reader_.Read( req_, this);

            // Now, we wait for the read to complete
            break;

        case State::FINISHING:
            if (!ok) [[unlikely]] {
                // The operation failed.
                LOG_WARN << "The finish-operation failed.";
            }

            state_ = State::DONE; // Not required, but may be useful if we investigate a crash.

            // We are done. There will be no further events for this instance.
            return done();

        default:
            LOG_ERROR << "Logic error / unexpected state in proceed()!";
        } // switch
    }

```

正如你注意到的，我们在 `State::READING` 代码块中同时处理了新消息、消息流结束以及 `Finish`。

在这段代码中，我并不关心请求中的数据或我们在回复中发送的内容。重点完全在于为了使用 gRPC 异步接口所需的样板代码。如果你用异步 gRPC 实现某些功能，关注点可能会是正确地实现你的 RPC 请求。这样的话，你的代码复杂度可能会比这个示例大得多。我建议你将自己的业务逻辑完全放在专门的类或模块中，然后从那里调用 gRPC 类，或者从 gRPC 类调用你的接口。即便如此，最终代码的复杂性可能仍会远远超过我在这里展示的内容。

# 实现一个异步客户端，其中包含一个消息和一个流

到目前为止，我们的代码仅处理一个正在进行的异步操作，标签是指向我们的请求类实例的指针。

当我们在客户端处理传入的流时，我们希望保持两个异步请求同时进行。一个用于连接，然后执行下一个从流中读取或向流中写入的操作（根据流的方向），另一个用于从服务器获取最终的回复和状态。这意味着我们至少需要两个不同的标签地址。

我们可以有创造性地使用 `this` 作为其中一个标签，并使用例如 `const auto tag = reinterpret_cast<void *>(reinterpret_cast<uint64_t>(this) + 1);` 作为另一个标签。由于这是从正常的内存分配中返回的，我们可以相当自信地认为它与 4 或 8 字节边界对齐（取决于目标二进制类型，是 32 位还是 64 位）。因此，我们可以查看标签，看看是否需要向下对齐 1 个字节以达到边界地址。这可能有效，但它将是典型的过早优化的例子。

在我看来，更安全的方法是使用一个中间变量，该变量反过来包含对请求对象的指针或引用。

下面是该想法的简化示例:

```c++
    // Request object
    struct Request {

        // intermediate type
        struct Tag {
            Request& request_;

            // Just relay the event to the Request instance
            void proceed() {
                request_.proceed();
            }
        };

        // State-machine for the request
        void proceed();

        // Two tags with different addresses
        Tag first_{*this};
        Tag second_{*this};
    };

    ...
    // Address of first_ is used a the "tag".
    StartAsyncSomething(cq_, &first_);

    // Address of second_ is used a the "tag".
    StartAsyncSomethingElse(cq_, &second_);

    ...

    // event-loop
    while(true) {
        void *tag;
        cq_->Next(&tag);

        // Call proceed() on the intermediate type.
        static_cast<Request::Tag *>(tag)->proceed();
    }

    ...

    // Our queue
    grpc::CompletionQueue cq_;
```

## Request基类

为了处理多个请求类型以及每个请求实例中不止一个标签的复杂性，我们开始通过创建一个请求处理程序的基类来实现客户端。

我们将所有请求类型共享的状态保存在基类中。在我们的例子中，我们还保留了对“父类”的引用，以便可以访问其状态。

最有趣的声明是纯虚方法 `proceed()`。正如你所注意到的，它除了 `ok` 参数外还有另一个参数。由于我们打算让多个异步操作同时进行，而且这些操作的完成顺序可能是随机的（至少对于我来说，获取 `Read()` 和 `Finish()` 的结果顺序看起来是随机的），因此为 `proceed` 增加了一个表示操作类型的参数是合理的。

```c++
 /*! Base class for requests
     *
     *  In order to use `this` as a tag and avoid any special processing in the
     *  event-loop, the simplest approach in C++ is to let the request implementations
     *  inherit form a base-class that contains the shared code they all need, and
     *  a pure virtual method for the state-machine.
     */
    class RequestBase {
    public:

        RequestBase(UnaryAndSingleStreamClient& parent)
            : parent_{parent} {
            LOG_TRACE << "Constructed request #" << client_id_ << " at address" << this;
        }

        virtual ~RequestBase() = default;

        // The state-machine
        virtual void proceed(bool ok, Handle::Operation op) = 0;

    protected:
        // The state required for all requests
        UnaryAndSingleStreamClient& parent_;
        int ref_cnt_ = 0;
        ::grpc::ClientContext ctx_;

    private:
        void done() {
            // Ugly, ugly, ugly
            LOG_TRACE << "If the program crash now, it was a bad idea to delete this ;)  #"
                      << client_id_ << " at address " << this;

            // Reference-counting of instances of requests in flight
            parent_.decCounter();
            delete this;
        }
    };
```

在这个基类中，我们添加了一个 `Handle` 类型来处理唯一的标签。我们发起的所有异步操作最终都会调用 `proceed()` 方法，因此使用引用计数来决定何时删除请求对象是合理的。

我将 `Handle` 作为 `RequestBase` 的一个子类。我把它们视为一个实体。这意味着 `Handle` 可以访问 `RequestBase` 中的受保护和私有变量及方法。这种模式在解决某些问题时非常有效（例如容器及其迭代器）。不过，我不推荐将这种方式作为一种通用方法。在这个场景中，我认为它是高效的。

```c++
 /*! Tag
        *
        *  In order to allow tags for multiple async operations simultaneously,
        *  we use this "Handle". It points to the request owning the
        *  operation, and it is associated with a type of operation.
        */
    class Handle {
    public:
        enum Operation {
            CONNECT,
            READ,
            WRITE,
            WRITE_DONE,
            FINISH
        };

        Handle(RequestBase& instance, Operation op)
            : instance_{instance}, op_{op} {}

        /*! Return a tag for an async operation.
            *
            *  Note that we use this method for reference-counting
            *  the pending async operations, so it cannot be called
            *  for other purposes!
            */
        [[nodiscard]] void *tag() {
            ++instance_.ref_cnt_;
            return this;
        }

        void proceed(bool ok) {
            --instance_.ref_cnt_;

            instance_.proceed(ok, op_);

            if (instance_.ref_cnt_ == 0) {
                instance_.done();
            }
        }

    private:
        RequestBase& instance_;
        const Operation op_;
    };
```

最后，由于我们已经抽象了新的复杂性，客户端类及其事件循环保持简单明了。

为了清晰起见，我删除了一些主要处理新请求实例的代码行。

```c++
 class UnaryAndSingleStreamClient {
    public:


        class RequestBase {
            class Handle {...}
            ...
        }

        UnaryAndSingleStreamClient(const Config& config)
            : config_{config} {}

        // Run the event-loop.
        // Returns when there are no more requests to send
        void run() {

            LOG_INFO << "Connecting to gRPC service at: " << config_.address;
            channel_ = grpc::CreateChannel(config_.address, grpc::InsecureChannelCredentials());

            stub_ = ::routeguide::RouteGuide::NewStub(channel_);

            while(pending_requests_) {
                // FIXME: This is crazy. Figure out how to use stable clock!
                const auto deadline = std::chrono::system_clock::now()
                                    + std::chrono::milliseconds(500);

                // Get any IO operation that is ready.
                void * tag = {};
                bool ok = true;

                // Wait for the next event to complete in the queue
                const auto status = cq_.AsyncNext(&tag, &ok, deadline);

                // So, here we deal with the first of the three states: The status of Next().
                switch(status) {
                case grpc::CompletionQueue::NextStatus::TIMEOUT:
                    LOG_TRACE << "AsyncNext() timed out.";
                    continue;

                case grpc::CompletionQueue::NextStatus::GOT_EVENT:
                    // Use a scope to allow a new variable inside a case statement.
                    {
                        auto handle = static_cast<RequestBase::Handle *>(tag);

                        // Now, let the relevant state-machine deal with the event.
                        // We could have done it here, but that code would smell **really** bad!
                        handle->proceed(ok);
                    }
                    break;

                case grpc::CompletionQueue::NextStatus::SHUTDOWN:
                    LOG_INFO << "SHUTDOWN. Tearing down the gRPC connection(s).";
                    return;
                } // switch
            } // event-loop

            LOG_DEBUG << "exiting event-loop";
            close();
        }

        void incCounter() {
            ++pending_requests_;
        }

        void decCounter() {
            --pending_requests_;
        }

    private:
        // This is the Queue. It's shared for all the requests.
        ::grpc::CompletionQueue cq_;

        // This is a connection to the gRPC server
        std::shared_ptr<grpc::Channel> channel_;

        // An instance of the client that was generated from our .proto file.
        std::unique_ptr<::routeguide::RouteGuide::Stub> stub_;

        size_t pending_requests_{0};
        const Config config_;
    };

```

上述的事件循环可以处理任意数量的请求类型和请求实例。它是完全通用的，只要标签是从 `RequestBase::Handle *` 派生的。

## GetFeature

现在，让我们看看我们熟悉的 `GetFeature` 请求的实现情况。

```c++
    /*! Implementation for the `GetFeature()` RPC request.
     */
    class GetFeatureRequest : public RequestBase {
    public:
        GetFeatureRequest(UnaryAndSingleStreamClient& parent)
            : RequestBase(parent) {

            // Initiate the async request.
            rpc_ = parent_.stub_->AsyncGetFeature(&ctx_, req_, &parent_.cq_);
            assert(rpc_);

            // Add the operation to the queue, so we get notified when
            // the request is completed.
            // Note that we use our handle's this as tag. We don't really need the
            // handle in this unary call, but the server implementation need's
            // to iterate over a Handle to deal with the other request classes.
            rpc_->Finish(&reply_, &status_, handle_.tag());

            // Reference-counting of instances of requests in flight
            parent.incCounter();
        }

        void proceed(bool ok, Handle::Operation /*op */) override {
            if (!ok) [[unlikely]] {
                LOG_WARN << boost::typeindex::type_id_runtime(*this).pretty_name()
                         << " - The request failed. Status: " << status_.error_message();
                return;
            }

            if (status_.ok()) {
                // Initiate a new request
                parent_.nextRequest();
            } else {
                LOG_WARN << boost::typeindex::type_id_runtime(*this).pretty_name()
                         << " - The request failed with error-message: " << status_.error_message();
            }

            // The reply is a single message, so at this time we are done.
        }

    private:
        Handle handle_{*this, Handle::Operation::FINISH};

        // We need quite a few variables to perform our single RPC call.
        ::routeguide::Point req_;
        ::routeguide::Feature reply_;
        ::grpc::Status status_;
        std::unique_ptr< ::grpc::ClientAsyncResponseReader< ::routeguide::Feature>> rpc_;
```

`handle_{*this, Handle::Operation::FINISH}` 变量包含对请求处理程序的引用以及我们唯一需要的操作 `Handle::Operation::FINISH`。请注意，操作枚举是我们自定义的，对 gRPC 没有意义。

由于 `handle` 使用引用计数来删除请求实例，它将始终在 `GetFeatureRequest::proceed()` 返回之后被删除。

## ListFeatures

到目前为止，功能与原始客户端完全相同，尽管我们有了更多的抽象和全新的 `Handle` ;)

当我们使用 gRPC 流时，事情变得更加有趣。

`GetFeature()` 仅发起一个异步操作，而 `ListFeatures()` 接受一个请求作为参数，并允许服务器在完成时发送任意数量的消息作为流式回复，除了最终的 `grpc::Status`。

让我们回顾一下 `ListFeatures()` 的 proto 定义。

```c++
rpc ListFeatures(Rectangle) returns (stream Feature) {}
```

如果流有两个传入消息，则客户端的正常工作流程将经历以下状态：

- connect()
- finish()
- CONNECTED
- read_next()
- HAS_READ
- read_next()
- HAS_READ
- read_next()
- FAILED_READ
- FINISHED

请注意，在这种情况下，`FAILED_READ` 并不是错误。如果协议允许流中有可变数量的消息，通常会为每次成功读取启动一个新的读取操作，直到读取操作失败为止。当完成操作时返回的 `grpc::Status` 可以告诉你是否有实际错误发生。

最后两个事件的顺序——读取失败和完成——似乎是随机的。由于我们使用引用计数来决定请求对象是否处理完所有挂起的请求，因此事件的顺序并不重要。

让我们来看看实现细节。

```c++
 /*! Implementation for the `ListFeatures()` RPC request.
     */
    class ListFeaturesRequest : public RequestBase {
    public:
        // Now we are implementing an actual, trivial state-machine, as
        // we will read an unknown number of messages.

        ListFeaturesRequest(UnaryAndSingleStreamClient& parent)
            : RequestBase(parent) {

            // Initiate the async request.
            // Note that this time, we have to supply the tag to the gRPC initiation method.
            // That's because we will get an event that the request is in progress
            // before we should (can?) start reading the replies.
            rpc_ = parent_.stub_->AsyncListFeatures(&ctx_, req_, &parent_.cq_, connect_handle.tag());

            // Also register a Finish handler, so we know when we are
            // done or failed. This is where we get the server's status when deal with
            // streams.
            rpc_->Finish(&status_, finish_handle.tag());
        }

        ...
    private:
        // We need quite a few variables to perform our single RPC call.

        Handle connect_handle   {*this, Handle::Operation::CONNECT};
        Handle read_handle      {*this, Handle::Operation::READ};
        Handle finish_handle    {*this, Handle::Operation::FINISH};

        ::routeguide::Rectangle req_;
        ::routeguide::Feature reply_;
        ::grpc::Status status_;
        std::unique_ptr< ::grpc::ClientAsyncReader< ::routeguide::Feature>> rpc_;

```

请注意，这个请求的初始化方法 `AsyncListFeatures()` 需要一个标签。这是因为我们将从流中读取数据，而我们不能开始读取，直到我们“连接”完成，即，直到我们在 `proceed()` 方法中看到那个标签，并且 `ok == true`。

除了用来自 `connect_handle` 变量的标签启动连接外，我们还注册了 `Finish()` 的处理程序。这意味着，如果在连接或读取过程中遇到错误，我们应该在处理该标签时在 `status_` 中得到一个值，以便我们可以进行检查。

状态机必须处理我们可能会遇到的三种事件中的任何一种（取决于连接的成功与否）。

```c++
    // As promised, the state-machine gets more complex when we have
        // streams. In this case, we have three states to deal with on each invocation:
        // 1) The state of the instance - how many async operations have we started?
        //    This is handled by reference-counting, so we don't have to deal with it in
        //    the loop. This greatly reduce the code below.
        // 2) The operation
        // 3) The ok boolean value.
        void proceed(bool ok, Handle::Operation op) override {

            switch(op) {

            case Handle::Operation::CONNECT:
                if (!ok) [[unlikely]] {
                    LOG_WARN << me() << " - The request failed.";
                    return;
                }

                // Now, register a read operation.
                rpc_->Read(&reply_, read_handle.tag());
                break;

            case Handle::Operation::READ:
                if (!ok) [[unlikely]] {
                    LOG_TRACE << me() << " - Failed to read a message.";
                    return;
                }

                // This is where we have an actual message from the server.
                // If this was a framework, this is where we would have called
                // `onListFeatureReceivedOneMessage()` or or unblocked the next statement
                // in a co-routine waiting for the next request

                // In our case, let's just log it.
                LOG_TRACE << me() << " - Request successful. Message: " << reply_.name();


                // Prepare the reply-object to be re-used.
                // This is usually cheaper than creating a new one for each read operation.
                reply_.Clear();

                // Now, lets register another read operation
                rpc_->Read(&reply_, read_handle.tag());
                break;

            case Handle::Operation::FINISH:
                if (!ok) [[unlikely]] {
                    LOG_WARN << me() << " - Failed to FINISH! Status: " << status_.error_message();
                    return;
                }

                if (!status_.ok()) {
                    LOG_WARN << me() << " - The request finished with error-message: " << status_.error_message();
                }
                break;

            default:
                LOG_ERROR << me()
                          << " - Unexpected operation in state-machine: "
                          << static_cast<int>(op);

            } // state
        }

        std::string me() const {
            return boost::typeindex::type_id_runtime(*this).pretty_name()
                   + " #" + std::to_string(client_id_);
        }
```

## RecordRoute

在本文中，我们将处理的最后一个请求是 `RecordRoute`。

```c++
    rpc RecordRoute(stream Point) returns (RouteSummary) {}
```

我们先从请求类开始，不考虑其状态机。

```c++
    /*! Implementation for the `RecordRoute()` RPC request.
     */
    class RecordRouteRequest : public RequestBase {
    public:
        // Now we are implementing an actual, trivial state-machine, as
        // we will send a fixed number of messages.

        RecordRouteRequest(UnaryAndSingleStreamClient& parent)
            : RequestBase(parent) {

            // Initiate the async request.
            // Note that this time, we have to supply the tag to the gRPC initiation method.
            // That's because we will get an event that the request is in progress
            // before we should (can?) start writing the requests.
            rpc_ = parent_.stub_->AsyncRecordRoute(&ctx_, &reply_, &parent_.cq_, connect_handle.tag());

            // Initiate a `Finish()` operation so we get the reply-message and a status from the server.
            rpc_->Finish(&status_, finish_handle.tag());
        }

        ...

    private:
        // We need quite a few variables to perform our single RPC call.
        size_t sent_messages_ = 0;

        Handle connect_handle   {*this, Handle::Operation::CONNECT};
        Handle write_handle     {*this, Handle::Operation::WRITE};
        Handle write_done_handle{*this, Handle::Operation::WRITE_DONE};
        Handle finish_handle    {*this, Handle::Operation::FINISH};

        ::routeguide::Point req_;
        ::routeguide::RouteSummary reply_;
        ::grpc::Status status_;
        std::unique_ptr< ::grpc::ClientAsyncWriter< ::routeguide::Point>> rpc_;
    };
```

首先你可能会注意到，我们现在有了四个状态句柄。任何时候我们只会使用其中的两个。

和之前一样，我们在构造函数中启动了两个操作：用正确的请求连接到服务器，并告诉 gRPC 在库接收到服务器的最终状态更新时调用我们的 finish 标签。

我们将 `reply_` 缓冲区提供给 `AsyncRecordRoute()`，并将 `status_` 缓冲区提供给 `Finish()`。我不确定 `reply_` 缓冲区何时被填充。我的假设是，直到我们处理 finish 事件之前，它都不安全使用。

状态机到目前为止是最复杂的。

```c++
void proceed(bool ok, Handle::Operation op) override {

        switch(op) {

        case Handle::Operation::CONNECT:
            if (!ok) [[unlikely]] {
                LOG_WARN << me() << " - The request failed.";
                break;
            }

            // We are ready to send the first message to the server.
            // If this was a framework, this is where we would have called
            // `onRecordRouteReadyToSendFirst()` or or unblocked the next statement
            // in a co-routine waiting for the next state
            req_.set_latitude(50);
            req_.set_longitude(sent_messages_);
            rpc_->Write(req_, write_handle.tag());
            break;

        case Handle::Operation::WRITE:
            if (!ok) [[unlikely]] {
                LOG_TRACE << me() << " - Failed to write a message.";
                break;
            }

            // This is where we have sent an actual message to the server.
            // If this was a framework, this is where we would have called
            // `onRecordRouteReadyToSendNext()` or or unblocked the next statement
            // in a co-routine waiting for the next state

            if (++sent_messages_ >= parent_.config_.num_stream_messages) {
                LOG_TRACE << me() << " - We are done sending messages.";
                rpc_->WritesDone(write_done_handle.tag());

                // Now we have two pending requests, write done and finish.
                break;
            }

            // Prepare the message-object to be re-used.
            // This is usually cheaper than creating a new one for each read operation.
            req_.Clear();

            req_.set_latitude(100);
            req_.set_longitude(sent_messages_);

            // Now, lets register another write operation
            rpc_->Write(req_, write_handle.tag());
            break;

        case Handle::Operation::WRITE_DONE:
            if (!ok) [[unlikely]] {
                LOG_WARN << me() << " - Failed to notify the server that we are done.";
            }
            break;

        case Handle::Operation::FINISH:
            if (!ok) [[unlikely]] {
                LOG_WARN << me() << " - Failed to FINISH! Status: " << status_.error_message();
                break;
            }

            // This is where we have sent all the message to the server.
            // If this was a framework, this is where we would have called
            // `onRecordRouteGotReply()` or or unblocked the next statement
            // in a co-routine waiting for the next state

            if (!status_.ok()) {
                LOG_WARN << me() << " - The request finished with error-message: " << status_.error_message();
            }
            break;

        default:
            LOG_ERROR << me()
                        << " - Unexpected operation in state-machine: "
                        << static_cast<int>(op);

            assert(false);

        } // state
    }

    std::string me() const {
        return boost::typeindex::type_id_runtime(*this).pretty_name()
                + " #" + std::to_string(client_id_);
    }
```

额外的状态来自于在没有消息可发送时需要调用 `WritesDone()`。这会启动另一个异步操作。到那时，我们将收到一个 `WRITE_DONE` 事件和一个 `FINISH` 事件。参考计数将为 0，实例将被删除。

# 实现完整的 RouteGuide 异步服务器

从之前的文章中学到的一个教训是，处理 RPC 请求的代码重复很多。为了减少重复，我们将从这次迭代开始创建一个模板化的基类。我们意识到服务器和客户端中的事件循环是相同的。那么为什么不使用相同的代码呢？

当然，这里面有一些复杂性。服务器代码使用的变量与客户端不同，而且一切都是基于命令行工具 `protoc` 生成的代码，基于我们独特的 proto 文件。

为了解决这个问题，“一切”基类将包含两个部分：

1. 一个模板类，包含事件循环、请求的基类以及更成熟的处理程序。其目的是将实际请求的样板代码减少到最小，同时保持请求实现简单易懂。
2. 一个针对服务器和客户端不同的模板类，但提供一个统一的接口给实际实现。这个类还接受一个模板参数，用于实现我们的 proto 接口的生成代码。

简化来看，它大致是这样的：

```c++
 template <class grpcT>
    struct ServerVars {
    };

    template <class grpcT>
    struct ClientVars {
    };

    template <typename T>
    struct EventLoopBase {

        struct RequestBase {

            struct Handle {
                void proceed(bool ok); // state-machine
            }; // Handle
        }; // RequestBase

        void run(); // event-loop
    }; // EventLoopBase

    ...

    // Instatiate and run a server
    EventLoopBase<ServerVars<::routeguide::RouteGuide>> server;
    server.run();

    // Instatiate and run a client
    EventLoopBase<ClientVars<::routeguide::RouteGuide>> client;
    client.run();
```

这给了我们一个机会，可以仔细编写和测试大量本来会被复制粘贴的代码。如果后续发现了错误，我们可以在一个地方进行修复。如果需要优化代码，也可以在一个地方进行。总的来说，这比大量几乎重复的代码要好得多 ;)

## EventLoopBase的实现

让我们从 `EventLoopBase` 类的轮廓开始。

```c++
 template <typename T>
    class EventLoopBase {
    public:

    EventLoopBase(const Config& config)
        : config_{config} {}

    template <typename reqT, typename parenT>
    void createNew(parenT& parent) {

        // Use make_uniqe, so we destroy the object if it throws an exception
        // (for example out of memory).
        try {
            auto instance = std::make_unique<reqT>(parent);

            // If we got here, the instance should be fine, so let it handle itself.
            instance.release();
        } catch(const std::exception& ex) {
            LOG_ERROR << "Got exception while creating a new instance. "
                      << "This may end my ability to handle any further requests. "
                      << " Error: " << ex.what();
        }
    }

    void stop() {
        grpc_.stop();
    }

    auto& grpc() {
        return grpc_;
    }

    auto * cq() noexcept {
        return grpc_.cq();
    }

    const auto& config() noexcept {
        return config_;
    }

protected:
    const Config& config_;
    size_t num_open_requests_ = 0;
    T grpc_;
```

请注意，我们有一个工厂方法 `createNew` 用于创建我们各种请求的新实例。我们还将一些方法转发给模板属性 `grpc_`。这使我们可以灵活地实现 `T`，只要它提供预期的 `stop()` 和 `cq()` 方法。在服务器中，实际的队列类型是 `std::unique_ptr<grpc::ServerCompletionQueue>`，这是 gRPC 所要求的。在客户端，我们使用的类型是 `::grpc::CompletionQueue`。通过在 `ServerVars` 和 `ClientVars` 中添加 `auto * cq()` 方法，这个实现细节对我们其余的代码是隐藏的。此外，使用 `auto * cq()` 会在服务器实例化时返回 `ServerCompletionQueue *`，在客户端实例化时返回 `CompletionQueue *`，这正是我们所需的。

事件循环在服务器和客户端中都使用。

```c++
/*! Runs the event-loop
    *
    *  The method returns when/if the loop is finished
    *
    *  createNew<T> must have been called on all request-types that are used
    *  prior to this call.
    */
void run() {
        // The inner event-loop
    while(num_open_requests_) {
        // The inner event-loop

        bool ok = true;
        void *tag = {};

        // FIXME: This is crazy. Figure out how to use stable clock!
        const auto deadline = std::chrono::system_clock::now()
                                + std::chrono::milliseconds(1000);

        // Get any IO operation that is ready.
        const auto status = cq()->AsyncNext(&tag, &ok, deadline);

        // So, here we deal with the first of the three states: The status from Next().
        switch(status) {
        case grpc::CompletionQueue::NextStatus::TIMEOUT:
            LOG_TRACE << "AsyncNext() timed out.";
            continue;

        case grpc::CompletionQueue::NextStatus::GOT_EVENT:

            {
                auto request = static_cast<typename RequestBase::Handle *>(tag);

                // Now, let the RequestBase::Handle state-machine deal with the event.
                request->proceed(ok);
            }
            break;

        case grpc::CompletionQueue::NextStatus::SHUTDOWN:
            LOG_INFO << "SHUTDOWN. Tearing down the gRPC connection(s) ";
            return;
        } // switch
    } // loop
}
```

在服务器实现中，我们在开始处理新的 RPC 请求时，总是会添加一个新的请求处理程序。因此，对于服务器来说，`num_open_requests_` 将总是大于或等于我们实现的 RPC 请求类型的数量。在客户端中，我们需要预加载我们想要开始的请求，然后根据需要添加更多请求。当所有请求完成后，`run()` 将返回。如果我们想在服务器应用程序中使用客户端，例如一个使用 gRPC 定期获取数据的 Web 服务或缓存，我们可以按需启动一个新的客户端，将 `while(num_open_requests_)` 更改为 `while(true)`，或者在启动之前将 `num_open_requests_` 增加 1（就像将“工作”添加到 asio 的 `io_context` 中一样）。

我们的 `RequestBase` 是服务器端和客户端请求的基类。

```c++
class RequestBase {
public:
    RequestBase(EventLoopBase& owner)
        : owner_{owner} {
        ++owner.num_open_requests_;
        LOG_TRACE << "Constructed request #" << client_id_ << " at address" << this;
    }

    virtual ~RequestBase() {
        --owner_.num_open_requests_;
    }

    template <typename reqT>
    static std::string me(reqT& req) {
        return boost::typeindex::type_id_runtime(req).pretty_name()
                + " #"
                + std::to_string(req.client_id_);
    }

protected:
    // The state required for all requests
    EventLoopBase& owner_;
    int ref_cnt_ = 0;
    const size_t client_id_ = getNewClientId();

private:
    void done() {
        // Ugly, ugly, ugly
        LOG_TRACE << "If the program crash now, it was a bad idea to delete this ;)  #"
                    << client_id_ << " at address " << this;
        delete this;
    }

}; // RequestBase;

```

到目前为止，代码还算简单。最有趣的方法可能是 `me()`，它是我们之前用来返回请求的类名和请求 ID 的方法的通用实现。这在日志事件中非常有用。

现在，让我们检查一下新的 `Handle` 实现。这次有点不同。

```c++
class RequestBase {
public:
    class Handle
    {
    public:
        // In this implementation, the operation is informative.
        // It has no side-effects.
        enum Operation {
            INVALID,
            CONNECT,
            READ,
            WRITE,
            WRITE_DONE,
            FINISH
        };

        using proceed_t = std::function<void(bool ok, Operation op)>;

        Handle(RequestBase& instance)
            : base_{instance} {}


        [[nodiscard]] void *tag(Operation op, proceed_t&& fn) noexcept {
            assert(op_ == Operation::INVALID);

            LOG_TRACE << "Handle::proceed() - " << base_.client_id_
                        << " initiating " << name(op) << " operation.";
            op_ = op;
            proceed_ = std::move(fn);
            return tag_();
        }

        void proceed(bool ok) {
            --base_.ref_cnt_;

            // We must reset the `op_` type before we call `proceed_`
            // See the comment below regarding `proceed()`.
            const auto current_op = op_;
            op_ = Operation::INVALID;

            if (proceed_) {
                // Move `proceed` to the stack.
                // There is a good probability that `proceed()` will call `tag()`,
                // which will overwrite the current value in the Handle's instance.
                auto proceed = std::move(proceed_);
                proceed(ok, current_op);
            }

            if (base_.ref_cnt_ == 0) {
                base_.done();
            }
        }

    private:
        [[nodiscard]] void *tag_() noexcept {
            ++base_.ref_cnt_;
            return this;
        }

        RequestBase& base_;
        Operation op_ = Operation::INVALID;
        proceed_t proceed_;
    };
}
```

有两个主要的变化。首先，我们使用了一个函数对象来直接实现状态机，以响应事件。其次，我们在适当的情况下重用处理程序。通常我们只需要一到两个处理程序，因此在请求实现中有多达四个处理程序并没有太大意义。我们仍然保留 `Operation` 枚举，但现在它只是用于向日志事件提供准确的信息——或者在调试器中获取一些有用的元信息。`tag()` 接受一个函数对象，通常是一个 lambda 表达式，当异步操作完成时执行。

最后，让我们简要看看 `ServerVars` 和 `ClientVars` 的实现。

```c++
template <typename grpcT>
struct ServerVars {
    // An instance of our service, compiled from code generated by protoc
    typename grpcT::AsyncService service_;

    // This is the Queue. It's shared for all the requests.
    std::unique_ptr<grpc::ServerCompletionQueue> cq_;

    [[nodiscard]] auto * cq() noexcept {
        assert(cq_);
        return cq_.get();
    }

    // A gRPC server object
    std::unique_ptr<grpc::Server> server_;

    void stop() {
        LOG_INFO << "Shutting down ";
        server_->Shutdown();
        server_->Wait();
    }
};

template <typename grpcT>
struct ClientVars {
    // This is the Queue. It's shared for all the requests.
    ::grpc::CompletionQueue cq_;

    [[nodiscard]] auto * cq() noexcept {
        return &cq_;
    }

    // This is a connection to the gRPC server
    std::shared_ptr<grpc::Channel> channel_;

    // An instance of the client that was generated from our .proto file.
    std::unique_ptr<typename grpcT::Stub> stub_;

    void stop() {
        // We don't stop the client...
        assert(false);
    }
};
```

这就是对样板代码的泛化。

## 最终的异步服务器实现

现在，让我们查看 Google 的 "routeguide" proto 文件示例中所有四个 RPC 调用的实际服务器实现。

```c++
service RouteGuide {
  rpc GetFeature(Point) returns (Feature) {}
  rpc ListFeatures(Rectangle) returns (stream Feature) {}
  rpc RecordRoute(stream Point) returns (RouteSummary) {}
  rpc RouteChat(stream RouteNote) returns (stream RouteNote) {}
}
```

在我们过于关注四个请求的实现细节之前，让我们简要看一下外部事件循环类的实现。

```c++
class EverythingSvr
    : public EventLoopBase<ServerVars<::routeguide::RouteGuide>> {
public:

    EverythingSvr(const Config& config)
        : EventLoopBase(config) {

        grpc::ServerBuilder builder;
        builder.AddListeningPort(config_.address, grpc::InsecureServerCredentials());
        builder.RegisterService(&grpc_.service_);
        grpc_.cq_ = builder.AddCompletionQueue();
        // Finally assemble the server.
        grpc_.server_ = builder.BuildAndStart();

        LOG_INFO
            // Fancy way to print the class-name.
            // Useful when I copy/paste this code around ;)
            << boost::typeindex::type_id_runtime(*this).pretty_name()

            // The useful information
            << " listening on " << config_.address;

        // Prepare the first instances of request handlers
        createNew<GetFeatureRequest>(*this);
        createNew<ListFeaturesRequest>(*this);
        createNew<RecordRouteRequest>(*this);
        createNew<RouteChatRequest>(*this);
    }
};
```

如同在服务器中一样，我们使用构建器来设置所需的变量并启动 gRPC 服务器。在最低层，这实际上是一个 HTTP/2 服务器，监听我们指定的主机地址/IP 和端口号。

然后，我们创建每种请求类型的一个实例，用于处理 proto 文件中的四个 RPC 请求。这些实例将用于处理每种类型的第一个传入 RPC 调用。在下面的实现中，我们将根据需要创建新的实例，以保持 gRPC 服务的响应。如果我们忘记这一点，客户端将会看到一个正常的客户端调用 "connect"——但 Finish 事件将永远不会被触发。

## GetFeature

```c++
class GetFeatureRequest : public RequestBase {
public:

    GetFeatureRequest(EverythingSvr& owner)
        : RequestBase(owner) {

        // Register this instance with the event-queue and the service.
        // The first event received over the queue is that we have a request.
        owner_.grpc().service_.RequestGetFeature(&ctx_, &req_, &resp_, cq(), cq(),
            op_handle_.tag(Handle::Operation::CONNECT,
            [this, &owner](bool ok, Handle::Operation /* op */) {

            LOG_DEBUG << me(*this) << " - Processing a new connect from " << ctx_.peer();

                if (!ok) [[unlikely]] {
                    // The operation failed.
                    // Let's end it here.
                    LOG_WARN << "The request-operation failed. Assuming we are shutting down";
                    return;
                }

                // Before we do anything else, we must create a new instance of
                // GetFeatureRequest, so the service can handle a new request from a client.
                owner_.createNew<GetFeatureRequest>(owner);

                // This is where we have the request, and may formulate an answer.
                // If this was code for a framework, this is where we would have called
                // the `onRpcRequestGetFeature()` method, or unblocked the next statement
                // in a co-routine waiting for the next request.
                //
                // In our case, let's just return something.
                reply_.set_name("whatever");
                reply_.mutable_location()->CopyFrom(req_);

                // Initiate our next async operation.
                // That will complete when we have sent the reply, or replying failed.
                resp_.Finish(reply_, ::grpc::Status::OK,
                    op_handle_.tag(Handle::Operation::FINISH,
                    [this](bool ok, Handle::Operation /* op */) {

                        if (!ok) [[unlikely]] {
                            LOG_WARN << "The finish-operation failed.";
                        }

                }));// FINISH operation lambda
            })); // CONNECT operation lambda
    }

private:
    Handle op_handle_{*this}; // We need only one handle for this operation.

    ::grpc::ServerContext ctx_;
    ::routeguide::Point req_;
    ::routeguide::Feature reply_;
    ::grpc::ServerAsyncResponseWriter<decltype(reply_)> resp_{&ctx_};
};
```

`GetFeatureRequest` 从来不算复杂。我们所做的就是移除了当前操作上的 `proceed()` 重写语句，而是将一个 lambda 表达式作为参数传递给处理程序的 `tag()` 方法。我个人觉得这段代码更易于阅读。虽然它仍不是协程，但逻辑流的呈现方式使其简单易读和理解。

## ListFeaturesRequest

请记住，在 `ListFeatures` 中我们返回的是一系列消息。在这里，我们使用 lambda 表达式来处理特定操作唯一的逻辑，并使用 `reply()` 来处理连接事件和回复事件的共享回复逻辑。

```c++
class ListFeaturesRequest : public RequestBase {
public:

    ListFeaturesRequest(EverythingSvr& owner)
        : RequestBase(owner) {

        owner_.grpc().service_.RequestListFeatures(&ctx_, &req_, &resp_, cq(), cq(),
            op_handle_.tag(Handle::Operation::CONNECT,
            [this, &owner](bool ok, Handle::Operation /* op */) {

                LOG_DEBUG << me(*this) << " - Processing a new connect from " << ctx_.peer();

                if (!ok) [[unlikely]] {
                    // The operation failed.
                    // Let's end it here.
                    LOG_WARN << "The request-operation failed. Assuming we are shutting down";
                    return;
                }

                // Before we do anything else, we must create a new instance
                // so the service can handle a new request from a client.
                owner_.createNew<ListFeaturesRequest>(owner);

            reply();
        }));
    }

private:
    void reply() {
        if (++replies_ > owner_.config().num_stream_messages) {
            // We have reached the desired number of replies

            resp_.Finish(::grpc::Status::OK,
                op_handle_.tag(Handle::Operation::FINISH,
                [this](bool ok, Handle::Operation /* op */) {
                    if (!ok) [[unlikely]] {
                        // The operation failed.
                        LOG_WARN << "The finish-operation failed.";
                    }
            }));

            return;
        }

        // This is where we have the request, and may formulate another answer.
        // If this was code for a framework, this is where we would have called
        // the `onRpcRequestListFeaturesOnceAgain()` method, or unblocked the next statement
        // in a co-routine awaiting the next state-change.
        //
        // In our case, let's just return something.


        // Prepare the reply-object to be re-used.
        // This is usually cheaper than creating a new one for each write operation.
        reply_.Clear();

        // Since it's a stream, it make sense to return different data for each message.
        reply_.set_name(std::string{"stream-reply #"} + std::to_string(replies_));

        resp_.Write(reply_, op_handle_.tag(Handle::Operation::FINISH,
            [this](bool ok, Handle::Operation /* op */) {
                if (!ok) [[unlikely]] {
                    // The operation failed.
                    LOG_WARN << "The reply-operation failed.";
                    return;
                }

                reply();
        }));
    }

    Handle op_handle_{*this}; // We need only one handle for this operation.
    size_t replies_ = 0;

    ::grpc::ServerContext ctx_;
    ::routeguide::Rectangle req_;
    ::routeguide::Feature reply_;
    ::grpc::ServerAsyncWriter<decltype(reply_)> resp_{&ctx_};
};
```

这比之前的稍微复杂一点，但如果去掉注释，代码行数并不多。

## RecordRouteRequest

现在我们将获得一系列消息，并在流结束时进行回复。

```c++
class RecordRouteRequest : public RequestBase {
public:

    RecordRouteRequest(EverythingSvr& owner)
        : RequestBase(owner) {

        owner_.grpc().service_.RequestRecordRoute(&ctx_, &io_, cq(), cq(),
            op_handle_.tag(Handle::Operation::CONNECT,
                [this, &owner](bool ok, Handle::Operation /* op */) {

                    LOG_DEBUG << me(*this) << " - Processing a new connect from " << ctx_.peer();

                    if (!ok) [[unlikely]] {
                        // The operation failed.
                        // Let's end it here.
                        LOG_WARN << "The request-operation failed. Assuming we are shutting down";
                        return;
                    }

                    // Before we do anything else, we must create a new instance
                    // so the service can handle a new request from a client.
                    owner_.createNew<RecordRouteRequest>(owner);

                    read(true);
                }));
    }

private:
    void read(const bool first) {

        if (!first) {
            // This is where we have read a message from the request.
            // If this was code for a framework, this is where we would have called
            // the `onRpcRequestRecordRouteGotMessage()` method, or unblocked the next statement
            // in a co-routine awaiting the next state-change.
            //
            // In our case, let's just log it.
            LOG_TRACE << "Got message: longitude=" << req_.longitude()
                        << ", latitude=" << req_.latitude();


            // Reset the req_ message. This is cheaper than allocating a new one for each read.
            req_.Clear();
        }

        io_.Read(&req_,  op_handle_.tag(Handle::Operation::READ,
            [this](bool ok, Handle::Operation /* op */) {
                if (!ok) [[unlikely]] {
                    // The operation failed.
                    // This is normal on an incoming stream, when there are no more messages.
                    // As far as I know, there is no way at this point to deduce if the false status is
                    // because the client is done sending messages, or because we encountered
                    // an error.
                    LOG_TRACE << "The read-operation failed. It's probably not an error :)";

                    // Initiate the finish operation

                    // This is where we have received the request, with all it's parts,
                    // and may formulate another answer.
                    // If this was code for a framework, this is where we would have called
                    // the `onRpcRequestRecordRouteDone()` method, or unblocked the next statement
                    // in a co-routine awaiting the next state-change.
                    //
                    // In our case, let's just return something.

                    reply_.set_distance(100);
                    reply_.set_distance(300);
                    io_.Finish(reply_, ::grpc::Status::OK, op_handle_.tag(
                        Handle::Operation::FINISH,
                        [this](bool ok, Handle::Operation /* op */) {

                            if (!ok) [[unlikely]] {
                            LOG_WARN << "The finish-operation failed.";
                        }

                        // We are done
                    }));
                    return;
                } // ok != false

                read(false);
        }));

    }

    Handle op_handle_{*this}; // We need only one handle for this operation.

    ::grpc::ServerContext ctx_;
    ::routeguide::Point req_;
    ::routeguide::RouteSummary reply_;
    ::grpc::ServerAsyncReader< decltype(reply_), decltype(req_)> io_{&ctx_};
};
```

这里的逻辑类似于 `ListFeaturesRequest`。不同的是，我们使用 `read()` 代替 `reply()`，并为了尽可能重用代码，我们设置了一个布尔标志来告诉我们这是第一次调用 `read()`——来自连接事件。如果不是第一次，则我们需要处理一个读取消息，然后在流上启动一个新的异步读取操作。

现在，我们将处理 gRPC 支持的最复杂的 RPC 请求类型——双向流。

## RouteChatRequest

在 proto 文件中，我们已指定请求和回复都是流。

```
  rpc RouteChat(stream RouteNote) returns (stream RouteNote) {}
```

我们无法指定实际的协议初始化。有些服务器可能希望在发送一个出站消息之前接收一个入站消息。其他服务器可能在成功发送一个消息之前不会开始读取流。一些协议采用轮流响应的方式，服务器和客户端对消息的响应都是通过发送消息来完成的。流中的消息可以是严格排序的，也可以是无关的，任意方向流动，只要一方有消息要发送即可。你在实现中的协议完全由你决定。只需记住，你在任何给定时间只能发送一条消息，并且只能有一个活跃的读取操作。如果需要发送一批消息，你必须提供自己的消息队列，并逐一将它们交给 gRPC，直到收到上一个消息成功发送的确认。

我们的 `RouteChat` 实现将会在请求激活后立即开始发送和读取。名字中有 "Chat" 呢！没有人指望互联网上的 "Chat" 会很有礼貌并遵守任何形式的 "协议" ;) 所以我们会不断向对方发送消息，直到我们累了为止。我们会读取并丢弃到达的入站消息，直到对方也感到疲倦。当双方都完成后，我们会说最后的话。作为服务器实现，我们定义了对话的状态。

```c++
class RouteChatRequest : public RequestBase {
public:

    RouteChatRequest(EverythingSvr& owner)
        : RequestBase(owner) {

        owner_.grpc().service_.RequestRouteChat(&ctx_, &stream_, cq(), cq(),
            in_handle_.tag(Handle::Operation::CONNECT,
                [this, &owner](bool ok, Handle::Operation /* op */) {

                    LOG_DEBUG << me(*this) << " - Processing a new connect from " << ctx_.peer();

                    if (!ok) [[unlikely]] {
                        // The operation failed.
                        // Let's end it here.
                        LOG_WARN << "The request-operation failed. Assuming we are shutting down";
                        return;
                    }

                    // Before we do anything else, we must create a new instance
                    // so the service can handle a new request from a client.
                    owner_.createNew<RouteChatRequest>(owner);

                    read(true);   // Initiate the read for the first incoming message
                    write(true);  // Initiate the first write operation.
        }));
    }

private:
    void read(const bool first) {
        if (!first) {
            // This is where we have read a message from the stream.
            // If this was code for a framework, this is where we would have called
            // the `onRpcRequestRouteChatGotMessage()` method, or unblocked the next statement
            // in a co-routine awaiting the next state-change.
            //
            // In our case, let's just log it.

            LOG_TRACE << "Incoming message: " << req_.message();

            req_.Clear();
        }

        // Start new read
        stream_.Read(&req_, in_handle_.tag(
            Handle::Operation::READ,
            [this](bool ok, Handle::Operation /* op */) {
                if (!ok) [[unlikely]] {
                    // The operation failed.
                    // This is normal on an incoming stream, when there are no more messages.
                    // As far as I know, there is no way at this point to deduce if the false status is
                    // because the client is done sending messages, or because we encountered
                    // an error.
                    LOG_TRACE << "The read-operation failed. It's probably not an error :)";

                    done_reading_ = true;
                    return finishIfDone();
                }

                read(false);  // Initiate the read for the next incoming message
        }));
    }

    void write(const bool first) {
        if (!first) {
            reply_.Clear();
        }

        if (++replies_ > owner_.config().num_stream_messages) {
            done_writing_ = true;

            LOG_TRACE << me(*this) << " - We are done writing to the stream.";
            return finishIfDone();
        }

        // This is where we are ready to write a new message.
        // If this was code for a framework, this is where we would have called
        // the `onRpcRequestRouteChatReadytoSendNewMessage()` method, or unblocked
        // the next statement in a co-routine awaiting the next state-change.

        reply_.set_message(std::string{"Server Message #"} + std::to_string(replies_));

        // Start new write
        stream_.Write(reply_, out_handle_.tag(
                            Handle::Operation::WRITE,
            [this](bool ok, Handle::Operation /* op */) {
                if (!ok) [[unlikely]] {
                    // The operation failed.
                    LOG_WARN << "The write-operation failed.";

                    // When ok is false here, we will not be able to write
                    // anything on this stream.
                    done_writing_ = true;
                    return finishIfDone();
                }

                write(false);  // Initiate the next write or finish
            }));
    }

    // We wait until all incoming messages are received and all outgoing messages are sent
    // before we send the finish message.
    void finishIfDone() {
        if (!sent_finish_ && done_reading_ && done_writing_) {
            LOG_TRACE << me(*this) << " - We are done reading and writing. Sending finish!";

            stream_.Finish(grpc::Status::OK, out_handle_.tag(
                Handle::Operation::FINISH,
                [this](bool ok, Handle::Operation /* op */) {

                    if (!ok) [[unlikely]] {
                        LOG_WARN << "The finish-operation failed.";
                    }

                    LOG_TRACE << me(*this) << " - We are done";
            }));
            sent_finish_ = true;
            return;
        }
    }

    bool done_reading_ = false;
    bool done_writing_ = false;
    bool sent_finish_ = false;
    size_t replies_ = 0;

    // We are streaming messages in and out simultaneously, so we need two handles.
    // One for each direction.
    Handle in_handle_{*this};
    Handle out_handle_{*this};

    ::grpc::ServerContext ctx_;
    ::routeguide::RouteNote req_;
    ::routeguide::RouteNote reply_;

    // Interestingly, the template the class is named `*ReaderWriter`, while
    // the template argument order is first Writer type and then Reader type.
    // Lot's of room for false assumptions and subtle errors here ;)
    ::grpc::ServerAsyncReaderWriter< decltype(reply_), decltype(req_)> stream_{&ctx_};
};
```

正如你所看到的，我们基本上将上述两个示例中的逻辑结合起来。然后，我们在 `finishIfDone()` 中添加了一些额外的逻辑，以允许流的两个方向都完成，然后再为 RPC 调用设置最终状态并结束它。

# 实现完整的 `routeguide` 异步客户端

现在，让我们复用为服务器创建的抽象，使用 gRPC 异步接口（或更准确地说是“桩”）来实现最终的客户端。

在查看请求实现之前，让我们简要看一下客户端对事件循环的重写。

```c++
    class EverythingClient
    : public EventLoopBase<ClientVars<::routeguide::RouteGuide>> {
    public:

        ...
        EverythingClient(const Config& config)
            : EventLoopBase(config) {


            LOG_INFO << "Connecting to gRPC service at: " << config.address;
            grpc_.channel_ = grpc::CreateChannel(config.address, grpc::InsecureChannelCredentials());

            grpc_.stub_ = ::routeguide::RouteGuide::NewStub(grpc_.channel_);
            assert(grpc_.stub_);

            // Add request(s)
            LOG_DEBUG << "Creating " << config_.parallel_requests
                    << " initial request(s) of type " << config_.request_type;

            for(auto i = 0; i < config_.parallel_requests;  ++i) {
                nextRequest();
            }
        }

    private:
        size_t request_count_{0};
```

在构造函数中，我们设置与 gRPC 服务器的连接，创建从我们的 proto 文件生成的 "stub" 实例，最后调用 `nextRequest()` 来初始化第一批传出的请求。我在这里省略了请求创建的代码，因为它对于更有趣的 gRPC 请求代码来说无关紧要。完整的源代码，包括能够运行我们所介绍的所有代码的测试客户端和测试服务器，可以在 GitHub 上找到。

## GetFeature

让我们从之前的GetFeature开始：

```c++
    class GetFeatureRequest : public RequestBase {
    public:

        GetFeatureRequest(EverythingClient& owner)
            : RequestBase(owner) {

            // Initiate the async request.
            rpc_ = owner.grpc().stub_->AsyncGetFeature(&ctx_, req_, cq());

            // Add the operation to the queue. We will be notified when
            // the request is completed.
            rpc_->Finish(&reply_, &status_, handle_.tag(
                Handle::Operation::FINISH,
                [this, &owner](bool ok, Handle::Operation /* op */) {

                    if (!ok) [[unlikely]] {
                    LOG_WARN << me(*this) << " - The request failed.";
                        return;
                    }

                    if (!status_.ok()) {
                        LOG_WARN << me(*this) << " - The request failed with error-message: "
                                 << status_.error_message();
                    }
                }));
        }

    private:
        Handle handle_{*this};

        // We need quite a few variables to perform our single RPC call.
        ::grpc::ClientContext ctx_;
        ::routeguide::Point req_;
        ::routeguide::Feature reply_;
        ::grpc::Status status_;
        std::unique_ptr< ::grpc::ClientAsyncResponseReader<decltype(reply_)>> rpc_;

    }; // GetFeatureRequest
```

这非常简单，大部分代码都在一个处理 `Finish` 事件的 lambda 函数中。

声明所需变量和初始化代码中仍然有一些样板代码。我们可以通过为一元 RPC 请求类型创建一个请求模板来增加另一层抽象。不过，gRPC 代码生成器对此没有提供太多帮助。如果它能为 `req_`、`reply_` 和 `rpc_` 三个变量类型提供 `typedef`，那就太好了。我们可能可以通过一些极端的模板元编程技巧，从初始化和完成方法中推断出这些类型，但今天我不打算深入这个问题。如果代码生成器能为我们添加 `using` 语句，那就太方便了 :/

让我们继续看 `ListFeatures`。

## ListFeatures

我们遵循与服务器流方法相同的模式。将处理特定事件类型的代码放入该事件的 lambda 表达式中。我们将连接和读取之间共享的逻辑放在 `read()` 方法中。

在构造函数中，我们同时初始化连接/请求操作和 `Finish` 操作。因此，我们有两个 `Handle` 变量。

```c++
 class ListFeaturesRequest : public RequestBase {
    public:

        ListFeaturesRequest(EverythingClient& owner)
            : RequestBase(owner) {

            // Initiate the async request.
            rpc_ = owner.grpc().stub_->AsyncListFeatures(&ctx_, req_, cq(), op_handle_.tag(
                Handle::Operation::CONNECT,
                [this](bool ok, Handle::Operation /* op */) {
                    if (!ok) [[unlikely]] {
                        LOG_WARN << me(*this) << " - The request failed (connect).";
                        return;
                    }

                    read(true);
            }));

            rpc_->Finish(&status_, finish_handle_.tag(
                Handle::Operation::FINISH,
                [this](bool ok, Handle::Operation /* op */) mutable {
                    if (!ok) [[unlikely]] {
                        LOG_WARN << me(*this) << " - The request failed (connect).";
                        return;
                    }

                    if (!status_.ok()) {
                        LOG_WARN << me(*this) << " - The request finished with error-message: "
                                 << status_.error_message();
                    }
            }));
        }

    private:
        void read(const bool first) {

            if (!first) {
                // This is where we have an actual message from the server.
                // If this was a framework, this is where we would have called
                // `onListFeatureReceivedOneMessage()` or or unblocked the next statement
                // in a co-routine waiting for the next request

                // In our case, let's just log it.
                LOG_TRACE << me(*this) << " - Request successful. Message: " << reply_.name();

                // Prepare the reply-object to be re-used.
                // This is usually cheaper than creating a new one for each read operation.
                reply_.Clear();
            }

            // Now, lets register another read operation
            rpc_->Read(&reply_, op_handle_.tag(
                                    Handle::Operation::READ,
                [this](bool ok, Handle::Operation /* op */) {
                    if (!ok) [[unlikely]] {
                        LOG_TRACE << me(*this) << " - The read-request failed.";
                        return;
                    }

                    read(false);
                }));
        }

        Handle op_handle_{*this};
        Handle finish_handle_{*this};

        ::grpc::ClientContext ctx_;
        ::routeguide::Rectangle req_;
        ::routeguide::Feature reply_;
        ::grpc::Status status_;
        std::unique_ptr< ::grpc::ClientAsyncReader< decltype(reply_)>> rpc_;
    }; // ListFeaturesRequest
```

## RecordRouteRequest

这与之前的示例非常相似。区别在于我们是在发送而不是接收流中的数据。

```c++
  class RecordRouteRequest : public RequestBase {
    public:

        RecordRouteRequest(EverythingClient& owner)
            : RequestBase(owner) {

            // Initiate the async request (connect).
            rpc_ = owner.grpc().stub_->AsyncRecordRoute(&ctx_, &reply_, cq(), io_handle_.tag(
                Handle::Operation::CONNECT,
                [this](bool ok, Handle::Operation /* op */) {
                    if (!ok) [[unlikely]] {
                        LOG_WARN << me(*this) << " - The request failed (connect).";
                        return;
                    }

                    // The server will not send anything until we are done writing.
                    // So let's get started.

                    write(true);
               }));

            // Register a handler to be called when the server has sent a reply and final status.
            rpc_->Finish(&status_, finish_handle_.tag(
                Handle::Operation::FINISH,
                [this](bool ok, Handle::Operation /* op */) mutable {
                    if (!ok) [[unlikely]] {
                        LOG_WARN << me(*this) << " - The request failed (connect).";
                        return;
                    }

                    if (!status_.ok()) {
                        LOG_WARN << me(*this) << " - The request finished with error-message: "
                                 << status_.error_message();
                    }
               }));
        }

    private:
        void write(const bool first) {

            if (!first) {
                req_.Clear();
            }

            if (++sent_messages_ > owner_.config().num_stream_messages) {

                LOG_TRACE << me(*this) << " - We are done writing to the stream.";

                rpc_->WritesDone(io_handle_.tag(
                    Handle::Operation::WRITE_DONE,
                    [this](bool ok, Handle::Operation /* op */) {
                        if (!ok) [[unlikely]] {
                            LOG_TRACE << me(*this) << " - The writes-done request failed.";
                            return;
                        }

                        LOG_TRACE << me(*this) << " - We have told the server that we are done writing.";
                    }));

                return;
            }

            // Send some data to the server
            req_.set_latitude(100);
            req_.set_longitude(sent_messages_);

            // Now, lets register another write operation
            rpc_->Write(req_, io_handle_.tag(
                Handle::Operation::WRITE,
                [this](bool ok, Handle::Operation /* op */) {
                    if (!ok) [[unlikely]] {
                        LOG_TRACE << me(*this) << " - The write-request failed.";
                        return;
                    }

                    write(false);
                }));
        }

        Handle io_handle_{*this};
        Handle finish_handle_{*this};
        size_t sent_messages_ = 0;

        ::grpc::ClientContext ctx_;
        ::routeguide::Point req_;
        ::routeguide::RouteSummary reply_;
        ::grpc::Status status_;
        std::unique_ptr<  ::grpc::ClientAsyncWriter< ::routeguide::Point>> rpc_;
    }; // RecordRouteRequest
```

最后一个示例是双向流。就像在服务器端一样，我们实现了一个真正的互联网聊天 (tm)，在其中我们对接收方大喊大叫，直到把我们想说的都喊出来。然后我们结束并等待服务器说出最后几句话（状态）。与此同时，我们从服务器读取消息并将其丢弃（像真正的互联网讨论参与者一样），直到他们有礼貌地闭嘴。

```c++
class RouteChatRequest : public RequestBase {
    public:

        RouteChatRequest(EverythingClient& owner)
            : RequestBase(owner) {

            // Initiate the async request.
            rpc_ = owner.grpc().stub_->AsyncRouteChat(&ctx_, cq(), in_handle_.tag(
                Handle::Operation::CONNECT,
                [this](bool ok, Handle::Operation /* op */) {
                    if (!ok) [[unlikely]] {
                        LOG_WARN << me(*this) << " - The request failed (connect).";
                        return;
                    }

                    // We are initiating both reading and writing.
                    // Some clients may initiate only a read or a write at this time,
                    // depending on the use-case.
                    read(true);
                    write(true);
                }));

            rpc_->Finish(&status_, finish_handle_.tag(
                Handle::Operation::FINISH,
                [this](bool ok, Handle::Operation /* op */) mutable {
                    if (!ok) [[unlikely]] {
                        LOG_WARN << me(*this) << " - The request failed (finish).";
                        return;
                    }

                    if (!status_.ok()) {
                        LOG_WARN << me(*this) << " - The request finished with error-message: "
                                 << status_.error_message();
                   }
                }));
        }

    private:
        void read(const bool first) {

            if (!first) {
                // This is where we have an actual message from the server.
                // If this was a framework, this is where we would have called
                // `onListFeatureReceivedOneMessage()` or or unblocked the next statement
                // in a co-routine waiting for the next request

                // In our case, let's just log it.
                LOG_TRACE << me(*this) << " - Request successful. Message: " << reply_.message();
                reply_.Clear();
            }

            // Now, lets register another read operation
            rpc_->Read(&reply_, in_handle_.tag(
                Handle::Operation::READ,
                [this](bool ok, Handle::Operation /* op */) {
                    if (!ok) [[unlikely]] {
                        LOG_TRACE << me(*this) << " - The read-request failed.";
                        return;
                    }

                    read(false);
                }));
        }

        void write(const bool first) {

            if (!first) {
                req_.Clear();
            }

            if (++sent_messages_ > owner_.config().num_stream_messages) {

                LOG_TRACE << me(*this) << " - We are done writing to the stream.";

                rpc_->WritesDone(out_handle_.tag(
                    Handle::Operation::WRITE_DONE,
                    [this](bool ok, Handle::Operation /* op */) {
                        if (!ok) [[unlikely]] {
                            LOG_TRACE << me(*this) << " - The writes-done request failed.";
                            return;
                        }

                        LOG_TRACE << me(*this) << " - We have told the server that we are done writing.";
                  }));

                return;
            }

            // Now, lets register another write operation
            rpc_->Write(req_, out_handle_.tag(
                Handle::Operation::WRITE,
                [this](bool ok, Handle::Operation /* op */) {
                    if (!ok) [[unlikely]] {
                        LOG_TRACE << me(*this) << " - The write-request failed.";
                        return;
                    }

                    write(false);
                }));
        }

        Handle in_handle_{*this};
        Handle out_handle_{*this};
        Handle finish_handle_{*this};
        size_t sent_messages_ = 0;

        ::grpc::ClientContext ctx_;
        ::routeguide::RouteNote req_;
        ::routeguide::RouteNote reply_;
        ::grpc::Status status_;
        std::unique_ptr<  ::grpc::ClientAsyncReaderWriter< ::routeguide::RouteNote, ::routeguide::RouteNote>> rpc_;
    };
```

代码与服务器端实现类似，只是我们没有最后的话要说 ;)

# 探索 C++ 中 gRPC 的异步回调接口

如果你读过本系列的上一篇文章，你可能会同意，即使是使用异步 API 实现最简单的 gRPC 接口也需要大量的样板代码。这还不包括深入线程模型和优化的部分。

幸运的是，我们可以使用一个更新的接口。使用这个接口，我们不需要实现事件循环和处理所有细节。rpcgen 实现了一个回调接口以及传统的异步接口。我没有在 gRPC C++ 的着陆页上看到它的提及——所以我希望我不是在揭露一个大秘密 ;)

回调接口也是异步的。如果我们在服务器端使用它，我们只需要为每个 RPC 重写一个方法。嗯——有点类似。带有一个流（任何方向）的那些方法本来可以实现为协程。gRPC 不支持这一点，我们不得不退回到实现第二个接口来处理流的流式方法。

我找到的关于回调接口的唯一文档是一个“C++ 基于回调的异步 API”提案文档，这个提案似乎已经被接受。

在 gRPC 代码库中还有两个示例，展示了如何使用它：

- `route_guide_callback_server.cc`
- `route_guide_callback_client.cc`

与异步 API 相比，回调接口提供了极大的简化。它仍然需要一些代码，但对于大多数项目来说——不处理事件循环和底层线程模型可能更好。gRPC 团队积累了来自许多项目的经验，通过将底层细节留给他们，我们可以期待比一般的 C++ 开发团队（没有深入 gRPC 知识或没有时间实现各种方法和实验来找到更好解决方案）更好的整体性能和资源利用。根据我的经验，大多数团队只是想实现一些 RPC 接口，他们的重点在于自己的功能。他们不想过多地深入 gRPC。

我认为，gRPC 团队把 C++ 开发者引导到传统的异步接口，而没有在 C++ 着陆页上提及回调接口是一个巨大的错误。我认为异步接口对于一些大规模实现或有特殊需求的用例可能有用。但这些用例需要具有深入 gRPC 知识的高级 C++ 开发人员。对于大多数项目，回调接口应该是起点。至少，我是这么认为的。

使用回调接口可以使我们在 C++ 项目中选择 gRPC 作为服务间或系统间通信变得更容易。它仍然需要仔细编码以避免性能瓶颈、资源泄漏和竞争条件。在 2023 年，这不应该是情况！

我忍不住——我仍然希望 gRPC 团队或其他人发布一个开源工具，生成友好的协程版本的 RPC，这些工具可以处理任务调度、引用计数，并移除我们在 C++ 代码中使用的 gRPC API 的所有当前限制。

在接下来的两篇文章中，我们将一起探索 gRPC 回调接口，首先是服务器端，然后是客户端。

# 探索如何使用 gRPC 异步回调实现服务器

回调接口在服务器端为我们提供了一个 C++ 虚拟接口，让我们可以重写方法来实现我们 proto 文件中的 RPC。这正是我对 C++ RPC 代码生成器的期望。当我几年前开始尝试异步 gRPC 时，我对当时只有传统异步接口感到有些惊讶。

那么，这次他们做得对吗？有是有，但也有不足。

有，因为他们现在处理了所有细节，这样我们可以直接处理 RPC 请求。我们实现了一个重写 rpcgen 创建的接口的方法，而不必担心调用之前或请求完成后的事情。我强调请求，因为对于流式 RPC 类型，回调只是工作流的启动。我们的 RPC 实现方法是由 gRPC 拥有的固定大小线程池调用的。如果我们在这里执行任何耗时的操作，我们的服务器将不会很高效。因此，我们必须立即从回调中返回。为了简化流式 RPC，我们将创建一个对象来执行完成请求所需的操作。这也是我们在异步实现中所做的——只不过那时我们为每个 RPC 方法预创建了一个实例，然后在有新的 RPC 请求进程时创建一个新实例。使用回调时，我们只需在有新的 RPC 时获得通知，然后由我们决定如何继续。

不足的是，我们仍然需要为每个流 API 创建一个实现类，以处理状态和事件流。

需要记住的一点是，回调可能会从不同线程同时调用。我们的实现必须是线程安全的。

简化来说，我们在服务器端流式 RPC 中实现的内容如下。我们为每个 RPC 方法创建重写，并且还为读取/写入流所需的流式类创建实现类。

```c++
class CallbackImpl : public ::routeguide::RouteGuide::CallbackService {
public:

    class ServerWriteReactorImpl
        : public ::grpc::ServerWriteReactor< ::routeguide::Feature> {
    public:
        ServerWriteReactorImpl() { ... }

        // Override callbacks for events we need
        void OnDone() override { ... }
        void OnWriteDone(bool ok) override { ... }

    private:
        // Implement functionality we need
        reply() { ... }
    };

    // Example of an RPC with one request in and a stream out
    auto ListFeatures (::grpc::CallbackServerContext* ctx,
        const ::routeguide::Rectangle* req) override {

        return new ServerWriteReactorImpl();
    }
    ...
};

```

真正令人兴奋的好消息是，单向 RPC（没有流的 RPC）实现起来非常简单。我认为这些将在大多数实际用例中占据主导地位。

## GetFeature

这是`GetFeature()`方法的重写。

```c++

class CallbackServiceImpl : public ::routeguide::RouteGuide::CallbackService {
public:
    ...

    grpc::ServerUnaryReactor *GetFeature(grpc::CallbackServerContext *ctx,
                                            const routeguide::Point *req,
                                            routeguide::Feature *resp) override {

        // Give a nice, thoughtful response
        resp->set_name("whatever");

        // We could have implemented our own reactor, but this is the recommended
        // way to do it in unary methods.
        auto* reactor = ctx->DefaultReactor();
        reactor->Finish(grpc::Status::OK);
        return reactor;
    }

    ...
};

```

我们的回调将在每次有人请求该 RPC 时被调用。请记住，我们无法控制线程。gRPC 将使用自己的线程池，我们必须预期我们的方法可能会被多次并行调用。如果我们使用了一些共享资源，我们必须使用锁或其他同步策略来处理竞态条件。

还要记住，回调必须立即返回。如果需要进行 I/O 或复杂的计算，必须将请求放到另一个线程上进行处理，然后从回调中返回。你可以稍后调用 Finish()。这当然会增加代码的复杂性。但这就是规则，我们必须遵守。顺便说一下，这在我们的异步实现中也是如此。

## ListFeatures

在这里，我们使用了与 gRPC 团队在其回调接口示例中使用的相同模式。我们将流类/工作流的实现放在回调方法的重写中。

```c++
/*! RPC callback event for ListFeatures
     */
    ::grpc::ServerWriteReactor< ::routeguide::Feature>* ListFeatures(
        ::grpc::CallbackServerContext* ctx, const ::routeguide::Rectangle* req) override {

        // Now, we need to keep the state and buffers required to handle the stream
        // until the request is completed. We also need to return from this
        // method immediately.
        //
        // We solve these conflicting requirements by implementing a class that overrides
        // the async gRPC stream interface `ServerWriteReactor`, and then adding our
        // state and buffers here.
        //
        // This class will deal with the request. The method we are in will just
        // allocate an instance of our class on the heap and return.
        //
        // Yes - it's ugly. It's the best the gRPC team have been able to come up with :/
        class ServerWriteReactorImpl
            // Some shared code we need for convenience for all the RPC Request classes
            : public ReqBase<ServerWriteReactorImpl>
            // The interface to the gRPC async stream for this request.
            , public ::grpc::ServerWriteReactor< ::routeguide::Feature> {
        public:
            ServerWriteReactorImpl(CallbackSvc& owner)
                : owner_{owner} {

                // Start replying with the first message on the stream
                reply();
            }

            /*! Callback event when the RPC is completed */
            void OnDone() override {
                done();
            }

            /*! Callback event when a write operation is complete */
            void OnWriteDone(bool ok) override {
                if (!ok) [[unlikely]] {
                    LOG_WARN << "The write-operation failed.";

                    // We still need to call Finish or the request will remain stuck!
                    Finish({grpc::StatusCode::UNKNOWN, "stream write failed"});
                    return;
                }

                reply();
            }

        private:
            void reply() {
                // Reply with the number of messages in config
                if (++replies_ > owner_.config().num_stream_messages) {
                    reply_.Clear();

                    // Since it's a stream, it make sense to return different data for each message.
                    reply_.set_name(std::string{"stream-reply #"} + std::to_string(replies_));

                    return StartWrite(&reply_);
                }

                // Didn't write anything, all is done.
                Finish(grpc::Status::OK);
            }

            CallbackSvc& owner_;
            size_t replies_ = 0;
            ::routeguide::Feature reply_;
        };

        return createNew<ServerWriteReactorImpl>(owner_);
    };
```

如你所见，逻辑与我们最后的异步示例类似。但它更简单。简单是好的，它减少了出错的机会 ;)

## RecordRoute

在这里，我们从流中读取数据，直到它结束（ok == false）。然后我们回复。

正如之前所述，代码比之前更简单。

```c++
 /*! RPC callback event for RecordRoute
      */
    ::grpc::ServerReadReactor< ::routeguide::Point>* RecordRoute(
        ::grpc::CallbackServerContext* ctx, ::routeguide::RouteSummary* reply) override {

        class ServerReadReactorImpl
            // Some shared code we need for convenience for all the RPC Request class
            : public ReqBase<ServerReadReactorImpl>
            // The async gRPC stream interface for this RPC
            , public grpc::ServerReadReactor<::routeguide::Point> {
        public:
            ServerReadReactorImpl(CallbackSvc& owner, ::routeguide::RouteSummary* reply)
                : owner_{owner}, reply_{reply} {
                assert(reply_);

                // Initiate the first read operation
                StartRead(&req_);
            }

                /*! Callback event when the RPC is complete */
            void OnDone() override {
                done();
            }

            /*! Callback event when a read operation is complete */
            void OnReadDone(bool ok) override {
                if (ok) {
                    // We have read a message from the request.

                    LOG_TRACE << "Got message: longitude=" << req_.longitude()
                                << ", latitude=" << req_.latitude();

                    req_.Clear();

                    // Initiate the next async read
                    return StartRead(&req_);
                }

                LOG_TRACE << "The read-operation failed. It's probably not an error :)";

                // Let's compose an exiting reply to the client.
                reply_->set_distance(100);
                reply_->set_distance(300);

                // Note that we set the reply (in the buffer we got from gRPC) and call
                // Finish in one go. We don't have to wait for a callback to acknowledge
                // the write operation.
                Finish(grpc::Status::OK);
                // When the client has received the last bits from us in regard of this
                // RPC, `OnDone()` will be the final event we receive.
            }

        private:
            CallbackSvc& owner_;

            // Our buffer for each of the outgoing messages on the stream
            ::routeguide::Point req_;

            // Note that for this RPC type, gRPC owns the reply-buffer.
            // It's a bit inconsistent and confusing, as the interfaces mostly takes
            // pointers, but usually our implementation owns the buffers.
            ::routeguide::RouteSummary *reply_ = {};
        };

        // This is all our method actually does. It just creates an instance
        // of the implementation class to deal with the request.
        return createNew<ServerReadReactorImpl>(owner_, reply);
    };
```

我对 protobuf 生成的头文件中缺少 typedefs 感到有些困惑。我们不得不在代码中详细说明完整的模板名称和参数，这非常耗时。通常，所有这些内容必须从生成的头文件中找到，然后复制和粘贴到代码中。

## RouteChat

再次说明，这是 gRPC 支持的最复杂的 RPC 类型。

快速回顾一下 proto 定义：

```c++
    rpc RouteChat(stream RouteNote) returns (stream RouteNote) {}
```

当我们使用回调时，我们必须准备好处理来自多个线程的并发事件。在我们的异步代码中，我们只使用了一个线程来处理事件循环，这个线程处理了所有事件。因此，我们不需要使事件处理程序线程安全。而当我们使用回调时，除了可能会有新的 RPC 同时在不同线程中到来，我们还可能会在双向流上同时接收到 IO 事件（读/写）。因此，如果事件处理程序共享了非 const 数据，请务必小心。

这个实现借鉴了我们异步代码中如何处理“双向”流的核心思想。它使用 `read()` 和 `write()` 方法来启动异步操作。然后，它重写了 `ServerBidiReactor<>` 接口中的 IO 事件处理程序来处理完成事件。

如前所述，直到完成读取和写入之前，我们不会调用 `Finish()`。这由 `finishIfDone()` 处理。

```c++
 /*! RPC callback event for RouteChat
     */
    ::grpc::ServerBidiReactor< ::routeguide::RouteNote, ::routeguide::RouteNote>*
        RouteChat(::grpc::CallbackServerContext* ctx) override {

        class ServerBidiReactorImpl
            // Some shared code we need for convenience for all the RPC Request classes
            : public ReqBase<ServerBidiReactorImpl>
            // The async gRPC stream interface for this RPC
            , public grpc::ServerBidiReactor<::routeguide::RouteNote, ::routeguide::RouteNote> {
        public:
            ServerBidiReactorImpl(CallbackSvc& owner)
                : owner_{owner} {

               /* There are multiple ways to handle the message-flow in a bidirectional stream.
                *
                * One party can send the first message, and the other party can respond with a message,
                * until one or both parties gets bored.
                *
                * Both parties can wait for some event to occur, and send a message when appropriate.
                *
                * One party can send occasional updates (for example location data) and the other
                * party can respond with one or more messages when appropriate.
                *
                * Both parties can start sending messages as soon as the connection is made.
                * That's what we are doing (or at least preparing for) in this example.
                */

                read();   // Initiate the read for the first incoming message
                write();  // Initiate the first write operation.
            }

            /*! Callback event when the RPC is complete */
            void OnDone() override {
                done();
            }

            /*! Callback event when a read operation is complete */
            void OnReadDone(bool ok) override {
                if (!ok) {
                    LOG_TRACE << me() << "- The read-operation failed. It's probably not an error :)";
                    done_reading_ = true;
                    return finishIfDone();
                }

                LOG_TRACE << "Incoming message: " << req_.message();
                read();
            }

            /*! Callback event when a write operation is complete */
            void OnWriteDone(bool ok) override {
                if (!ok) [[unlikely]] {
                    // The operation failed.
                    LOG_WARN << "The write-operation failed.";

                    // When ok is false here, we will not be able to write
                    // anything on this stream.
                    done_writing_ = true;

                    // This RPC call did not end well
                    status_ = {grpc::StatusCode::UNKNOWN, "write failed"};

                    return finishIfDone();
                }

                write();
            }

        private:
            void read() {
                // Start a new read
                req_.Clear();
                StartRead(&req_);
            }

            void write() {
                if (++replies_ > owner_.config().num_stream_messages) {
                    done_writing_ = true;

                    LOG_TRACE << me() << " - We are done writing to the stream.";
                    return finishIfDone();
                }

                // Prepare a new message to the stream
                reply_.Clear();

                // Shout something nice to the other side
                reply_.set_message(std::string{"Server Message #"} + std::to_string(replies_));

                // Start new write on the stream
                StartWrite(&reply_);
            }

            void finishIfDone() {
                if (!sent_finish_ && done_reading_ && done_writing_) {
                    LOG_TRACE << me() << " - We are done reading and writing. Sending finish!";
                    Finish(status_);
                    sent_finish_ = true;
                    return;
                }
            }

            CallbackSvc& owner_;
            ::routeguide::RouteNote req_;
            ::routeguide::RouteNote reply_;
            grpc::Status status_;
            size_t replies_ = 0;
            bool done_reading_ = false;
            bool done_writing_ = false;
            bool sent_finish_ = false;
        };


        auto instance = createNew<ServerBidiReactorImpl>(owner_);
        LOG_TRACE << instance->me()
                    << " - Starting new bidirectional stream conversation with "
                    << ctx->peer();
        return instance;
    }
```

即便是双向 RPC 也比异步版本简单得多。在实际用例中，情况无疑会更加复杂，因为我们需要处理实际的功能，可能需要同时排队处理传出的消息（由于我们一次只能发送一条），以及通过自己的工作线程转发请求和响应，以便服务器能够承受磁盘或网络延迟和数据库查询带来的延迟。不过，现在处理 RPC 和流所需的代码已经降低到了一个可以管理的水平了 ;)

# 探索如何使用 gRPC 异步回调实现客户端

客户端接口与服务器接口不同。在这里，我们不会重写接口。相反，我们调用存根接口上的方法，这与我们在异步版本中所做的类似。

从 rpcgen 生成的“回调”代码如下：

```c++

class async final :
    public StubInterface::async_interface {
    public:
    void GetFeature(::grpc::ClientContext* context, const ::routeguide::Point* request, ::routeguide::Feature* response, std::function<void(::grpc::Status)>) override;
    void GetFeature(::grpc::ClientContext* context, const ::routeguide::Point* request, ::routeguide::Feature* response, ::grpc::ClientUnaryReactor* reactor) override;
    void ListFeatures(::grpc::ClientContext* context, const ::routeguide::Rectangle* request, ::grpc::ClientReadReactor< ::routeguide::Feature>* reactor) override;
    void RecordRoute(::grpc::ClientContext* context, ::routeguide::RouteSummary* response, ::grpc::ClientWriteReactor< ::routeguide::Point>* reactor) override;
    void RouteChat(::grpc::ClientContext* context, ::grpc::ClientBidiReactor< ::routeguide::RouteNote,::routeguide::RouteNote>* reactor) override;
    private:
    friend class Stub;
    explicit async(Stub* stub): stub_(stub) { }
    Stub* stub() { return stub_; }
    Stub* stub_;
};
```

单一 RPC，如 GetFeature，有两个变体，其中一个接受 `std::function<>` 作为参数。在我们的示例中，我们将使用这个变体。

我们得到的方法将启动一个异步请求到服务器。然后发生的各种事件由我们作为最后一个参数提供的回调处理。对于提供函数的单一方法，当 RPC 完成时，该函数将被调用一次。这是很方便的。其他方法则要求我们传递一个重写事件的类，以便我们可以通过流发送/接收数据。

在下面的代码中，我将这些方法封装在更方便的方法中，通过回调来调用 RPC 并与其事件进行交互。这对于大多数 C++ 开发人员来说是一个熟悉的模式。

我还添加了展示如何使用这些封装方法的方法。

让我们从初始化开始，直到我们有一个可用的 `stub_`，可以用来调用 RPC。

```c++
class EverythingCallbackClient {
public:

EverythingCallbackClient(const Config& config)
    : config_{config} {

    LOG_INFO << "Connecting to gRPC service at: " << config.address;
    channel_ = grpc::CreateChannel(config.address, grpc::InsecureChannelCredentials());

    stub_ = ::routeguide::RouteGuide::NewStub(channel_);
    assert(stub_);
}

...

private:
    const Config config_;

    // This is a connection to the gRPC server
    std::shared_ptr<grpc::Channel> channel_;

    // An instance of the client that was generated from our .proto file.
    std::unique_ptr<::routeguide::RouteGuide::Stub> stub_;
```

## GetFeature

现在，让我们看一下最简单的单一 RPC 包装方法。

我们使用一个类来保存 RPC 的缓冲区和状态。虽然在这种情况下，这些可以由调用者处理，但如果你想在代码的各个地方调用一些 RPC，管理共享指针或其他所有缓冲区的方法可能会增加负担。将这些细节隐藏在包装器中并在内部处理，更为简便。

```c++
/// Callback function with the result of the unary RPC call
    using get_feature_cb_t = std::function<void(const grpc::Status& status,
                                                const ::routeguide::Feature& feature)>;

    /*! Naive implementation of `GetFeature` with an actual callback.
     */
    void getFeature(::routeguide::Point& point, get_feature_cb_t && fn) {

        // In order to keep the state for the duration of the async request,
        // we use this class.
        class Impl : Base {
        public:
            Impl(EverythingCallbackClient& owner,
                 ::routeguide::Point& point,
                 get_feature_cb_t && fn)
                : Base(owner), req_{std::move(point)}, caller_callback_{std::move(fn)} {

                LOG_TRACE << "getFeature starting async request.";

                owner_.stub_->async()->GetFeature(&ctx_, &req_, &reply_,
                                           [this](grpc::Status status) {

                    LOG_TRACE << "getFeature calling finished callback.";
                    caller_callback_(status, reply_);
                    delete this;
                });
            }

        private:
            grpc::ClientContext ctx_;
            ::routeguide::Point req_;
            ::routeguide::Feature reply_;
            get_feature_cb_t caller_callback_;
        };

        // This is all our method actually does. It just creates an instance
        // of the implementation class to deal with the request.
        new Impl(*this, point, std::move(fn));
    } // getFeature

```

然后这是一个任何使用这个方法的例子。

```c++

    /*! Example on how to use getFeature() */
    void nextGetFeature() {
        // Initiate a new request
        ::routeguide::Point point;
        point.set_latitude(10);
        point.set_longitude(100);

        getFeature(point, [this](const grpc::Status& status,
                                 const ::routeguide::Feature& feature) {
            if (status.ok()) {
                LOG_TRACE << "Received feature: " << feature.name();

            } else {
                LOG_TRACE << "#" << recid << " failed: "
                          << status.error_message();
            }
        });
    }

```

现在，即使这些单一的 gRPC 调用是异步的，它们变得更加方便使用了。这真是个好消息！

## ListFeatures

我们继续为 `ListFeatures` 创建一个包装器。

```c++
 /*! Data for a callback function suitable for `ListFeatures`.
     *
     *  Here we use a union that either contains a Feature message received
     *  from the stream, or the final Status message. Alternatively we could have
     *  used two callbacks. However, std::variant (C++ unions) can be useful in
     *  such cases as this.
     *
     *  We use a pointer to the Feature so that we don't have to make a deep
     *  copy for each received message just for the purpose of "doing the right thing" ;)
     */
    using feature_or_status_t = std::variant<
            // I would have preferred a reference, but that don't work in C++ 20 in variants
            const ::routeguide::Feature *,
            grpc::Status
        >;

    /*! A callback function suitable to handle the events following a ListFeatures RPC
     */
    using list_features_cb_t = std::function<void(feature_or_status_t)>;

    /*! Naive implementation of `ListFeatures` with a callback for the events.
     */
    void listFeatures(::routeguide::Rectangle& rect, list_features_cb_t&& fn) {

        /*! A class that deals with this RPC
         *
         *  It overrides the events we need, owns our buffers and implements
         *  our boilerplate code needed to call the user supplied callback.
         */
        class Impl
            // Our shared code for all the RPC's we implementn here
            : Base
            // The async gRPC stream interface for this RPC
            , grpc::ClientReadReactor<::routeguide::Feature> {
        public:

            Impl(EverythingCallbackClient& owner,
                 ::routeguide::Rectangle& rect,
                 list_features_cb_t && fn)
                : Base(owner), req_{std::move(rect)}, caller_callback_{std::move(fn)} {

                LOG_TRACE << "listFeatures starting async request.";

                // Glue this instance of this class to an initiation
                // of the ListFeatures RPC.
                owner_.stub_->async()->ListFeatures(&ctx_, &req_, this);

                // Initiate the first async read.
                // Since StartCall is not yet called, the operation is
                // queued by gRPC.
                StartRead(&resp_);

                // Here we initiate the actual async RPC
                StartCall();
            }

            /*! Callback event when a read operation is complete */
            void OnReadDone(bool ok) override {

                // We go on reading while ok
                if (ok) {
                    LOG_TRACE << "Request successful. Message: " << resp_.name();

                    caller_callback_(&resp_);

                    resp_.Clear();
                    return StartRead(&resp_);
                }

                LOG_TRACE << "Read failed (end of stream?)";
            }

            /*! Callback event when the RPC is complete */
            void OnDone(const grpc::Status& s) override {
                if (s.ok()) {
                    LOG_TRACE << "Request succeeded.";
                } else {
                    LOG_WARN << "Request failed: " << s.error_message();
                }

                caller_callback_(s);
                delete this;
            }

        private:
            grpc::ClientContext ctx_;
            ::routeguide::Rectangle req_;
            ::routeguide::Feature resp_;
            list_features_cb_t caller_callback_;
        };

        // All this method actually does.
        new Impl(*this, rect, std::move(fn));
    } // listFeatures
```

那代码多了一些。现在我们来看一下如何使用这个包装器：

```c++
/*! Example on how to use listFeatures() */
    void nextListFeatures() {
        ::routeguide::Rectangle rect;
        rect.mutable_hi()->set_latitude(1);
        rect.mutable_hi()->set_longitude(2);

        LOG_TRACE << "Calling listFeatures";

        listFeatures(rect, [this](feature_or_status_t val) {

            if (std::holds_alternative<const ::routeguide::Feature *>(val)) {
                auto feature = std::get<const ::routeguide::Feature *>(val);

                LOG_TRACE << "nextListFeatures - Received feature: " << feature->name();
            } else if (std::holds_alternative<grpc::Status>(val)) {
                auto status = std::get<grpc::Status>(val);
                if (status.ok()) {
                    LOG_TRACE << "nextListFeatures done. Initiating next request ...";
                    nextRequest();
                } else {
                    LOG_TRACE << "nextListFeatures failed: " <<  status.error_message();
                }
            } else {
                assert(false && "unexpected value type in variant!");
            }

        });
    }
```

看看这个！这是一个简洁的回调接口用于流式 RPC！

## **RecordRoute**

这个包装器与之前的类似，只不过流的方向相反。

```c++
/*! Definition of a callback function that need to provide the data for a write.
     *
     *  The function must return immediately.
     *
     *  \param point. The buffer for the data.
     *  \return true if we are to write data, false if all data has been written and
     *      we are done.
     */
    using on_ready_to_write_point_cb_t = std::function<bool(::routeguide::Point& point)>;

    /*! Definition of a callback function that is called when the RPC is complete.
     *
     *  \param status. The status for the RPC.
     *  \param summary. The reply from the server. Only valid if `staus.ok()` is true.
     */
    using on_done_route_summary_cb_t = std::function<
        void(const grpc::Status& status, ::routeguide::RouteSummary& summary)>;

    /*! Naive implementation of `RecordRoute` with callbacks for the events.
     */
    void recordRoute(on_ready_to_write_point_cb_t&& writerCb, on_done_route_summary_cb_t&& doneCb) {

        /*! A class that deals with this RPC
         *
         *  It overrides the events we need, owns our buffers and implements
         *  our boilerplate code needed to call the user supplied callback.
         */
        class Impl
            // Our shared code for all the RPC's we implement here
            : Base
            // The async gRPC stream interface for this RPC
            , grpc::ClientWriteReactor<::routeguide::Point> {
        public:
            Impl(EverythingCallbackClient& owner,
                 on_ready_to_write_point_cb_t& writerCb,
                 on_done_route_summary_cb_t& doneCb)
                : Base(owner), writer_cb_{std::move(writerCb)}
                , done_cb_{std::move(doneCb)}
            {
                LOG_TRACE << "recordRoute starting async request.";

                // Glue this instance of this class to an initiation
                // of the RecordRoute RPC.
                owner_.stub_->async()->RecordRoute(&ctx_, &resp_, this);

                // Start the first async write operation on the stream
                write();

                // Here we will initiate the actual async RPC
                StartCall();
            }

            /*! Callback event when a write operation is complete */
            void OnWriteDone(bool ok) override {
                if (!ok) [[unlikely]] {
                    LOG_WARN << "RecordRoute - Failed to write to the stream: ";

                    // Once failed here, we cannot write any more.
                    return StartWritesDone();
                }

                // One write operation was completed.
                // Off we go with the next one.
                write();
            }

            /*! Callback event when the RPC is complete */
            void OnDone(const grpc::Status& s) override {
                done_cb_(s, resp_);
                delete this;
            }

        private:

            // Initiate a new async write operation (if appropriate).
            void write() {
                // Get another message from the caller
                req_.Clear();
                if (writer_cb_(req_)) {

                    // Note that if we had implemented a delayed `StartWrite()`, for example
                    // by relaying the event to a task manager like `boost::asio::io_service::post()`,
                    // we would need to call `AddHold()` either here or in the constructor.
                    // Only when we had called `RemoveHold()` the same number of times, would
                    // gRPC consider calling the `OnDone()` event method.

                    return StartWrite(&req_);
                }

                // The caller don't have any further data to write
                StartWritesDone();
            }

            grpc::ClientContext ctx_;
            ::routeguide::Point req_;
            ::routeguide::RouteSummary resp_;
            on_ready_to_write_point_cb_t writer_cb_;
            on_done_route_summary_cb_t done_cb_;
        };

        // All this method actually does.
        new Impl(*this, writerCb, doneCb);
    } // recordRoute
```

然后这是如何使用这个包装器的例子：

```c++
 /*! Example on how to use recordRoute() */
    void nextRecordRoute() {

        recordRoute(
            // Callback to provide data to send to the server
            // Note that we instantiate a local variable `count` that lives
            // in the scope of one instance of the lambda function.
            [this, count=size_t{0}](::routeguide::Point& point) mutable {
                if (++count > config_.num_stream_messages) [[unlikely]] {
                    // We are done
                    return false;
                }

                // Just pick some data to set.
                // In a real implementation we would have to get prerpared data or
                // data from a quick calculation. We have to return immediately since
                // we are using one of gRPC's worker threads.
                // If we needed to do some work, like fetching from a database, we
                // would need another workflow where the event was dispatched to a
                // task manager or thread-pool, argument was a write functor rather than
                // the data object itself.
                point.set_latitude(count);
                point.set_longitude(100);

                LOG_TRACE << "RecordRoute reuest# sending latitude " << count;

                return true;
            },

            // Callback to handle the completion of the request and its status/reply.
            [this](const grpc::Status& status, ::routeguide::RouteSummary& summery) mutable {
                if (!status.ok()) {
                    LOG_WARN << "RecordRoute request failed: " << status.error_message();
                    return;
                }

                LOG_TRACE << "RecordRoute request is done. Distance: "
                          << summery.distance();

                nextRequest();
            });
    }
```

在这个例子中，我们有一些逻辑来发送 `n` 条消息。但正如你所看到的，使用这个包装器还是相当简单的。

### **RouteChat**

最后是一个使用回调的双向流 RPC 的包装器。

```c++
/*! Definition of a callback function to provide the next outgoing message.
     *
     *  The function must return immediately.
     *
     *  \param msg Buffer for the data to send when the function returns.
     *  \return true if we are to send the message, false if we are done
     *      sending messages.
     */
    using on_say_something_cb_t = std::function<bool(::routeguide::RouteNote& msg)>;

    /*! Definition of a callback function regarding an incoming message.
     *
     *  The function must return immediately.
     */
    using on_got_message_cb_t = std::function<void(::routeguide::RouteNote&)>;

    /*! Definition of a callback function to notify us that the RPC is complete.
     *
     *  The function must return immediately.
     */
    using on_done_status_cb_t = std::function<void(const grpc::Status&)>;


    /*! Naive implementation of `RecordRoute` with callbacks for the events.
     *
     *  As before, we are prepared for a shouting contest, and will start sending
     *  message as soon as the RPC connection is established.
     *
     *  \param outgoing Callback function to provide new messages to send.
     *  \param incoming Callback function to notify us about an incoming message.
     *  \param done Callcack to inform us that the RPC is completed, and if
     *      it was successful.
     */
    void routeChat(on_say_something_cb_t&& outgoing,
                   on_got_message_cb_t&& incoming,
                   on_done_status_cb_t&& done) {

        class Impl
            // Our shared code for all the RPC's we implement here
            : Base
            // The async gRPC stream interface for this RPC
            , grpc::ClientBidiReactor<::routeguide::RouteNote,
                                      ::routeguide::RouteNote>
        {
        public:
            Impl(EverythingCallbackClient& owner,
                 on_say_something_cb_t& outgoing,
                 on_got_message_cb_t& incoming,
                 on_done_status_cb_t& done)
                : Base(owner), outgoing_{std::move(outgoing)}
                , incoming_{std::move(incoming)}
                , done_{std::move(done)} {

                LOG_TRACE << "routeChat starting async request.";

                // Glue this instance of this class to an initiation
                // of the RouteChat RPC.
                owner_.stub_->async()->RouteChat(&ctx_, this);

                // Start sending the first outgoing message
                read();

                // Start receiving the first incoming message
                write();

                // Here we will initiate the actual async RPC
                // The queued requests for read and write will be initiated.
                StartCall();

            }

            /*! Callback event when a write operation is complete */
            void OnWriteDone(bool ok) override {
                write();
            }

            /*! Callback event when a read operation is complete */
            void OnReadDone(bool ok) override {
                if (ok) {
                    incoming_(in_);
                    read();
                }
            }

            /*! Callback event when the RPC is complete */
            void OnDone(const grpc::Status& s) override {
                done_(s);
                delete this;
            }

        private:
            // Initiate a new async read operation
            void read() {
                in_.Clear();
                StartRead(&in_);
            }

            // Initiate a new async write operation (if appropriate).
            void write() {
                out_.Clear();
                if (outgoing_(out_)) {
                    return StartWrite(&out_);
                }

                StartWritesDone();
            }

            grpc::ClientContext ctx_;
            ::routeguide::RouteNote in_;
            ::routeguide::RouteNote out_;
            on_say_something_cb_t outgoing_;
            on_got_message_cb_t incoming_;
            on_done_status_cb_t done_;
        };

        // All this method actually does.
        new Impl(*this, outgoing, incoming, done);
    } // routeChat
```

这是使用包装器的示例：

```c++
/*! Example on how to use routeChat() */
    void nextRouteChat() {

        routeChat(
            // Compose an outgoing message
            [this, count=size_t{0}](::routeguide::RouteNote& msg) mutable {
                if (++count > config_.num_stream_messages) [[unlikely]] {
                    // We are done
                    return false;
                }

                // Say something thoughtful to make us look smart.
                // Something to print on T-shirts and make memes from ;)
                msg.set_message(std::string{"chat message "} + std::to_string(count));

                LOG_TRACE << "RouteChat reuest outgoing message " << count;

                return true;
            },
            // We received an incoming message
            [this](::routeguide::RouteNote& msg) {
                LOG_TRACE << "RouteChat reuest incoming message: " << msg.message();
            },
            // The conversation is over.
            [this](const grpc::Status& status) {
                if (!status.ok()) {
                    LOG_WARN << "RouteChat reuest failed: " << status.error_message();
                    return;
                }

                LOG_TRACE << "RecordRoute request is done.";
                nextRequest();
            }
            );
    }

```

现在，使用这个抽象，实际项目中使用双向流变得简单了。

我编写这些包装器是为了突出我认为是合适的“回调”接口暴露方式。

当我编写代码时，我尝试保持一个方法（或类）专注于一个任务。如果我编写一个将数据插入到数据库的方法，该方法应该负责数据的组成。该方法中的算法应该对任何查看代码的人来说都很清晰。这意味着同一个方法不应该处理数据库的细节，如打开句柄、处理连接或重新连接，并在所有返回路径中关闭句柄。当处理数据时，数据库操作应该限制在类似 `db.write(data);` 的操作上。

同样在使用 RPC 时。有些方法可能会详细说明如何使用 gRPC 的存根。但当我需要使用这些 RPC 时，我不想了解细节。我只需要实现我的算法所需的部分——例如一个事件处理程序来提供更多数据，以及一个事件处理程序来处理完成情况。

# 使用 Qt 的原生 gRPC 支持

我在 2023 年写了这系列文章的前 10 部分。2023 年 12 月，我开始开发一个我思考了十年的应用程序。这是一个个人组织器，采用“获取事物完成”方法加上一些其他想法。这个应用程序设计用于 PC 和移动设备，使用服务器来跟踪所有内容。我的想法是，我用 PC 的大屏幕和键盘来规划时间，然后在外出时切换到笔记本电脑或手机。因此，基本上它使用了事件驱动的 MVC 模式，其中模型驻留在服务器上。gRPC 对这种情况非常完美。我可以使用简单的 RPC 方法来添加和更改数据，并从服务器接收所有活动客户端设备的更改流。

我只是一个没有风险投资或大公司赞助的个人。所以我需要在 PC 和移动设备上重用 UI 应用程序的代码。为此，QT 很棒。它让我可以编写一个 C++ UI 应用程序，该程序可以在 PC（Linux、MacBook、Windows）和移动设备（Android、IOS）上运行，只需对不同平台进行少量调整。

过去我曾经玩过 QT 和 gRPC。在我花时间了解如何高效使用 gRPC 之前，那是个痛点——因为 QT 使用 QString 而 gRPC 使用 std::string。因此，除了处理通信外，我还必须在两个方向上进行数据转换。这真是不愉快！

我在 2023 年 12 月发现 QT 正在为他们的工具添加 gRPC 支持。他们已经有了自己的代码生成器和 CMake 宏，可以直接将 protobuf 序列化为 C++ 类，使用他们熟悉的类，如 QString 和 QList。当时几乎没有文档。不过，在“Qt 论坛”的帮助下，我能够使用它。

2024 年 6 月，他们发布了 QT 6.8-beta1，其中包含了更成熟的 gRPC 工具版本。如果你在 QT 应用程序中使用 gRPC，我强烈建议你仔细看看。这真的很棒，比标准的 protoc 生成的代码简单多了。

他们生成的 protobuf 类派生自 QObject。每个属性都是一个完整的属性，具有 getter 和 setter 以及通知更改的信号。它们可以直接在 QML 中使用。在编辑对话框中，在我自己的代码中，我现在得到一个从服务器接收到的数据实例，对其进行更改，然后将其发送回我的 C++ 代码，经过一些验证后再发送回服务器。这使得编写应用程序代码变得非常方便和快速。

所以，让我们详细看看它是如何工作的。我为 QT 使用他们的 gRPC 支持编写了一个新的“路线指南”接口示例。该应用程序有一个非常简单的 C++ 类；ServerComm 和一个简单的 QML UI。

## 让我们从Cmake开始

首先，我们必须告诉 CMake 我们将使用哪些模块。对于 gRPC，我们需要 Protobuf、ProtobufQuick 和 Grpc。

```c++
cmake_minimum_required(VERSION 3.16)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

project(QtGrpcExample VERSION 0.1.0 LANGUAGES CXX)

find_package(Qt6 6.8 REQUIRED COMPONENTS Core Gui Quick QuickControls2 Protobuf ProtobufQuick Grpc Concurrent)

qt_policy(
    SET QTP0001 NEW
)

qt_standard_project_setup()

set (protofile "${CMAKE_CURRENT_SOURCE_DIR}/../grpc/route_guide.proto")

qt_add_protobuf(RouteGuideLib
    QML
    QML_URI routeguide.pb
    PROTO_FILES ${protofile}
)

qt_add_grpc(GrpcClient CLIENT
    PROTO_FILES ${protofile}
)

qt_add_executable(${PROJECT_NAME}
    main.cpp
)

target_include_directories(${PROJECT_NAME}
    PRIVATE
    $<BUILD_INTERFACE:${FUN_ROOT}/include>
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
    $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
    $<BUILD_INTERFACE: ${CMAKE_BINARY_DIR}/generated-include>
    )

add_dependencies(${PROJECT_NAME}
    logfault
)

qt_add_qml_module(${PROJECT_NAME}
    URI appExample
    VERSION ${VERSION}
    QML_FILES
        Main.qml
    RESOURCES
    SOURCES
        ServerComm.h
        ServerComm.cpp
)


add_dependencies(${PROJECT_NAME} logfault GrpcClient RouteGuideLib)

target_link_libraries(GrpcClient
    PRIVATE
    RouteGuideLib
    Qt6::Core
    Qt6::Protobuf
    Qt6::Grpc
)

target_link_libraries(${PROJECT_NAME}
    PRIVATE
        GrpcClient
        RouteGuideLib
        Qt6::Core
        Qt6::Concurrent
        Qt6::Gui
        Qt6::Quick
        Qt6::QuickControls2
        Qt6::Protobuf
        Qt6::ProtobufQuick
        Qt6::Grpc
)

qt_import_qml_plugins(${PROJECT_NAME})
qt_finalize_executable(${PROJECT_NAME})
```

注意新添加的 `qt_add_protobuf` 和 `qt_add_grpc` 宏。对于应用程序，我们链接这些宏创建的目标。

在 `main.cpp` 文件中，我们像在其他 QML 应用程序中一样，启动 QML 引擎。

```c++

    QQmlApplicationEngine engine;
    engine.loadFromModule("appExample", "Main");
    if (engine.rootObjects().isEmpty()) {
        LOG_ERROR << "Failed to initialize engine!";
        return -1;
    }

    return app.exec();
```

`ServerComm` 类非常简单。它基本上是一个简单的启用了 QML 的类，具有一些属性和方法，我们可以从 QML 中调用这些方法来测试 "Route Guide" 中的四个 gRPC 方法。

```c++
class ServerComm : public QObject
{
    Q_OBJECT
    QML_ELEMENT
    QML_SINGLETON

    Q_PROPERTY(QString status READ status NOTIFY statusChanged)
    Q_PROPERTY(bool ready READ ready NOTIFY readyChanged)

public:
    ServerComm(QObject *parent = {});

    /*! Start the gRPC client.
    * This method is called from QML.
    *
    * We can call it again to change the server address or for example
    * if the server restarted.
    */
    Q_INVOKABLE void start(const QString& serverAddress);

    /*! Call's GetFeature on the server */
    Q_INVOKABLE void getFeature();
    Q_INVOKABLE void listFeatures();
    Q_INVOKABLE void recordRoute();
    Q_INVOKABLE void sendRouteUpdate();
    Q_INVOKABLE void finishRecordRoute();

    Q_INVOKABLE void routeChat();
    Q_INVOKABLE void sendChatMessage(const QString& message);
    Q_INVOKABLE void finishRouteChat();

```

然后，我添加了一个简单的模板，使调用普通（单一）gRPC 方法变得更加简单。

```c++
// Simple template to hide the complexity of calling a normal gRPC method.
// It takes a method to call with its arguments and a functor to be called when the result is ready.
template <typename respT, typename callT, typename doneT, typename ...Args>
void callRpc(callT&& call, doneT && done, Args... args) {
    auto exec = [this, call=std::move(call), done=std::move(done), args...]() {
        auto rpc_method = call(args...);
        rpc_method->subscribe(this, [this, rpc_method, done=std::move(done)] () {
                std::optional<respT> rval = rpc_method-> template read<respT>();
                    done(rval);
            },
            [this](QGrpcStatus status) {
                LOG_ERROR << "Comm error: " << status.message();
            });
    };

    exec();
}
```

最后，该类有一些成员用于 gRPC 流（连接）以及两个流。

```c++

routeguide::RouteGuide::Client client_;
std::shared_ptr<QGrpcClientStream> recordRouteStream_;
std::shared_ptr<QGrpcBidirStream> routeChatStream_;
```

在构造函数中，我们订阅了来自 gRPC 的通信错误。

```c++

ServerComm::ServerComm(QObject *parent)
    : QObject(parent)
{
    connect(&client_, &routeguide::RouteGuide::Client::errorOccurred,
            this, &ServerComm::errorOccurred);

}
```

在 `start()` 函数中，我们准备将客户端连接到我们在 UI 中指定的 gRPC 服务器。

我们本可以在下面的 `channelOptions` 中添加属性，例如，用于认证或会话信息。

```c++
void ServerComm::start(const QString &serverAddress)
{
    auto channelOptions = QGrpcChannelOptions{};

    client_.attachChannel(std::make_shared<QGrpcHttp2Channel>(
        QUrl(serverAddress, QUrl::StrictMode),
        channelOptions));
    LOG_INFO << "Using server at " << serverAddress;

    setReady(true);
    setStatus("Ready");
}
```

上面的代码有一个好处，就是如果你丢失了与服务器的连接，或者想连接到另一个服务器时，可以反复调用。

请注意，如果服务器地址无效或出现其他问题，直到我们调用 gRPC 方法之前不会报错。

下面是 `GetFeature`。它是异步的，完成 RPC 调用时会在主线程中调用 lambda 函数。

```c++
void ServerComm::getFeature()
{
    routeguide::Point point;
    point.setLatitude(1);
    point.setLongitude(2);

    callRpc<routeguide::Feature>([&] {
        LOG_DEBUG << "Calling GetFeature...";
        return client_.GetFeature(point); // Call the gRPC method
    }, [this](const std::optional<routeguide::Feature>& feature) {
        // Handle the result
        if (feature) {
            LOG_DEBUG << "Got Feature!";
            setStatus("Got Feature: " + feature->name());
        } else {
            LOG_DEBUG << "Failed to get Feature!";
            setStatus("Failed to get Feature");
        }
    });
}
```

要使用来自服务器的流，我们首先通过调用相应的 gRPC 方法创建一个异步流。然后，我们订阅 `QGrpcServerStream::messageReceived` 和 `QGrpcServerStream::finished` 信号。

```c++
void ServerComm::listFeatures()
{
    // Update the status in the UI.
    setStatus("...\n");

    // Prepare some data to send to the server.
    routeguide::Rectangle rect;
    rect.hi().setLatitude(1);
    rect.hi().setLatitude(2);
    rect.lo().setLatitude(3);
    rect.lo().setLatitude(4);

    // The stream is owned by client_.
    auto stream = client_.ListFeatures(rect);

    // Subscribe for incoming messages.
    connect(stream.get(), &QGrpcServerStream::messageReceived, [this, stream=stream.get()] {
        LOG_DEBUG << "Got message signal";
        if (stream->isFinished()) {
            LOG_DEBUG << "Stream finished";
            emit streamFinished();
            return;
        }

        if (const auto msg = stream->read<routeguide::Feature>()) {
            emit receivedMessage("Got feature: " + msg->name());
            setStatus(status_ + "Got feature: " + msg->name() + "\n");
        }
    });

    // Subscribe for the stream finished signal.
    connect (stream.get(), &QGrpcServerStream::finished, [this] {
        LOG_DEBUG << "Stream finished signal.";
        emit streamFinished();
    });
}
```

对于发送流，我们首先通过调用相应的 gRPC 方法来创建它。

我们还连接到 `QGrpcClientStream::finished` 信号，以便在我们告诉服务器完成发送消息后，获取服务器发送给我们的消息。

```c++

void ServerComm::recordRoute()
{
    // The stream is owned by client_.
    auto stream = client_.RecordRoute(routeguide::Point());
    recordRouteStream_ = stream;

    setStatus("Send messages...\n");

    connect(stream.get(), &QGrpcClientStream::finished, [this, stream=stream.get()] {
        LOG_DEBUG << "Stream finished signal.";

        if (auto msg = stream->read<routeguide::RouteSummary>()) {
            setStatus(status_ + "Finished trip with " + QString::number(msg->pointCount()) + " points\n");
        }
    });
}
```

然后，我们需要一个方法，让我们的 UI 可以调用它，以向服务器发送一条消息。

```c++
void ServerComm::sendRouteUpdate()
{
    routeguide::Point point;
    point.setLatitude(1);
    point.setLongitude(2);
    recordRouteStream_->writeMessage(point);
}
```

最后，对于 `RouteUpdate`，我们需要一个方法来告诉服务器我们已经完成。

```c++

void ServerComm::finishRecordRoute()
{
    recordRouteStream_->writesDone();
}
```

对于 `RouteChat`，我们结合了之前两个方法的模式。我们通过调用适当的 gRPC 方法来创建一个双向流。然后，我们订阅传入消息和完成信号。对于发送消息和结束聊天，我们实现了上述两个方法。

首先，我们创建流并订阅相关事件：

```c++
void ServerComm::routeChat()
{
    routeChatStream_ = client_.RouteChat(routeguide::RouteNote());

    connect(routeChatStream_.get(), &QGrpcBidirStream::messageReceived, [this, stream=routeChatStream_.get()] {
        if (const auto msg = stream->read<routeguide::RouteNote>()) {
            emit receivedMessage("Got chat message: " + msg->message());
            setStatus(status_ + "Got chat message: " + msg->message() + "\n");
        }
    });

    connect(routeChatStream_.get(), &QGrpcBidirStream::finished, [this] {
        LOG_DEBUG << "Stream finished signal.";
        emit streamFinished();
    });
}

```

然后，我们添加一个方法来发送消息。

```c++
void ServerComm::sendChatMessage(const QString& message)
{
    routeguide::RouteNote note;
    note.setMessage(message);
    routeChatStream_->writeMessage(note);
}
```

最后，我们添加一个方法来告诉服务器我们已经完成了聊天。

```c++
void ServerComm::finishRouteChat()
{
    routeChatStream_->writesDone();
}
```

我们实现中的错误处理器非常简单。它只是记录错误，将状态文本设置到 UI 中，并触发一个信号使 UI 禁用数据按钮。

```c++

void ServerComm::errorOccurred(const QGrpcStatus &status)
{
    LOG_ERROR << "errorOccurred: Call to gRPC server failed: " << status.message();
    setStatus(QString{"Error: Call to gRPC server failed: "} + status.message());
    setReady(false);
}

```

这就是基本用法的全部。在我看来，比起protoc生成的接口和存根，这要简单得多。

