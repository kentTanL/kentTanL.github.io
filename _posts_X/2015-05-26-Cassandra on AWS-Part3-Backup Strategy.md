---
layout: post
title: Cassandra On AWS - Part 3 - Backup Strategy
description: ""
tags: [Cassandra, AWS]
categories: [Cassandra, AWS]
image:
  background: triangular.png
---

This third part will focus on the databack up stragey for Cassandra on AWS.

##Cassandra Data Backup Strategy

In case of a node failure in a cluster, Cassandra can automatically repair and get the node up to speed on the data using the replicated
data that is present on the other nodes.

It is advised to enabled incremental backup and then collect snapshots at a regular intervals( say 24 hrs).

The snapshot backups and incremental backups can be collected from the nodes and then pushed into S3 for use during restore.

- Snapshot backup should be taken on a daily basis at 12:00 A.M midnight.
- Snapshots & incremental backup files older than 7 days can be deleted from S3.
- S3 backup location should be named and organized based on Cluster and node names. In case of AWS these will be Region and AZs.


##Cassandra Node Failure Recovery Strategy

If a node in a cluster fails or if the node restarts and is out of
the cluster for a certain time then it is mandatory to run the node
repair tool on the node.

Execute the following command on the failed node soon after startup:

{% highlight yaml%}
nodetool repair -dc <datacenter or AZ name> -h localhost

Example:
nodetool repair -dc us-west-2 -h localhost

{% endhighlight %}

Node repair makes sure that that old delete data(Tombstoned) does not resurrect as new data on this node.

##Reference

<a href="https://docs.datastax.com/en/cassandra/2.0/cassandra/operations/ops_backup_takes_snapshot_t.html">[Taking a Snapshot]</a>

<a href="https://docs.datastax.com/en/cassandra/2.0/cassandra/operations/ops_backup_incremental_t.html">[Incremental Backup]</a>

<a href="https://docs.datastax.com/en/cassandra/2.0/cassandra/operations/ops_backup_snapshot_restore_t.html">[Restoring from a snapshot]</a>