---
layout: post
title: Implementing CHAT system with MongoDB Atlas
tags: [MongoDB]
categories: [MongoDB]
image:
  background: triangular.png
---

A CHAT system essentially lets one user send messages to and receive messages from other users. The messages are usually text based but the chat system should be able to support different media types like Videos, Audio, images etc.

In the following article I will concentrate on data storage aspect of the CHAT system. I will explain the various considerations behind choosing the current date store.  

## Choosing A Data Store
This is what we wanted from the storage aspects of the system :

1. To move away from RDBMS.
	
2. Move towards a NoSQL solution with - 

	1. Flexible schema.
	
	2. Horizontally scalable to meet increased load.
	
	3. High performance.
	
3. Easy maintenance.

4. Resilient and quick recovery.

4. Monitoring.



RDBMS with their normalized storage structure, licensing cost and transactional, locking feature did not seem to fit into our idea of a solution with can support large DB ops/sec with minimum latency.

Flexible schema helps with evolving requirements and new features. Data store maintenence like backup, restore , scaling up etc. should not be a ton of work.


## What We Choose:

<figure class="half center">
<img src="/images/mongo/mongo-db-logo.png" height="400px"></img>
</figure>
  
<b>MongoDB Atlas</b> - A software-as-a-solution version of the well known popular document based NoSQL database.


## Why We Choose MongoDB Atlas:

1. Hosted service. No upfront investment in hardwares or need to have a NoSQL DB Administrator to setup the cluster.

2. Document based storage with support for social media features like tags, likes, comments etc.

3. Ability to easily scale up or across by either increasing the machine configuration or by adding additional shards.

4. Data locality supports fast data retrival and storage.

5. Automated backup, data snapshotting and one click restore.

6. High availability with automated failover and recovery.

7. Automated minor version upgrade of the MonogDB cluster.

8. One click upgrade for major versions.

9. Dashboards visually representing various system and database level metrics. These metrics are archived to provide a historical overview.
 
10. Alerting based on system and database level metrics.

11. Alerting integration with other third part tools like SLACK and Pagerduty.

12. Access to MongoDB enterprise support.

13. Cluster security using TLS/SSL Encryption, authentication, authorization, IP whitelisting etc.

14. Pay as you go model.


 









 


