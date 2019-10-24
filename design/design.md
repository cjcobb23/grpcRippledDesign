# Native support for GRPC in rippled

## Overview
GRPC is an RPC system that allows client applications to directly call methods
on a server application as if it was a local object. gRPC uses protocol buffers
as its Interface Definition Language, as well as the underlying message
interchange format. gRPC has bindings in many languages. gRPC uses the protocol
buffer compiler (`protoc`) with the gRPC plugin to generate data access classes
as well as client and server code. This generated code handles serialization,
deserialization and I/O. The messages and methods are defined in `.proto` files
that are used by both client and server. For more information about gRPC and
protocol buffers, go [here](https://grpc.io/docs/).

Currently, rippled supports RPC over Web Socket, REST-JSON and CLI. gRPC would
be in addition to these existing RPC technologies. The reason for adding gRPC
support to rippled is that gRPC is very easy to use as a client, and has
bindings in many languages, allowing developers to more quickly develop
applications without worrying about small details and nuances related to RPC.

## ExecutionConcept

In `Application::setup()`, a grpc server begins listening for requests,
and an event loop begins running in a separate, dedicated
thread; this thread is used exclusively for servicing grpc requests, and is an
additional thread used by the application, separate from the threads used by the
JobQueue or the I/O threads used by asio. In the setup of the event loop, each rpc 
call (request type), is bound to the `CompletionQueue`; when a request arrives,
the request is placed on the `CompletionQueue`. The event loop is an infinite
loop that queries the `CompletionQueue` by calling `CompletionQueue::Next()`,
which returns the next event that has occurred, blocking until such an event is
available. The events, in our application, can be one of two things:
* A request has arrived.
* A response has been sent.

When `CompletionQueue::Next()` returns a request, the event loop posts a
coroutine onto the JobQueue, to be invoked later; this coroutine, when invoked, 
calls the corresponding handler, populates the response, and submits the response for sending.
Posting to the JobQueue in this way frees up the event loop to process more events
while requests are being handled. Once grpc sends the response, an event is
placed on the `CompletionQueue`; when `CompletionQueue::Next()` returns such an
event, signaling the response has been sent, the event loop cleans up the resources
associated with this request/response.

While events are consumed from the `CompletionQueue` sequentially, responses are
populated and sent in parallel. `CompletionQueue` is thread safe. It is possible
to have multiple event loops running in parallel (one `CompletionQueue` and one
thread per event loop), though this may not necessarily lead to better
scalablity, since handlers are called from the JobQueue.

## Classes
The server code is adapted from the example found
[here](https://github.com/grpc/grpc/blob/v1.24.0/examples/cpp/helloworld/greeter_async_server.cc)

`CompletionQueue` is a class of the GRPC library. `CompletionQueue` is used to queue events, and provides a method, `CompletionQueue::Next()` to query for any events that have occurred. Events are returned by `CompletionQueue` in the form of a `void *` pointer. In our application, `CompletionQueue` returns pointers to `CallData` objects, which are used to service requests.

Each request is represented by a `CallData` object. `CallData` is an abstract
class that implements a method `Proceed()` and can be in one of three states:
CREATE, PROCESS or FINISH. These states correspond to different states in the lifecycle of an rpc call, and `Proceed()` transitions the object from one
state to the next. `CallData` has three pure virtual methods:
`doCreate()`, `doProcess()` and `doFinish()`, which are implemented by derived
classes. For each rpc method, there is a class that derives from `CallData`, and
implements the pure virtual methods.

```
class CallData {
    void Proceed() {
      if (status_ == CREATE) {
          status_ = PROCESS;
          doCreate();
      } else if (status_ == PROCESS) {
          status_ = FINISH;
          doProcess();
      } else {
          GPR_ASSERT(status_ == FINISH);
          doFinish();
      }
    }

    virtual void doCreate() = 0;
    virtual void doProcess() = 0;
    virtual void doFinish() = 0;

    CompletionQueue* cq_;
    enum CallStatus { CREATE, PROCESS, FINISH};
    CallStatus status_;
}
```

* `doCreate()`, which is called upon object creation (from the constructor), creates a listener for the
rpc method the object is meant to service, binding that listener, along with the object itself, 
to the `CompletionQueue` (each `CallData` object has a pointer to the same `CompletionQueue`).
* `doProcess()`, which is called when a request is received, passes to the JobQueue
a coroutine that calls the appropriate handler. `doProcess()` also creates
another `CallData` object to listen for this request type, since this object
is processing a response and no longer listening.
* `doFinish()`, which is called when the response has been sent,
destroys `this` object

In the setup of the event loop, a `CallData` object for each rpc method is
created. When an event occurs, `CompletionQueue::Next()` returns a pointer to a
`CallData` object, and the event loop calls `Proceed()` on the returned object. 
If the event was reception of a request, `Proceed()` will call `doProcess()`. If
the event was a response was sent, `Proceed()` will call `doFinish()`.

```
  void EventLoop() {
    // Spawn new CallData instances to serve new clients.
    new AccountInfoCallData(&service_, completion_queue_.get(), app_);
    new FeeCallData(&service_, completion_queue_.get(), app_);
    ///... Additional CallData instances for additional rpc methods

    void* tag;  // uniquely identifies a request.
    bool ok;
    while (true) {
      // Block waiting to read the next event from the completion queue. The
      // event is uniquely identified by its tag, which is the memory address
      // of a CallData instance.
      completion_queue_->Next(&tag, &ok);
      static_cast<CallData*>(tag)->Proceed();
    }
  }
```

There will be a handler for each grpc method, separate from the existing json
handlers. The handler will take in the protobuf object that represents the
request, and return a protobuf object that represents the response. The protobuf
object will be a member of a templated class `RPC::ContextGeneric<T>`, where T
is the type of the protobuf object. `RPC::ContextGeneric<T>` is very similar to
`RPC::Context`, except the `params` member is of type `T` and there are no
`Headers`.

Below is an example handler signature; `io::xpring` is the namespace of the
protobuf objects, whereas `GetFeeRequest` is the request type and `Fee` is the
response type.
```
io::xpring::Fee doFee(RPC::ContextGeneric<io::xpring::GetFeeRequest>& context);
```
Versioning for grpc is usually done by changing the package name, which creates
protobuf objects in a new c++ namespace. For example, changing the package name
to `io.xpring.v2` results in objects with namespace `io::xpring::v2`. In the event
that a handler does not change between versions (which implies the objects are
the same in all but namespace), the old handler can be reused by templating the
handler signature like so:

```
template<class T, class R>
R doFee(RPC::ContextGeneric<T>& context);
```
## Use Case
The GRPC service will be used in a request response format (as opposed to streaming).
The client will issue a request, and then receive a response.

An application developer can quickly write client code in their language of
choice using the `.proto` files and the code generation tools of gRPC.

## Interface
The interface to the GRPC service is defined by the .proto files. The `.proto` files
define the structure of the messages, the methods available, their inputs and outputs, and the service itself.
GRPC, with the help of protobuf, autogenerates code based on these `.proto` files,
for both client and server. GRPC has bindings in many common programming languages
([link](https://grpc.io/docs/reference/)).
A client of the GRPC service we are building would use the exact same `.proto`
files as the server, to generate the client code. The generated code (both
client and server), handles serialization, deserialization and I/O. Protobuf
allows one to define messages with named and typed fields, and allows for user
defined types. Client and server code can then manipulate the generated objects using the
message names, field names and types defined in the `.proto` file. For example,
using the below example `.proto` file:
```
service XRPLedgerAPI {
  // Get account info for an account on the XRP Ledger.
  rpc GetAccountInfo (GetAccountInfoRequest) returns (AccountInfo);

  // Get the fee for a transaction on the XRP Ledger.
  rpc GetFee (GetFeeRequest) returns (Fee);
  message GetAccountInfoRequest {
    // The address to get info about.
    string address = 1;
  }

  message AccountInfo {
    XRPAmount balance = 1;
  
    uint64 sequence = 2;
  
  }
  message XRPAmount {
    // A numeric string representing the number of drops of XRP.
    string drops = 1;
  }


}
```
A client can do things like:
```
GetAccountInfoRequest request;
request.set_address("rLkMJhSVwhmummLjJPVrwQRZZYiYQhVQ1A");
//Send the request, receive populated response
uint64_t sequence = response.sequence();
std::string drops = response.balance.drops();
```

For more information about protocol buffers see
[here](https://developers.google.com/protocol-buffers/docs/overview).
For more information about grpc and protocol buffers, see
[here](https://grpc.io/docs/guides/)

## Sequence Diagrams?
Below is a sequence diagram of handling two grpc requests in parallel. In the diagram, the responses are sent in the order the requests were received; this is not a rule, and responses could be sent in a different order than the requests were recieved.

![Alt text](https://g.gravizo.com/source/svg?https://raw.githubusercontent.com/cjcobb23/grpcRippledDesign/master/design/execution_sequence.plantuml)

## State Diagrams?
Below is a state diagram for the CallData objects. Note, `Proceed()` is used to advance the state and perform work, and is called first from the constructor, and subsequently from the event loop, when the appropriate event occurs.

![Alt text](https://g.gravizo.com/source/svg?https://raw.githubusercontent.com/cjcobb23/grpcRippledDesign/master/design/calldata_state.plantuml)
## Future work?
* TLS
* Streaming?

## Reverse Engineering

