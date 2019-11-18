---
layout: post
title: Hystrix, Turbine with Spring Boot Admin 
tags: [Java Microservices, Spring Boot Admin , Hystrix, Circuit Breaker, Turbine]
categories: [Hystrix, Microservices, Circuit Breaker, Spring Boot, Admin, Turbine]
image:
  background: triangular.png
---

<figure class="center half">
	<img src="/images/sb-admin/admin-logo.png" height="800px"></img>
</figure>

<a href="https://github.com/codecentric/spring-boot-admin">Spring Boot Admin</a> is a library which can be added to spring boot application to provide administrative capabilities.

You can look at the github pages of the 'Spring Boot Admin' project to see what various feature of the 'Spring Boot Admin' libraries are. 

In this article, I will focus on how Spring Boot Admin can be integrated with microservices supporting Hystrix dashboard. Spring Boot Admin provides single point of access to view dashboard of all registered services individually or aggregate all dashboard into a single view using Turbine.  


Once you add the Hystrix library to your micro-service, the library exposes the metrics through the <b>'/hystrix.stream'</b> endpoint on the service.
There's one stream per service. Clicking on the service in the spring boot admin will give you an easy access to the hystrix stream dashboard for that service. 
 
<figure class="center">
	<img src="/images/sb-admin/spring-boot-admin.png" height="800px"></img>
	<figcaption><b>Spring Boot Admin - Hystrix & Turbine</b></figcaption>
</figure>


<a href="https://github.com/Netflix/Turbine/wiki">Turbine</a> is another library from Netflix which helps aggregate multiple Hystrix stream and display them in a single dashboard.

In order to be able to aggregate multiple Hystrix stream, Turbine needs to be able to determine which services are currently available and where the services are. This is usually achieved by using a service discovery framework like Consul or Eureka.

Here, I am using an Eureka server. Each micro service has a Eureka client which registers with the Eureka server on startup. Turbine makes use of the instance discovery information provided by Eureka to establish connection to the microservice and read the Hystrix stream. The individual streams are passed through the Turbine aggregator so as to be able to render data in a single view.