# Requirements for native support of GRPC by rippled
## Overview
Motivation of this project is to have rippled natively support GRPC. GRPC will be supported in addition to the REST-JSON, Web Socket and Command Line Interface, and is not meant to replace those interfaces.

Each numbered item below corresponds to a requirement; a corresponding *Note* is not a requirement, but simply clarification.

## Requirements List

1. **GRPC server will listen for requests in a single dedicated thread, in the same process as rippled**
    - *Note: this thread will solely be for I/O*
2. **Request processing, response population and other application logic will be done via a coroutine run by the JobQueue.**
    - *Note: as little work as possible will be done by the dedicated GRPC I/O thread, and as much work as possible will be done by the JobQueue*
3. **rippled will serve multiple grpc requests concurrently**
4. **Each request will have a corresponding input and output message defined in .proto files**
5. **Each parameter of a grpc request will have a corresponding named and typed field in the protobuf request message**
    - *Note: this prohibits passing a blob of json in a protobuf message*
6. **Each data member of a grpc response will have a corresponding named and typed field in the protobuf response message**
    - *Note: again prohibits returning json blob*
    - *Note: 4,5 and 6 imply that the input and output of each grpc method is strictly defined*
7. **The first iteration of GRPC implementation will implement `account_info`, `fee` and `submit`**
    - *Note: GRPC calls will mirror the functionality of existing RPC calls, except deprecated functionality*
