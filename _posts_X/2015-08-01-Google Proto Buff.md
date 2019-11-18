---
layout: post
title: Google Protobuff as Data Transfer Object
description: ""
tags: [Java]
categories: [Java]
image:
  background: triangular.png
---

<figure class="half center">
<img src="/images/proto/google-protocol-buffers.jpg" height="400px"></img>
</figure>

# Why use Protos ?

1. Enforces a contract between sender and receiver services by using its own schema specification language. The PROTOC (Proto compiler) can translate the schema into language specific objects for use in your application.

2. The size of the payload resulting from Protobuff encoding is smaller in comparison to how it would have been if the data were transmitted as JSON or XML.

3. The Protobuff schema is backward compatible. This means that two services using objects generated from different versions of ProtoBuff schema can still send and receive information between them without resulting in any runtime exceptions. The Protobuff framework takes care of serializing fields which match the ones defined in its schema version and ignore any that don't match. 

4. Field level validation and extensibility. Validation for field types, values using Enum and ordinality can be easily defined in the schema. Also one can easily import the schema into other schemas to create new structures.

5. Language interoperability. Since the payload is bytes, the sender and receiver sharing the schema can be implemented in any language. You can have a Java Webservice talk to a different server written in C++.


# When to use Protos 

1. Protos makes sense for back-end system to system services where there is no need to have visibility of the payload being transmitted.

2. Ideal for microservices architecture where each service is deployed independent of each other. The backward compatability of the schemas ensures that different service can work well while still using different versions of the library.


# Protos are not ideal for

1. Front end to backend communication. The payload in case of protos is byte data. It's easier for the front-end to receive a JSON or XML directly than encoded bytes.

2. When large HTML or XML data needs to be transmitted.

3. Want responses in human understandable format.

4. When you don't want to have a contract for communication.








