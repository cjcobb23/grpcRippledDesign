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

Each RPC is represented by a `CallData` object. `CallData` is a templated
class, in which the template parameters are the request type and response type.
The constructor to CallData takes in two function pointers (as well as other
data); one of these function pointers is used to create a listener for the specific
RPC that this `CallData` represents (the function pointed to is a codegen'd GRPC
call), and the other is used process a request that has arrived (the function
pointed to is a handler). The function to create a listener is called in the
constructor of `CallData`.

All `CallData` objects derived from an abstract class `Processor`. Processor has
several pure virtual methods. These pure virtual methods are called from the
event loop.

```
class Processor
{
    public:

    virtual ~Processor() {}

    // process a request that has arrived. Can only be called once per instance
    virtual void process() = 0;

    //store an iterator to this object
    //all Processor objects are stored in a std::list as shared_ptrs, and the 
    //iterator points to that object's position in the list. When object finishes
    //processing a request, the iterator is used to delete the object from the list
    virtual void set_iter(
            std::list<std::shared_ptr<Processor>>::iterator const& it) = 0;

    //get iterator to this object. see above comment
    virtual std::list<std::shared_ptr<Processor>>::iterator get_iter() = 0;

    //abort processing this request. called when server shutsdown
    virtual void abort() = 0;

    //create a new instance of this CallData object, with the same type
    //(same template parameters) as original. This is called when a CallData
    //object starts processing a request. Creating a new instance allows the 
    //server to handle additional requests while the first is being processed
    virtual std::shared_ptr<Processor> clone() = 0;

    //true if this object has finished processing the request. Object will be
    //deleted once this function returns true
    virtual bool isFinished() = 0;
};
```

The event loop queries the `CompletionQueue` for the next event that has
occurred. `CompletionQueue::Next()` is a blocking call. When a request arrives,
`CompletionQueue::Next()` sets the `tag` variable to the `Processor` object that
is handling the request. The event loop calls `Processor::process()` to handle
the request (`process()` simply posts a coroutine to the `JobQueue` and
returns). In addition to calling `process()`, the event loop also calls
`Processor::clone()` to create a new listener; this allows the gRPC server to
handle additional requests while the response to the first request is being
populated. When the coroutine executes, a response is submitted to the
`CompletionQueue` for sending. Once the response is actually sent,
`CompletionQueue::Next()` will set the `tag` variable to the `Processor` object
that handled the request. At this point, `Processor::isFinished()` will return
true, and the object will be deleted. See the next paragraph on lifetime
management for more details about the deletion.
```
//event loop
void GRPCServerImpl::handleRpcs() {
    void* tag;  // uniquely identifies a request.
    bool ok;
    // Block waiting to read the next event from the completion queue. The
    // event is uniquely identified by its tag, which in this case is the
    // memory address of a CallData instance.
    // The return value of Next should always be checked. This return value
    // tells us whether there is any kind of event or cq_ is shutting down.
    while (cq_->Next(&tag,&ok)) {

        //if ok is false, event was terminated as part of a shutdown sequence
        //need to abort any further processing
        if(!ok)
        {
            //abort first, then erase. Otherwise, erase can delete object
            static_cast<Processor*>(tag)->abort();
            requests_.erase(static_cast<Processor*>(tag)->get_iter());
        }
        else
        {
            auto ptr = static_cast<Processor*>(tag);
            if(!ptr->isFinished())
            {
                //ptr is now processing a request, so create a new CallData
                //object to handle additional requests
                auto cloned = ptr->clone();
                requests_.push_front(cloned);
                //set iterator as data member for later lookup
                cloned->set_iter(requests_.begin());
                //process the request
                ptr->process();
            }
            else
            {
                //rpc is finished, delete CallData object
                requests_.erase(static_cast<Processor*>(tag)->get_iter());
            }
        }
    }
}
```

`requests_` is of type `std::list<std::shared_ptr<Processor>>`. This list
contains each `CallData` object that is waiting for a request (there are
always `n` CallData objects waiting for a request at a time, where `n` is the number of
RPCs). This list also contains every `CallData` object that is currently
handling a request (could be many), as well as any `CallData` objects that have
sent a response and are waiting to be deleted. Each `CallData` object is added
to this list upon creation. After the `CallData` object is added to the list,
we also retrieve an iterator that points to the `CallData` object in the list,
and store that iterator as a data member of the `CallData` object. When the
`CallData` object is ready for deletion, we use the iterator to remove the
`CallData` object from the list.

Special care must be taken to handle shutting down the server. When the server
shuts down, all pending events are immediately cancelled. The way this works is
that the gRPC runtime adds every outstanding `tag` (pointer to `Processor` in
our application) to the `CompletionQueue`.
This cancels any current listeners, and cancels any responses queued for
sending. `CompletionQueue::Next()` will return each of these pointers, and also
sets the `ok` variable to false (meaning the event was cancelled). For every
cancelled event, we erase the `Processor` object from the list. However, there
exists a race condition, due to the handler function being executed from the
`JobQueue`. When the event is cancelled, the coroutine posted to the `JobQueue`
will still execute, if it has not already executed. To make sure that the object
is not deleted in the middle of handler execution, the coroutine captures a
shared_ptr to itself (via `shared_from_this()`). In this manner, when the event
loop erases the `Processor` shared_ptr from the `requests_` list, the object
will not be deleted if the coroutine is still waiting to be executed. The event
loop also calls `abort()` on the `Processor` object, which simply sets a member
boolean member variable to signal abortion. When the coroutine executes, it
first locks a mutex and checks if the RPC has been aborted. If so, the coroutine
immediately returns and does not populate a response. If the `Processor` object
has already been erased from the list, returning from the coroutine will cause
the `Processor` object to be deleted.

Note, there is no contention for this mutex unless the server is shutting down.
Also, the mutex acts as a memory barrier to force cache coherency of the
`aborted_` and `status_` members.

```

template <class Request, class Response>
void GRPCServerImpl::CallData<Request, Response>::process()
{
    if (status_ == PROCESSING) {
        std::shared_ptr<CallData<Request,Response>> this_s =
            this->shared_from_this();
        app_.getJobQueue().postCoro(JobType::jtRPC, "gRPC-Client",
                [this_s](std::shared_ptr<JobQueue::Coro> coro)
                {
                    std::lock_guard<std::mutex> lock(this_s->mut_);

                    //Do nothing if call has been aborted due to server shutdown
                    if(this_s->aborted_)
                        return;

                    this_s->process(coro);
                    this_s->status_ = FINISH;
                });
    }
}


```

So the general flow for a single `CallData` object is:
1. Create a `CallData` object for each rpc method, and add it as a `shared_ptr<Processor>`
to the `requests_` list. Creation of the `CallData`object creates a listener for
associated RPC.
2. When a request is received, `CompletionQueue::Next()` returns a pointer to a `Processor`
   object representing the request
3. `Processor::process()` is called by the event loop.
This posts a coroutine to
   the `JobQueue`.  The event loop also calls `Processor::clone()` to create a new 
`CallData` object, with the same template parameters, to serve additional requests.
4. The coroutine is executed (eventually) by the `JobQueue`, populating the response and sending
   the response
5. Once the response has been successfully sent, a pointer to the `Processor` object is placed
   back on the `CompletionQueue`
6. `CompletionQueue::Next()` returns a pointer to the `Processor` object for which a response
   has been sent
7. The `Processor` object is erased from the `requests_` list, causing the object
   to be deleted. 

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

Below is an example handler signature; `rpc::v1` is the namespace of the
protobuf objects, whereas `GetFeeRequest` is the request type and `GetFeeResponse` is the
response type.
```
std::pair<grpc::Status,rpc::v1::GetFeeResponse> doFee(RPC::ContextGeneric<rpc::v1::GetFeeRequest>& context);
```
Versioning for gRPC is usually done by changing the package name, which creates
protobuf objects in a new c++ namespace. For example, changing the package name
to `rpc.v2` results in objects with namespace `rpc::v2`. In the event
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
using the below example `.proto` file (this is just an example, not the real
service definition):
```
service XRPLedgerAPIService {
  // Get account info for an account on the XRP Ledger.
  rpc GetAccountInfo (GetAccountInfoRequest) returns (GetAccountInfoResponse);

  // Get the fee for a transaction on the XRP Ledger.
  rpc GetFee (GetFeeRequest) returns (GetFeeResponse);

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


