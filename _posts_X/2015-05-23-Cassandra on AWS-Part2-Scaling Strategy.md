---
layout: post
title: Cassandra On AWS - Part 2 - Scaling
description: ""
tags: [Cassandra, AWS]
categories: [Cassandra, AWS]
image:
  background: triangular.png
---

This is the second part of the Cassandra series and I will try to talk about Scaling Strategy.

#Cassandra Scaling Strategy

Cassandra scaling will be based on two factors :

- Latency of response for a request.
- Amount of disk space left on the Cassandra node.

The cluster will always scale up and never down. Scaling up will
improve the latency of a request and also provide new disk space to
provide new incoming data.

Scaling will be done as at a cluster level as follows :

##Strategy 1

- Select the cluster to scale.
- Select AZ's with low number of nodes.
- Choose two AZ's and add one node to each of the AZ's.

##Strategy 2

- A cluster will always have a minimum of 3 nodes. Hence scaling in a cluster will be achieve in increments of 3 nodes.
- If a region has 3 AZ's then add one node to each AZ.
- If a region has less than 3 AZ's then one of the AZ will end up having more nodes than the other AZ.

##Scaling Trigger Points or Threshold

- Latency > 500ms for more than 1 minute.
- Scale when disk space remaining is less than 30% of the total available diskspace on the node.

 

Caching Configuration

 
Cassandra Unit Testing

Cassandra Unit provides a library which sprins off an embedded Cassandra server for testing against.

This is very useful for performing Unit/Integration testing of code related to Cassandra.

Steps for Spring JUnit :

    Include the following dependencies in POM

< dependency>

     < groupId>org.cassandraunit< /groupId>
     < artifactId>cassandra-unit-spring< /artifactId>
     < version>2.1.3.1</version>
     < scope>test< /scope>

< /dependency>

< dependency>

     < groupId>org.cassandraunit< /groupId>
     < artifactId>cassandra-unit< /artifactId>
     < version>2.0.2.1< /version>

< /dependency>

2 . Annotate your Junit class with the following

@SpringApplicationConfiguration(classes = CassandraConfiguration.class)

@TestExecutionListeners({ CassandraUnitDependencyInjectionTestExecutionListener.class,
DependencyInjectionTestExecutionListener.class })

@CassandraDataSet(value = { "create-table.cql" }, keyspace = "ioe")

@EmbeddedCassandra

@Configuration

@RunWith(SpringJUnit4ClassRunner.class)

3 . Place your table creating queries in a file named 'create-table.cql' under /resource folder of your TEST .

Cassandara Performance Testing[ Work in Progress]

Leverage the Jmeter plugin for Cassandra developed by Netflix.

[CassJMeter]
 