---
layout:     post
title:      "自定义ClassLoader实现隔离运行多版本jar包的方式"
subtitle:   "自定义ClassLoader实现隔离运行多版本jar包的方式"
date:       2017-06-14 21:20:00
author:     "Kent"
header-img: "img/post-bg-2016.jpg"
catalog: true
tags:
    - classLoader
---


## 1. 应用场景

有时候我们需要在一个 Project 中运行多个不同版本的 jar 包，以应对不同集群的版本或其它的问题。如果这个时候选择在同一个项目中实现这样的功能，那么通常只能选择更低版本的 jar 包，因为它们通常是向下兼容的，但是这样也往往会失去新版本的一些特性或功能，所以我们需要以扩展的方式引入这些 jar 包，并通过隔离执行，来实现版本的强制对应。


## 2. 实现

在 Java 中，所有的类默认通过 ClassLoader 加载，而 Java 默认提供了三层的 ClassLoader，并通过双亲委托模型的原则进行加载，其基本模型与加载位置如下（更多ClassLoader相关原理请自行搜索）：

![Class Loader](/img/2017-06-14-classLoader-jar/classLoader.png)

Java 中默认的 ClassLoader 都规定了其指定的加载目录，一般也不会通过 JVM 参数来使其加载自定义的目录，所以我们**需要自定义一个 ClassLoader 来加载装有不同版本的 jar 包的扩展目录**，同时为了使运行扩展的 jar 包时，与启动项目实现绝对的隔离，我们**需要保证他们所加载的类不会有相同的 ClassLoader**，根据双亲委托模型的原理可知，我们必须使自定义的 ClassLoader 的 parent 为 null，这样不管是 JRE 自带的 jar 包或一些基础的 Class 都不会委托给 App ClassLoader（当然仅仅是将 Parent 设置为 null 是不够的，后面会说明）。与此同时这些实现了不同版本的 jar 包，是经过二次开发后的可以独立运行的项目。

### 2.1 实例

现在假定有这样一个需求，实现针对集群（比如 Hadoop 集群）版本为 V1 与 V2 的对应的执行程序，那么假定有如下项目：

```
Executor-Parent: 提供基础的 Maven 引用，可利用 Maven 一键打包所有的子模块/项目
Executor-Common: 提供基础的接口，已经有公有的实现等
Executor-Proxy: 执行不同版本程序的代理程序
Executor-V1: 版本为V1的执行程序
Executor-V2: 版本为V2的执行程序
```

这里为了更凸显 ClassLoader 的实现，不做 `Executor-Parent` 的实现，同时为了简便，也没有设置包名。

#### 1) Executor-Common

在 `Executor-Common` 中提供一个接口，声明执行的具体方法：

```java
public interface Executor {
    void execute(final String name);
}
```

这里的方法使用了基础类型 `String`，实际中可能会使用自定义的类型，那么在 Porxy 的实现中则需要使用自定义的 ClassLoader 来加载参数，并使用反射来获取方法（后面会有一个简单的示例）。回到之前的示例，这里同时提供一个抽象的实现类：

```java
public class AbstractExecutor implements Executor {

    @Override
    public void execute(final String name) {
        this.handle(new Handler() {
            @Override
            public void handle() {
                System.out.println("V:" + name);
            }
        });
    }
    
    protected void handle(Handler handler) {
        handler.call();
    }
    
    protected abstract class Handler {
        public void call() {
            ClassLoader oldClassLoader = Thread.currentThread().getContextClassLoader();
            // 临时更改 ClassLoader
            Thread.currentThread().setContextClassLoader(AbstractExecutor.class.getClassLoader());
            
            handle();
            
            // 还原为之前的 ClassLoader
            Thread.currentThread().setContextClassLoader(oldClassLoader);
        }
        
        public abstract void handle();
    }
}
```


这里需要临时更改当前线程的 ContextClassLoader, 以应对扩展程序中可能出现的如下代码：

```java
ClassLoader classLoader = Thread.currentThread().getContextClassLoader();

classLoader.loadClass(...);
```

因为它们会获取当前线程的 `ClassLoader` 来加载 class，而当前线程的`ClassLoader`极可能是`App ClassLoader`而非自定义的`ClassLoader`, 也许是为了安全起见，但是这会导致它可能加载到启动项目中的`class`（如果有），或者发生其它的异常，所以我们在执行时需要临时的将当前线程的`ClassLoader`设置为自定义的`ClassLoader`，以实现绝对的隔离执行。


#### 2) Executor-V1 & Executor-V2

`Executor-V1` 和 `Executor-V2` 依赖了 `Executor-Common.jar`，并实现了 `Executor` 接口的方法：

```java
public class ExecutorV1 extends AbstractExecutor {

    @Override
    public void execute(final String name) {
        this.handle(new Handler() {
            @Override
            public void handle() {
                System.out.println("V1:" + name);
            }
        });
    }

}
```

```java
public class ExecutorV2 extends AbstractExecutor {

    @Override
    public void execute(final String name) {
        this.handle(new Handler() {
            @Override
            public void handle() {
                System.out.println("V2:" + name);
            }
        });
    }

}
```

这里仅仅是打印了它们的版本信息，实际中，它们可能需要引入不同的版本的 Jar 包，然后根据这些 Jar 包完成相应的操作。

#### 3) Executor-Proxy

`Executor-Proxy` 利用自定义的 `ClassLoader` 和反射来实现加载与运行 `ExecutorV1` 和 `ExecutorV2` 中 `Executor` 接口的实现，而 `ExecutorV1` 和 `ExecutorV2` 将以 jar 包的形式被分别放置在 `${Executor-Proxy_HOME}\ext\v1` 和 `${Executor-Proxy_HOME}\ext\v2` 目录下，其中自定义的 `ClassLoader` 实现如下：

```java
public class StandardExecutorClassLoader extends URLClassLoader {
    private final static String baseDir = System.getProperty("user.dir") + File.separator + "ext" + File.separator;
    
    public StandardExecutorClassLoader(String version) {
    	super(new URL[] {}, null); // 将 Parent 设置为 null
    	
    	loadResource(version);
    }
    
    @Override
    public Class<?> loadClass(String name) throws ClassNotFoundException {
    	// 测试时可打印看一下
    	System.out.println("Class loader: " + name);
    
    	return super.loadClass(name);
    }
    
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        try {
            return super.findClass(name);
        } catch(ClassNotFoundException e) {
            return StandardExecutorClassLoader.class.getClassLoader().loadClass(name);
        }
    }
    
    private void loadResource(String version) {
    	String jarPath = baseDir + version;
    	
    	// 加载对应版本目录下的 Jar 包
    	tryLoadJarInDir(jarPath);
    	// 加载对应版本目录下的 lib 目录下的 Jar 包
    	tryLoadJarInDir(jarPath + File.separator + "lib");
    }
    
    private void tryLoadJarInDir(String dirPath) {
    	File dir = new File(dirPath);
    	// 自动加载目录下的jar包
    	if (dir.exists() && dir.isDirectory()) {
    		for (File file : dir.listFiles()) {
    			if (file.isFile() && file.getName().endsWith(".jar")) {
    				this.addURL(file);
    				continue;
    			}
    		}
    	}
    }
    
    private void addURL(File file) {
    	try {
    		super.addURL(new URL("file", null, file.getCanonicalPath()));
    	} catch (MalformedURLException e) {
    		e.printStackTrace();
    	} catch (IOException e) {
    		e.printStackTrace();
    	}
    }

}
```

`StandardExecutorClassLoader` 在实例化时，会自动加载扩展目录下与其lib目录下的 jar 包，这里之所以要加载 lib 目录下的 jar，是为了加载扩展的依赖包。

有了`StandardExecutorClassLoader`，我们还需要一个调用各版本程序的代理类`ExecutorPorxy`，其实现如下：

```java
import java.lang.reflect.Method;

public class ExecutorProxy implements Executor {
    private String version;
    private StandardExecutorClassLoader classLoader;
    
    public ExecutorProxy(String version) {
        this.version = version;
        classLoader = new StandardExecutorClassLoader(version);
    }
    
    @Override
    public void execute(String name) {
        try {
            // Load ExecutorProxy class
            Class<?> executorClazz = classLoader.loadClass("Executor" + version.toUpperCase());
            
            Object executorInstance = executorClazz.newInstance();
            Method method = executorClazz.getMethod("execute", String.class);
            
            method.invoke(executorInstance, name);
        } catch (Exception e) {
            e.printStackTrace();
        }
        
    }
}
```

这里是一个比较简单的实现，因为通过反射调用的方法的参数是基本类型，在实际中，更多的可能是自定义的参数，那么这时候则需要先通过自定义的 `ClassLoader` 加载其 `Class`，然后才能去获取对应的方法，下面是一个省去上下文的一个例子（不能直接运行）：

```java
public void call() throws IOException {
    try {
        // Load HBaseApi class
        Class<?> hbaseApiClazz = loadHBaseApiClass();
        Object hbaseApiInstance = hbaseApiClazz.newInstance();

        // Load parameter class
        Class<?> paramClazz = classLoader.loadClass(VO_PACKAGE_PATH + "." + sourceParame.getClass().getSimpleName());
        
        // Transition parameter to targeParameter from sourceParameter 
        Object targetParam = BeanUtils.transfrom(paramClazz, sourceParame);
        
        // Get function
        Method method = hbaseApiClazz.getMethod(methodName, paramClazz);
        // Invoke function by targetParam
        method.invoke(hbaseApiInstance, targetParam);
        
    } catch(ClassNotFoundException | NoSuchMethodException | SecurityException | InstantiationException | IllegalAccessException e) {
        e.printStackTrace();
    } catch (IllegalArgumentException e) {
        // TODO Auto-generated catch block
        e.printStackTrace();
    } catch (InvocationTargetException e) {
        // TODO Auto-generated catch block
        e.printStackTrace();
    }
}
```

## 3. 运行

将`ExecutorV1` 和 `ExecutorV2`分别打包，并将其打包后的 jar包与其依赖（lib目录下）放入 `Executor-Proxy` 项目的 `ext\v1` 和 `ext\v2` 目录下，在 `Executor-Proxy` 项目中则可以使用 Junit 进行测试：

```Java
public class ExecutorTest {
    
    @Test
    public void testExecuteV1() {
        
        Executor executor = new ExecutorProxy("v1");
        
        executor.execute("TOM");
    }
    
    @Test
    public void testExecuteV2() {
        
        Executor executor = new ExecutorProxy("v2");
        
        executor.execute("TOM");
    }
    
}

```

打印结果最终分别如下：

```
execute testExecuteV1():

V1:TOM
```

```
execute testExecuteV2():

V2:TOM
```

## 4. 总结

总的来说，实现隔离允许指定 jar 包，主要需要做到以下几点：

- 自定义 ClassLoader，使其 Parent = null，避免其使用系统自带的 ClassLoader 加载 Class。
- 在调用相应版本的方法前，更改当前线程的 ContextClassLoader，避免扩展包的依赖包通过`Thread.currentThread().getContextClassLoader()`获取到非自定义的 ClassLoader 进行类加载
- 通过反射获取 Method 时，如果参数为自定义的类型，一定要使用自定义的 ClassLoader 加载参数获取 Class，然后在获取 Method，同时参数也必须转化为使用自定义的 ClassLoade 加载的类型（不同 ClassLoader 加载的同一个类不相等）

实际运用中，往往容易做到第一点或第三点，而忽略第二点，比如使用 HBase 相关包时。

当然，这只是一种解决的方式，我们仍然可以使用微服务来达到同样甚至更棒的效果，

以上。
