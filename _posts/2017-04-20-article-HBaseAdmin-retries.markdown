---
layout:     post
title:      "[HBase源码] HBaseAdmin的重试机制"
subtitle:   "[HBase源码] HBaseAdmin的重试机制"
date:       2017-04-20 21:20:00
author:     "Kent"
header-img: "img/post-bg-2016.jpg"
catalog: true
tags:
    - HBase
---


#### **1. 前言**

HBase 版本：V 1.0.0

在项目中获取HTable的详细信息时，Http connection一直处于pending状态，这是由于我的机器无法Ping通新加入的集群，但是这个connection 的pending时长已经超过了1小时之久，并且在``org.apache.hadoop.hbase.client.ConnectionManager.checkIfBaseNodeAvailable(..)``方法中会写入一大堆的错误日志，源码如下：
```java
package org.apache.hadoop.hbase.client;

class ConnectionManager {
    // More ...
    
    private void checkIfBaseNodeAvailable(ZooKeeperWatcher zkw)
          throws MasterNotRunningException {
          String errorMsg;
          try {
            if (ZKUtil.checkExists(zkw, zkw.baseZNode) == -1) {
              errorMsg = "The node " + zkw.baseZNode+" is not in ZooKeeper. "
                + "It should have been written by the master. "
                + "Check the value configured in 'zookeeper.znode.parent'. "
                + "There could be a mismatch with the one configured in the master.";
              LOG.error(errorMsg);
              throw new MasterNotRunningException(errorMsg);
            }
          } catch (KeeperException e) {
            errorMsg = "Can't get connection to ZooKeeper: " + e.getMessage();
            LOG.error(errorMsg);
            throw new MasterNotRunningException(errorMsg, e);
          }
    }
    
    // More ...
}
```

按理，如果无法连接，那么应该及时返回才对，但是HBase却不断的重试，如果不进行超时之类的配置，那么等待的时间超过数小时也是正常的。

那么HBase再获取HTable描述时的重试机制究竟是怎样实现的？看看源码玩吧（其实``HBaseAdmin``中的其它操作也时类似的）。

#### **2. 源码**

既然是获取表描述时引出的问题，那么就从``HBaseAdmin.getTableDescriptor(...)``方法看起吧，源码如下：


```java
package org.apache.hadoop.hbase.client;

public class HBaseAdmin implements Admin {
    // More ...
    
     public HTableDescriptor getTableDescriptor(final TableName tableName)
  throws TableNotFoundException, IOException {
    if (tableName == null) return null;
    if (tableName.equals(TableName.META_TABLE_NAME)) {
      return HTableDescriptor.META_TABLEDESC;
    }
    // 主要是这个方法
    HTableDescriptor htd = executeCallable(new MasterCallable<HTableDescriptor>(getConnection()) {
      @Override
      public HTableDescriptor call(int callTimeout) throws ServiceException {
        GetTableDescriptorsResponse htds;
        GetTableDescriptorsRequest req =
            RequestConverter.buildGetTableDescriptorsRequest(tableName);
        htds = master.getTableDescriptors(null, req);

        if (!htds.getTableSchemaList().isEmpty()) {
          return HTableDescriptor.convert(htds.getTableSchemaList().get(0));
        }
        return null;
      }
    });
    if (htd != null) {
      return htd;
    }
    throw new TableNotFoundException(tableName.getNameAsString());
  }
  
  // More ...
}
```

``executeCallable`` 方法源码：

```java
package org.apache.hadoop.hbase.client;

public class HBaseAdmin implements Admin {
    // More ...
    
    private <V> V executeCallable(MasterCallable<V> callable) throws IOException {
        RpcRetryingCaller<V> caller = rpcCallerFactory.newCaller();
        try {
          return caller.callWithRetries(callable, operationTimeout);
        } finally {
          callable.close();
        }
    }
  
    // More ...
}
```

这里通过``RpcRetryingCaller``对象的``callWithRetries(...)``方法调用``callable.call(...)``方法，其源码如下：


```java
package org.apache.hadoop.hbase.client;

public class RpcRetryingCaller<T> {
    // More ...
    
    public T callWithRetries(RetryingCallable<T> callable, int callTimeout)
  throws IOException, RuntimeException {
    List<RetriesExhaustedException.ThrowableWithExtraContext> exceptions =
      new ArrayList<RetriesExhaustedException.ThrowableWithExtraContext>();
    this.globalStartTime = EnvironmentEdgeManager.currentTime();
    context.clear();
    // Code Mark:A
    for (int tries = 0;; tries++) {
      long expectedSleep;
      try {
        callable.prepare(tries != 0); // if called with false, check table status on ZK
        interceptor.intercept(context.prepare(callable, tries));
        // Code Mark:B
        return callable.call(getRemainingTime(callTimeout));
      } catch (PreemptiveFastFailException e) {
        throw e;
      } catch (Throwable t) {
        ExceptionUtil.rethrowIfInterrupt(t);
        if (tries > startLogErrorsCnt) {
          LOG.info("Call exception, tries=" + tries + ", retries=" + retries + ", started=" +
              (EnvironmentEdgeManager.currentTime() - this.globalStartTime) + " ms ago, "
              + "cancelled=" + cancelled.get() + ", msg="
              + callable.getExceptionMessageAdditionalDetail());
        }

        // translateException throws exception when should not retry: i.e. when request is bad.
        interceptor.handleFailure(context, t);
        t = translateException(t);
        callable.throwable(t, retries != 1);
        RetriesExhaustedException.ThrowableWithExtraContext qt =
            new RetriesExhaustedException.ThrowableWithExtraContext(t,
                EnvironmentEdgeManager.currentTime(), toString());
        exceptions.add(qt);
        // Code Mark:C
        if (tries >= retries - 1) {
          throw new RetriesExhaustedException(tries, exceptions);
        }
        // If the server is dead, we need to wait a little before retrying, to give
        //  a chance to the regions to be
        // tries hasn't been bumped up yet so we use "tries + 1" to get right pause time
        expectedSleep = callable.sleep(pause, tries + 1);

        // If, after the planned sleep, there won't be enough time left, we stop now.
        long duration = singleCallDuration(expectedSleep);
        // Code Mark:D
        if (duration > callTimeout) {
          String msg = "callTimeout=" + callTimeout + ", callDuration=" + duration +
              ": " + callable.getExceptionMessageAdditionalDetail();
          throw (SocketTimeoutException)(new SocketTimeoutException(msg).initCause(t));
        }
      } finally {
        interceptor.updateFailureInfo(context);
      }
      try {
        if (expectedSleep > 0) {
          synchronized (cancelled) {
            if (cancelled.get()) return null;
            cancelled.wait(expectedSleep);
          }
        }
        if (cancelled.get()) return null;
      } catch (InterruptedException e) {
        throw new InterruptedIOException("Interrupted after " + tries + " tries  on " + retries);
      }
    }
    
    // More ...
}

```

上面的代码略多，但是最主要的就是标记为``Code Mark: A : *``的几处代码了：

- Code Mark A : 循环
- Code Mark B : 调用具体的实现MasterCallable接口的方法
- Code Mark C : 如果重试次数超过允许的重试次数，则抛出RetriesExhaustedException
- Code Mark D : 如果持续的时间大于允许的超时时间，则抛出SocketTimeoutException

所以，这里有两个很重要的参数能够影响重试的次数或时间，第一个参数是``retries``，它决定了最多的重试次数，第二个参数时``callTimeout``，它决定了超时时间，接下来再看看它们是怎样赋值的。

##### **重试次数**

在上面的``executeCallable``方法内，有一行代码：
```java
RpcRetryingCaller<V> caller = rpcCallerFactory.newCaller();
```

它实例化了一个``RpcRetryingCaller``，``newCaller()``的源码如下：

```java
package org.apache.hadoop.hbase.client;

public class RpcRetryingCallerFactory {
    // More ...
    
    public <T> RpcRetryingCaller<T> newCaller() {
        // We store the values in the factory instance. This way, constructing new objects
        //  is cheap as it does not require parsing a complex structure.
        RpcRetryingCaller<T> caller = new RpcRetryingCaller<T>(pause, retries, interceptor,
            startLogErrorsCnt);
    
        // wrap it with stats, if we are tracking them
        if (enableBackPressure && this.stats != null) {
          caller = new StatsTrackingRpcRetryingCaller<T>(pause, retries, interceptor,
              startLogErrorsCnt, stats);
        }
    
        return caller;
    }
    
    // More ...
}
```

可以看见，``retries``是``RpcRetryingCallerFactory``的一个实例变量，它的值在构造方法中便设置了：

```java
package org.apache.hadoop.hbase.client;

public class RpcRetryingCallerFactory {
    // More ...
    
    public RpcRetryingCallerFactory(Configuration conf, RetryingCallerInterceptor interceptor) {
        this.conf = conf;
        pause = conf.getLong(HConstants.HBASE_CLIENT_PAUSE,
            HConstants.DEFAULT_HBASE_CLIENT_PAUSE);
        // 这里尝试从配置文件中获取retries的值
        retries = conf.getInt(HConstants.HBASE_CLIENT_RETRIES_NUMBER,
            HConstants.DEFAULT_HBASE_CLIENT_RETRIES_NUMBER);
        startLogErrorsCnt = conf.getInt(AsyncProcess.START_LOG_ERRORS_AFTER_COUNT_KEY,
            AsyncProcess.DEFAULT_START_LOG_ERRORS_AFTER_COUNT);
        this.interceptor = interceptor;
        enableBackPressure = conf.getBoolean(HConstants.ENABLE_CLIENT_BACKPRESSURE,
            HConstants.DEFAULT_ENABLE_CLIENT_BACKPRESSURE);
    }
    
    // More ...
}
```
如果配置文件中没有配置``hbase.client.retries.number``的值，那么``retries=31``，两个常量的声明如下：

```java
package org.apache.hadoop.hbase;

public final class HConstants {
    // More ...
    public static final String HBASE_CLIENT_RETRIES_NUMBER = "hbase.client.retries.number";
    
    public static final int DEFAULT_HBASE_CLIENT_RETRIES_NUMBER = 31;
    // More ...
}
```

到此，如果我们配置了``hbase.client.retries.number``的值，即可控制HBase获取表描述的重试次数。

##### **超时时间**

另外一个参数是``callTimeout``，即控制超时时间的变量，它是通过上面的``HBaseAdmin.executeCallable(...)``方法传入的，即``operationTimeout``，``operationTimeout``是``HBaseAdmin``类的一个实例变量，在``HBaseAdmin``中，只有``getOperationTimeout()``方法，所以没有办法可以设置``operationTimeout``的值，但是在``HBaseAdmin``的一个构造函数中（另外几个构造函数已经被标记为@Deprecated了），可以看见它，源码如下：

```java
package org.apache.hadoop.hbase.client;

public class HBaseAdmin implements Admin {
    // More ...
    
    HBaseAdmin(ClusterConnection connection) {
        this.conf = connection.getConfiguration();
        this.connection = connection;
    
        this.pause = this.conf.getLong(HConstants.HBASE_CLIENT_PAUSE,
            HConstants.DEFAULT_HBASE_CLIENT_PAUSE);
        this.numRetries = this.conf.getInt(HConstants.HBASE_CLIENT_RETRIES_NUMBER,
            HConstants.DEFAULT_HBASE_CLIENT_RETRIES_NUMBER);
        this.retryLongerMultiplier = this.conf.getInt(
            "hbase.client.retries.longer.multiplier", 10);
        
        // MARK
        this.operationTimeout = this.conf.getInt(HConstants.HBASE_CLIENT_OPERATION_TIMEOUT,
            HConstants.DEFAULT_HBASE_CLIENT_OPERATION_TIMEOUT);
    
        this.rpcCallerFactory = RpcRetryingCallerFactory.instantiate(this.conf);
    }
    
    // More ...
}
```

如果配置文件中没有``hbase.client.operation.timeout``的值，那么``operationTimeout=Integer.MAX_VALUE=2147483647``，是的，596.5小时左右，相关的两个常量声明如下：

```java
package org.apache.hadoop.hbase;

public final class HConstants {
    // More ...
    /** Parameter name for HBase client operation timeout, which overrides RPC timeout */
    public static final String HBASE_CLIENT_OPERATION_TIMEOUT = "hbase.client.operation.timeout";
    
    /** Default HBase client operation timeout, which is tantamount to a blocking call */
    public static final int DEFAULT_HBASE_CLIENT_OPERATION_TIMEOUT = Integer.MAX_VALUE;
}
```

到此，如果我们配置了``hbase.client.operation.timeout``的值，即可控制HBase的重试超时时间。

同时对于上上一段源码，可以看见有一行代码：

```java
this.numRetries = this.conf.getInt(HConstants.HBASE_CLIENT_RETRIES_NUMBER,
            HConstants.DEFAULT_HBASE_CLIENT_RETRIES_NUMBER);
```

``numRetries``的值与``RpcRetryingCallerFactory#retries``的值的获取方式是一样的，它在执行``HBaseAdmin.createTable(...)``等方法时会用到。


#### **3. 最后**

要完成上面两个值的配置的方式有两种，一种是在配置文件中，一种是在代码中，分别如下：

在/src/main/resources下的hbase-site.xml中添加：

```java
<property>
	<name>hbase.client.retries.number</name>
	<value>10</value>
</property>
<property>
	<name>hbase.client.operation.timeout</name>
	<value>60000</value>
</property>
```

或者在代码中获取``org.apache.hadoop.hbase.client.Connection``时设置配置信息：

```java

public org.apache.hadoop.hbase.client.Connection getConnection(...) throws IOException {
    Configuration config = HBaseConfiguration.create();
    
    config.set("hbase.client.retries.number", "10");
    config.set("hbase.client.operation.timeout", "60000");
    
    // More ...
    
    org.apache.hadoop.hbase.client.Connection conn =  ConnectionFactory.createConnection(config);
    
    return conn;
}

```

完成，这下便不会很久都获取不到结果了。

当然了，这里还可配置每次重试的间隔/暂停时间：
```java
package org.apache.hadoop.hbase;

public final class HConstants {
    // More ...
    /**
      * Parameter name for client pause value, used mostly as value to wait
      * before running a retry of a failed get, region lookup, etc.
      */
    public static final String HBASE_CLIENT_PAUSE = "hbase.client.pause";
      
    /**
      * Default value of {@link #HBASE_CLIENT_PAUSE}.
      */
    public static final long DEFAULT_HBASE_CLIENT_PAUSE = 100;

    // More ...
}
```

以上。