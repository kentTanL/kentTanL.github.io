---
layout: post
title: Microservices - Part 1
description: ""
tags: [Java, Microservices]
categories: [Java]
image:
  background: triangular.png
---

<figure class="half center">

<img src="/images/microservice/micro-service.png" height="400px"></img>

</figure>

Hello Everyone. This will be the first part of my multi-part series which will explain the concept of microservices and how to orchestrate calls between them keeping in mind things like performance, latency and scalability.

At the end of this series I intend to have a fully functional multi-tiered application which follows best practices for design and implementation.

I will concentrate on one component per post and also share the source code on Git for your learning purpose and reference.

 
# Concepts:  What is a Microservice ?

The traditional way of building web applications was to build one single artifact (say WAR file) which will have all your functionality wrapped inside of it. This single application will do everything from serving user requests, running reports, CRON job, interfacing with databases and other external third part services.

This kind of application is said to be Monolithic in nature.

Microservice based architecture emphasizes on not having a single deployable artifact but to have multiple smaller independently deployable artifacts.  Each smaller artifact deals with only part of the applications functionality. All modules work together to provide the functionality required of the application.
Advantages of Microservices over Monolitic

### 1. Faster Development

`Faster releases and changes out the door !!!`

Microservices consists of smaller independently deployable modules in comparison to a single deployable artifact in Monolithic architecture. This means that each application has its own code based. So, you can easily split your team into smaller functional teams and have them work independently on different modules without having each other step over one another or be blocked on some dependencies.

If at all there are any dependencies, those can be easily mocked and the work can be carried out without any interruption.    

### 2. Better Unit Testing

`More confidence with well tested code !!!!`

Since each module is just dealing with a subset of functionality provided by the application as a whole, it much easier to test manually or using automated tool. Its also easy for developers to write unit tests

### 3. Choice of Technology

`Keep your tech stack up to date !!!`

The main goal is for all modules to work well together. It is not necessary for all of the modules to leverage the same technology or frameworks/libraries. Each team can choose to implement the module they own in whatever technology they have expertise in. This also means that its easier to keep the applications tech stack up to date with contemporary library versions and dependencies while giving each team the independence to do what they want.

### 4. Scalability

`Scale independently, when and how you want !!!!`

Each module can be independently scaled up (vertical) or sideways (horizontal) to handle any spike in incoming requests. Modules performing any heavy lifting can scale vertically while those handing light weight stateless requests can scale horizontally.

Microservices help you scale out and back again once the traffic spike is gone.

### 5. Virtual Machine or Instance Configuration

`Save $$$ with infrastructure !!!`

With microservices you no longer need to have all your VM's or nodes have the same amount of resource.You are free to allocate more resource to more demanding modules and keep light weight modules on lean hardwares. This will eventually help bring down the computing/infrastructure cost.

### 6. Downtimes

`Less downtime impact !!!!`

Each module can be deployed independently without affecting the others. Also, if system is architectured well, the application can still be functional when one or more less important modules are brought down for maintenance  or due to some issue.

### 7. Monitoring

`More granular monitoring`

More precise monitoring of the applications. Depending on what tools you are using one can easily find out how each module is performing and how its contributing towards any request latency.

### 8. Easier problem isolation

`Save time !!!`

Its much easier to isolate a problem to a particular module. This saves time and energy and avoids you having to debug the entire application as a whole.
 
# Some drawback of Microservices

### 1. Problem isolation and troubleshooting

Since now you are dealing with a distributed system, it becomes difficult to isolate where something is breaking.It is easier to isolate the module but sometimes or most of the times the breaking module might have dependency on other modules for data. The incoming data might be the problem which could be due to an issue in the other module.

Each developer would have a higher responsibility to under the over all system and how different modules interact with each other.

### 2. Operational overhead

Many moudles and different teams means that there is always some deployment happening somewhere. There is an operational overhead to take care of the deployments and ensuring that the updated components play well with the other existing components.

### 3. Speed over duplication

If the application is large there is a very possibility of having duplication of code/functionality in different modules. But this is a compromise that one has to make when demanding faster development and shorter release cycles for an application.

 

 

 

 

 

 

 