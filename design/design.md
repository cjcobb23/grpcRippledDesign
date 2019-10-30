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

## Execution Concept

A class encapsulting the gRPC server is a member of `ApplicationImpl`, and is
constructed at the time of `ApplicationImpl` construction.  The gRPC server does
not start listening for requests yet.  In `Application::setup()`, the gRPC
server begins listening for requests, and an event loop begins running in a
separate, dedicated thread; this thread is used exclusively for servicing gRPC
requests, and is an additional thread used by the application, separate from the
threads used by the `JobQueue` or the I/O threads used by asio. In the setup of
the event loop, each rpc call (request type), is bound to the `CompletionQueue`;
when a request arrives, the request is placed on the `CompletionQueue`. The
event loop is an infinite loop that queries the `CompletionQueue` by calling
`CompletionQueue::Next()`, which returns the next event that has occurred,
blocking until such an event is available. The events, in our application, can
be one of two things:
* A request has arrived.
* A response has been sent.

When `CompletionQueue::Next()` returns a request, the event loop posts a
coroutine onto the `JobQueue`, to be invoked later; this coroutine, when invoked,
calls the corresponding handler, populates the response, and submits the
response for sending. In the meantime, the gRPC server continues to listen for
requests; if an additional request arrives before the first coroutine is
executed by the `JobQueue`, an additonal coroutine will be posted to the `JobQueue`.
The coroutines executed by the `JobQueue` are processed by a pool of worker
threads, so multiple coroutines (which are calling handlers) can be executed
concurrently. Furthermore, the gRPC server will continue to receive requests and
post coroutines to the `JobQueue`, irrespective of how many coroutines are waiting
in the `JobQueue` to be processed; in other words, requests are always accepted by
the gRPC server, regardless of how many requests are currently being processed.

The coroutines posted to the `JobQueue` populate the response and submit the
response for sending. Once gRPC actually sends
the response (gRPC handles the sending internally), an event is placed on the `CompletionQueue`; when
`CompletionQueue::Next()` returns such an event, signaling the response has been
sent, the event loop cleans up the resources associated with this
request/response.

While events are consumed from the `CompletionQueue` sequentially, responses are
populated and sent in parallel. This is because responses are populated and sent
from coroutines executed by the worker threads of the `JobQueue`, of which there are
several (meaning several coroutines could be executing in
parallel), and the event loop does not wait for the coroutines to execute in
order to post more coroutines corresponding to additional requests.
`CompletionQueue` is thread safe.

The `ResourceManager` of rippled will be used to track resource consumption per
endpoint. If the `ResourceManager` indicates that we should "disconnect" from
a specific endpoint, a coroutine will not be posted to the `JobQueue`, the
handler will not be called and an error will be returned to the client.

## Classes
The server code is adapted from the example found
[here](https://github.com/grpc/grpc/blob/v1.24.0/examples/cpp/helloworld/greeter_async_server.cc)

`CompletionQueue` is a class of the GRPC library. `CompletionQueue` is used to
queue events, and provides a method, `CompletionQueue::Next()` to query for any
events that have occurred. Events are returned by `CompletionQueue` in the form
of a `void **` pointer, known as a tag. In our application, `CompletionQueue` returns pointers to
`CallData` objects, which are used to service requests.

Each request is represented by a `CallData` object. `CallData` is an abstract
class that implements a method `Proceed()` and can be in one of three states:
LISTEN, PROCESS or FINISH. These states correspond to different states in the
lifecycle of an rpc call, and `Proceed()` transitions the object from one state
to the next. `CallData` has two pure virtual methods: `makeListener()` and
`process()`, which are implemented by derived classes. For
each rpc method, there is a class that derives from `CallData`, and implements
the pure virtual methods. `finish()` can be overridden if necessary. The
destructor is virtual so that `delete this` actually deletes the entire object,
including the derived class.

```
class CallData {

    CallData(CompletionQueue* cq) :cq_(cq), _status(LISTEN)
    {
        makeListener();
    }

    virtual ~CallData() {}
    void Proceed() {
      if (status_ == LISTEN) {
          status_ = PROCESS;
          process();
      } else {
          _status = FINISH;
          finish();
          delete this;
      }
    }

    virtual void makeListener() = 0;
    virtual void process() = 0;
    virtual void finish() {};

    CompletionQueue* cq_;
    enum CallStatus { LISTEN, PROCESS, FINISH};
    CallStatus status_;
}
```

* `makeListener()`, which is called upon object creation (from the constructor),
  creates a listener for the rpc method the object is meant to service, binding
that listener, along with a pointer to the object itself (`this`), to the `CompletionQueue` (each
`CallData` object has a pointer to the same `CompletionQueue`). The effect of
this listener binding is that when a request is received, the pointer to this `CallData` object
will be returned by `CompletionQueue::Next()`. Note that even though the
`CallData` object is in a LISTEN state, `CallData` itself is not listening, but
rather is associated with an internal gRPC listener for this particular request.
* `process()`, which is called after a request has been received, passes to the `JobQueue`
a coroutine that calls the appropriate handler. `process()` also creates
another `CallData` object to service any additional requests, since the current
object is busy processing a request. Note, `doProcess()` itself
does not execute the handler, but simply posts a coroutine to the `JobQueue` and
returns.
* `finish()`, is called after the response has been successfully sent, and
  performs any necessary cleanup or post processing. In most cases, no post
processing is necessary, so the base class provides a default implementation
that does nothing. After `finish()` is executed, the `CallData` object destroys
itself.

In the setup of the event loop, a `CallData` object for each rpc method is
created, which causes gRPC to listen for each type of request.
When an event occurs, `CompletionQueue::Next()` returns a pointer to a
`CallData` object as a tag, and the event loop calls `Proceed()` on the returned object. 
If the event was reception of a request, `Proceed()` will call `process()`,
which places a coroutine on the `JobQueue`, creates a new `CallData` object for
this request type, and returns immediately (the coroutine executes the handler, which
populates and sends the response, some time later); once the coroutine is
executed by the `JobQueue` and the response has been sent, a pointer to this `CallData` object will be placed back on
the `CompletionQueue`. When `CompletionQueue::Next()` returns a pointer to a `CallData`
object for which a response has been sent, `Proceed()` will call `finish()` and
then delete the object.

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

Note, each individual request is encapsulated by its own
`CallData` object. There is one `CallData` object per request type in the LISTEN
state at any given time; once a request is received, that `CallData` object is now
processing the request (in the PROCESS state), and a new `CallData` object is
created to service additional requests. There could be many `CallData` objects processing requests (same or
different type) at any one time, as well as many `CallData` objects waiting to
be destroyed (after the response has been sent), but there is only one
`CallData` object per rpc method in the LISTEN state at any given time.



So the general flow for a single `CallData` object is:
1. Create a `CallData` object for each rpc method, which calls `makeListener()` and 
binds a listener, along with a pointer to itself, to the `CompletionQueue`
2. When a request is received, `CompletionQueue::Next()` returns a pointer to a `CallData`
   object representing the request
3. `CallData::Proceed()` is called by the event loop, which calls `process()`.
This posts a coroutine to
   the `JobQueue` **and** creates a new `CallData` object to serve additional
requests
4. The coroutine is executed (eventually) by the `JobQueue`, populating the response and sending
   the response
5. Once the response has been successfully sent, a pointer to the `CallData` object is placed
   back on the `CompletionQueue`
6. `CompletionQueue::Next()` returns a pointer to the `CallData` object for which a response
   has been sent
7. `CallData::Proceed()` is called on the returned object, which calls
   `finish()` and destroys the object

Note that `3` creates a new `CallData` object to serve additional requests, which
begins an additional execution of this flow, executing in parallel to this flow.
Once the second flow is created, it does not wait for the first flow to finish,
or synchronize with the first flow in any way,
and could potentially finish executing before the first flow finishes.

There will be a handler for each gRPC method, separate from the existing json
handlers. The handler will take in the protobuf object that represents the
request, and return a protobuf object that represents the response. The protobuf
object will be a member of a templated class `RPC::ContextGeneric<T>`, where T
is the type of the protobuf object. `RPC::ContextGeneric<T>` is very similar to
`RPC::Context`, except the `params` member is of type `T` and there are no
`Headers`.

Below is an example handler signature; `xrp::v1` is the namespace of the
protobuf objects, whereas `GetFeeRequest` is the request type and `Fee` is the
response type.
```
xrp::v1::Fee doFee(RPC::ContextGeneric<xrp::v1::GetFeeRequest>& context);
```
Versioning for gRPC is usually done by changing the package name, which creates
protobuf objects in a new c++ namespace. For example, changing the package name
to `xrp.v2` results in objects with namespace `xrp::v2`. In the event
that a handler does not change between versions (which implies the objects are
the same in all but namespace), the old handler can be reused by templating the
handler signature like so:

```
template<class T, class R>
R doFee(RPC::ContextGeneric<T>& context);
```

## Interface
The interface to the GRPC service is defined by the .proto files. The `.proto`
files define the structure of the messages, the methods available, their inputs
and outputs, and the service itself.  GRPC, with the help of protobuf,
autogenerates code based on these `.proto` files, for both client and server.
GRPC has bindings in many common programming languages
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
    bytes address = 1;
  }

  message AccountInfo {
    XRPAmount balance = 1;
  
    uint32 sequence = 2;
  
  }
  message XRPAmount {
    // A numeric string representing the number of drops of XRP.
    bytes drops = 1;
  }
}
```
A client can do things like:
```
GetAccountInfoRequest request;
request.set_address("rLkMJhSVwhmummLjJPVrwQRZZYiYQhVQ1A");
//Send the request, receive populated response
uint32_t sequence = response.sequence();
std::string drops = response.balance.drops();
```
Note, the `bytes` type in a protobuf message is decoded as `std::string` in C++.

For more information about protocol buffers see
[here](https://developers.google.com/protocol-buffers/docs/overview).
For more information about gRPC and protocol buffers, see
[here](https://grpc.io/docs/guides/)

To support gRPC inside rippled, existing handlers will be mapped 1:1 to `.proto`
files and protobuf handlers; for each RPC method, there will exist a protobuf message
with the same fields and data that the existing handlers are expecting in JSON form,
as well as a handler that accepts this protobuf message.
Incrementally, all functionality of existing handlers (excluding deprecated
functionality) will be implemented by gRPC handlers.

The types of the protobuf messages will be derived from the types of the data in
the JSON used by existing RPC infrastructure. The types will be mapped according
to the below diagram:

| Json::ValueType | protobuf type |
| --------------- | ------------- |
| intValue | sint32 | |
| uintValue | uint32 | |
| realValue | double | |
| stringValue | bytes or string * |
| booleanValue | bool |
| arrayValue | repeated * |
| objectValue | message * |

Data that is currently represented as strings in the existing RPC
infrastructure will be represented as bytes, unless the data is truly text. JSON
arrays will be represented using the `repeated` keyword. A generic JSON object
will be represented as a distinct protobuf message.

## Sequence Diagrams
Below is a sequence diagram of handling two gRPC requests in parallel. In the
diagram, the responses are sent in the order the requests were received; this is
not a rule, and responses could be sent in a different order than the requests
were recieved.

![Alt text](https://g.gravizo.com/source/svg?https://raw.githubusercontent.com/cjcobb23/grpcRippledDesign/master/design/execution_sequence.plantuml)

## State Diagrams
Below is a state diagram for the CallData objects. Note, `Proceed()` is used to
advance the state and perform work, and is called from the event loop when the
appropriate event occurs

![Alt text](https://g.gravizo.com/source/svg?https://raw.githubusercontent.com/cjcobb23/grpcRippledDesign/master/design/calldata_state.plantuml)


