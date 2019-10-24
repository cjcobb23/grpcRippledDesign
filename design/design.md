# Native support for GRPC in rippled

## Overview
What is grpc?

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

Each request is represented by a `CallData` object. `CallData` is an abstract
class that implements a method `Proceed()` and can be in one of three states:
CREATE, PROCESS or FINISH. `Proceed()` transitions the object from one
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

Below is an example handler signature; io::xpring is the namespace of the
protobuf objects, whereas `GetFeeRequest` is the request type and `Fee` is the
response type.
```
io::xpring::Fee doFee(RPC::ContextGeneric<io::xpring::GetFeeRequest>& context);
```
Versioning for grpc is usually done by changing the package name, which creates
protobuf objects in a new c++ namespace. For example, changing the package name
to io.xpring.v2 results in objects with namespace io::xpring::v2. In the event
that a handler does not change between versions (which implies the objects are
the same in all but namespace), the old handler can be reused by templating the
handler signature like so:

```
template<class T, class R>
R doFee(RPC::ContextGeneric<T>& context);
```

An abstract class CallData implements a method Proceed(), which transitions from
one state to the next. There are three states, CREATE, PROCESS, FINISH, and
progression goes from CREATE to PROCESS to FINISH. the methods
doCreate(),doProcess() and doFinish() are pure virtual and implemented by
subclasses. Each subclass corresponds to a particualr grpc request, The sublcass
objects are in the create state upon construction and call process() in the
ctor, 
        // As part of the initial CREATE state, we *request* that the system
        // start processing GetAccountInfo requests. In this request, "this" acts are
        // the tag uniquely identifying the request (so that different CallData
        // instances can serve different requests concurrently), in this case
        // the memory address of this CallData instance.


**Just copy documentation from call data**

the event loop calls process for each object
GRPC classes (CompletionQueue)
## Use Case
Request-Response
## Interface
.proto files
## Sequence Diagrams?
grpc server starts listening on socket
client connects, hand data to job queue
wait for next request
eventually, job queue populates response, puts on completionqueue
event loop of completionqueue sends response
object destroyed

![Alt text](https://g.gravizo.com/source/svg?https://raw.githubusercontent.com/cjcobb23/grpcRippledDesign/design/execution_sequence.plantuml)


![Alt text](https://g.gravizo.com/source/custom_mark14?https%3A%2F%2Fraw.githubusercontent.com%2FTLmaK0%2Fgravizo%2Fmaster%2FREADME.md)
<details> 
<summary></summary>
custom_mark14
@startuml;
actor Client;
participant "GRPC Server" as G;
participant "JobQueue" as JQ;
Client -> G: Send Request;
G -> JQ : Post Coroutine
Client -> G: Send Request;
G -> JQ : Post Coroutine
JQ -> Client : Execute Coroutine and Send Response
JQ -> Client : Execute Coroutine and Send Response
`
@enduml
custom_mark13
</details>

## State Diagrams?
## Future work?
* TLS
* Streaming?

## Reverse Engineering

