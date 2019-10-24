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
## State Diagrams?
## Future work?
* TLS
* Streaming?

## Reverse Engineering

