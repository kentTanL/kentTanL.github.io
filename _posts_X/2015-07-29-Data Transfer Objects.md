---
layout: post
title: Data Transfer Objects
description: ""
tags: [Java]
categories: [Java]
image:
  background: triangular.png
---

# Data Transfer Objects

These are objects which are used to hold and transmit information between different processes. The processes could be different services within the same application or could be spread across multiple application in a distributed system. The application may be communicating with each other using Remote calls like RMI or web services (SOAP or REST).

Since communication between different applications/modules involves steps like opening connection, transmitting data over  the network  and closing it, which makes it an expensive operation.

Data transfer objects are meant as a way of aggregating all the information that needs to be transferred into a simple coherent entity and get it across to the other side without making too much of expensive operations.

 

Below diagram describes a scenario without the user of data transfer objects. In this case multiple calls are being made to the application service to retrieve the required information.

<figure class="half center">
<img src="/images/proto/proto-1.png" height="400px"></img>
<figcaption>Fig 1: Without Data Transfer Object</figcaption>
</figure>


With DTO ( data transfer objects) , the server will assemble all the required information into a single entity and send it back to the client. The client can use what ever fields it needs and simply ignore the other fields or store them for use at another time. Even though the payload returned back is larger, it does definitely reduce the amount of request the server needs to handle for an operation and also reduces any latency associated with data transmission over the internet.

<figure class="half center">
<img src="/images/proto/proto-2.png" height="400px"></img>
<figcaption>Fig 2: Using Data Transfer Object</figcaption>
</figure>
