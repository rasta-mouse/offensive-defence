---
title: "gRPC Attack Surface"
date: 2021-03-26T11:33:01Z
draft: false
authors:
    - rastamouse
tags:
    - grpc
    - web
---

In this post we'll look at methods to analyse the attack surface of gRPC services.  It won't focus on specific vulnerabilities.

## (Google) Remote Procedure Calls

gRPC is an open source RPC system developed at Google.  You can think of gRPC services like APIs, although there are many key differences.  Where APIs typically use JSON or XML, gRPC uses protocol buffers (protobufs) to serialise data being sent between a client and server.  The way this serialisation happens is very strict.

Take this C# model as an example:

```c#
public class Customer
{
    public string FirstName { get; set; }
    public string LastName { get; set;}
}
```

A JSON serialiser may produce an output that looks like this:

```c#
var customer = new Customer { FirstName = "Rasta", LastName = "Mouse" };
var json = JsonConvert.SerializeObject(customer);

Console.WriteLine(json);

result => {"FirstName":"Rasta","LastName":"Mouse"}
```

However, there's nothing stopping it re-ordering the properties.  Even if it did so, it can still be deserialised back into a Customer object:

```c#
var json = @"{""LastName"":""Mouse"",""FirstName"":""Rasta""}";
var customer = JsonConvert.DeserializeObject<Customer>(json);

Console.WriteLine($"{customer.FirstName} {customer.LastName}");

result => Rasta Mouse
```

A gRPC protbuf not only defines the data properties, but also the order in which they should appear.  A code-first gRPC contract may look something like this:

```c#
[DataContract]
public class Customer
{
    [DataMember(Order = 1)]
    public string FirstName { get; set; }

    [DataMember(Order = 2)]
    public string LastName { get; set; }
}
```

Unless you have existing knowledge of this contract, it's very unlikely that you'll be able to get anything useful from the service.

gRPC Reflection is an optional extension for servers, which provides the protobuf contracts in human-readable format.  The primary use case is to aid in client-side tool debugging - think of gRPC Reflection as Swagger for APIs.

[gRPCurl](https://github.com/fullstorydev/grpcurl) is a CLI tool for interacting with gRPC services.

If reflection is not enabled, then you're pretty much out of luck.

```text
PS C:\> grpcurl -insecure localhost:8443 list
Failed to list services: server does not support the reflection API
```

Otherwise, you can start listing and describing the services.

```text
PS C:\> grpcurl -insecure localhost:8443 list
greet.Greeter
grpc.reflection.v1alpha.ServerReflection
```

`greet.Greeter` is the only service - to get the contract for it, use `grpc describe`.

```text
PS C:\> grpcurl -insecure localhost:8443 describe greet.Greeter
greet.Greeter is a service:
service Greeter {
  rpc SayHello ( .greet.HelloRequest ) returns ( .greet.HelloReply );
}
```

This service has a single method called `SayHello`.  It takes in a `HelloRequest` and returns a `HelloReply`.  For which we can also get the data members.

```text
PS C:\> grpcurl -insecure localhost:8443 describe greet.HelloRequest
greet.HelloRequest is a message:
message HelloRequest {
  string name = 1;
}

PS C:\> grpcurl -insecure localhost:8443 describe greet.HelloReply
greet.HelloReply is a message:
message HelloReply {
  string message = 1;
}
```

So `HelloRequest` requires a single `string` property called `name`; and `HelloRequest` returns a single `string` property called `message`.

Now we have enough information to call the service.

```text
PS C:\> grpcurl -insecure -d '{\"name\":\"Rasta\"}' localhost:8443 greet.Greeter.SayHello
{
  "message": "Hello Rasta"
}
```

Note that even the property names are case-sensitive.

```text
PS C:\> grpcurl -insecure -d '{\"Name\":\"Rasta\"}' localhost:8443 greet.Greeter.SayHello
Error invoking method "greet.Greeter.SayHello": error getting request data: message type greet.HelloRequest has no known field named Name
```

Hopefully this gives you some helpful pointers if you ever come across gRPC services with relfection enabled.