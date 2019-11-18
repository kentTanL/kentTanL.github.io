---
layout: post
title: Cassandra Integration Testing
description: ""
tags: [Cassandra, Unit Testing]
categories: [Cassandra]
image:
  background: triangular.png
---

<figure class="half center">

<img src="/images/cassandra-aws/cassandra_logo.png" height="400px"></img>

</figure>


Cassandra Unit is a library which can intergrate with your Java code and help dynamically spin up an embedded Cassandra instances when needed.
Once you are done using this instance, you can just termiated it and the library will take care of cleaning up after it.

What purpose does this library serve ?

1. Biggest advantage is in `Unit` and `Integration` testing. Developers or CI/CD tools no longer need to depend on a centralized database.
2. Developers can run all kinds of test scenarios on their code and also test you their DML/DDL queryies before executing them on main Cassandra Cluster. Which means faster development, reduced latency and dependency.

##Steps for Spring JUnit :

### 1. Include the following dependencies in pom.xml

{% highlight yaml %}

<!--Cassandra Unit Integration With Spring -->
<dependency>
     <groupId>org.cassandraunit</groupId>
     <artifactId>cassandra-unit-spring</artifactId>
     <version>2.1.3.1</version>
     <scope>test</scope>
</dependency>

<!--Cassandra Unit Library-->
<dependency>
     <groupId>org.cassandraunit</groupId>
     <artifactId>cassandra-unit</artifactId>
     <version>2.0.2.1</version>
</dependency>

{% endhighlight %}

### 2. Annotate your Junit class with the following

{% highlight yaml %}

@SpringApplicationConfiguration(classes = CassandraConfiguration.class)

@TestExecutionListeners({ CassandraUnitDependencyInjectionTestExecutionListener.class,
DependencyInjectionTestExecutionListener.class })

@CassandraDataSet(value = { "create-table.cql" }, keyspace = "ioe")

@EmbeddedCassandra

@Configuration

@RunWith(SpringJUnit4ClassRunner.class)

{% endhighlight %}


### 3. Place your DDL queries in a file named 'create-table.cql'under /resource folder of your TEST

Leverage the <a href="https://github.com/Netflix/CassJMeter" >Jmeter plugin for Cassandra </a> developed by Netflix.

