# Requirements for native support of GRPC by rippled
## Overview
Motivation of this project is to have rippled natively support GRPC. GRPC will be supported in addition to the REST-JSON, Web Socket and Command Line Interface, and is not meant to replace those interfaces.

Each numbered item below corresponds to a requirement; a corresponding *Note* is not a requirement, but simply clarification.

## Requirements List

1. **gRPC server shall run in the same process as rippled**
2. **rippled shall serve multiple grpc requests concurrently**
3. **Each request shall have a corresponding input and output message defined in .proto files**
4. **Each parameter of a grpc request shall have a corresponding named and typed field in the protobuf request message**
    - *Note: this prohibits passing a blob of json in a protobuf message*
5. **Each data member of a grpc response shall have a corresponding named and typed field in the protobuf response message**
    - *Note: again prohibits returning json blob*
    - *Note: 3,4 and 5 imply that the input and output of each grpc method is strictly defined*
6. **gRPC calls shall mirror the functionality of existing RPC calls, except deprecated functionality**

## Requirements Table

| Requirement | Comments |
| ----------- | ----------- |
| 1. gRPC server shall run in the same process as rippled | |
| 2. rippled shall serve multiple gRPC requests concurrently| |
| 3. Each request shall have a corresponding input and output message defined in .proto files | |
| 4. Each parameter of a grpc request shall have a corresponding named and typed field in the protobuf request message | *Note: this prohibits passing a blob of json in a protobuf message* |
| 5. Each data member of a grpc response shall have a corresponding named and typed field in the protobuf response message | *Note: prohibits returning blob of json. 3, 4 and 5 imply that the input and output of each grpc method is strictly defined*|
| 6. gRPC methods shall mirror the functionality of existing RPC calls, except deprecated functionality | |
