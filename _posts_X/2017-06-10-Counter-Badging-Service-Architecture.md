---
layout: post
title: Counter-Badging Service Architecture 
tags: [MicroServices, Counters, Badging Service]
categories: [Counters, Kafka, Badging Service, MicroServices]
image:
  background: triangular.png
---

Most of the applications have some sort of badging functionality which is used to display certain counts to the users for CTA (Call To Action).

Once the user has clicked or acknowledged the counter then the value associated with the counter will be reset.

Given a system with large number of users and working off a large data set, it's not always possible to compute the counter values in real time.
Also, it may not make sense to update multiple records for different users when the counters are acknowledged.

There are three actions associated with the counters

1. Increment
2. Decrement   - This usually happens when an action is undone or something is deleted.
3. Reset       - When user acknowledges the counter.


All the above actions could happen concurrently and be associated with an user. For this reason, all the operations on the counter needs to be atomic else the counters would be off the correct value.

A common example for the badging/counter feature is the <b>Facebook</b> notification system.

The counters associated with the user are based on the actions performed by other users. 

An user can be associated with several other users and many of those other users could be performing the same or different actions at the same time. 
Like, there could be 3 people sending the user messages or another 4 people sending friend requests. 


<figure class="center half">
	<img src="/images/badging/fb-notifications.png" height="800px"></img>
	<figcaption><b>Facebook Notifications</b></figcaption>
</figure>

Given the above behavior of the system, the badging system can be architectured as below. The system would be event driven and the counter values eventually consistent.

Each microservice would generate events which gets listened by an events processor service. Based on the data in the event, the processor will update the
value of counters associated with the user.

<figure class="center">
	<img src="/images/badging/badging-architecture.png" height="800px"></img>
	<figcaption><b>Simple Badging/Counter Service Architecture</b></figcaption>
</figure>


Here is a simple example of the REDIS HASH data structure which can be used to store the counters associated with the user.
Each HASH is associated with a userId and all the counters are tied to the hash as key-value pairs.

<figure class="center">
	<img src="/images/badging/badging-redis-structure.png" height="800px"></img>
	<figcaption><b>Bading Redis Structure</b></figcaption>
</figure>
