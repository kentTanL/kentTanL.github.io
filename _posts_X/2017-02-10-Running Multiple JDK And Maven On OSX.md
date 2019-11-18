---
layout: post
title: Running Multiple JDK and Maven On OSX
tags: [Development]
categories: [Development Environment]
image:
  background: triangular.png
---

If you are a Java developer working on MicroServices then its highly probable that you will be dealing with older services which are written on
JDK 1.7 or older with Maven 2.x and newer services written in JDK 1.8 and Maven 3.x.

In order to work on both the newer and older services, you would need to have all of Maven 2.x, Maven 3.x , JDK 1.8, JDK 1.7 installed on 
you OSX. 

Even though you have multiple versions installed, you system environment variables can point to only one of the version. 

However, you can use the following script to easily switch between different version on same of different terminal window.

### Create File 

{% highlight yaml %}
vi ~/.bash_profile
{% endhighlight %}  


### Enter contents

{% highlight yaml %}

export MAVEN_HOME=/Users/esrinivasan/develop/maven

export PATH=$PATH:$MAVEN_HOME/bin

export JAVA_HOME=$(/usr/libexec/java_home -v 1.7)
setjdk() {
  export JAVA_HOME=$(/usr/libexec/java_home -v $1)
}


setmaven(){
    tmp="$PWD"
    cd /Users/esrinivasan/develop
    rm maven
    ln -s /Users/esrinivasan/develop/apache-maven-$1 maven
    cd "$tmp"
}

{% endhighlight %}  


### Usage

{% highlight yaml %}
esrinivasan:localhost$ setjdk 1.7
{% endhighlight %}  


{% highlight yaml %}
esrinivasan:localhost$ setmaven 3.3.3
{% endhighlight %}  


### Output


{% highlight yaml %}
$ mvn -version
Listening for transport dt_socket at address: 8453
Apache Maven 3.3.3 (7994120775791599e205a5524ec3e0dfe41d4a06; 2015-04-22T04:57:37-07:00)
Maven home: /Users/esrinivasan/develop/maven
Java version: 1.7.0_79, vendor: Oracle Corporation
Java home: /Library/Java/JavaVirtualMachines/jdk1.7.0_79.jdk/Contents/Home/jre
Default locale: en_US, platform encoding: UTF-8
OS name: "mac os x", version: "10.10.5", arch: "x86_64", family: "mac"
{% endhighlight %}  


### Note

#### 1. There are two different version of maven installed in two different locations :

	/Users/esrinivasan/develop/apache-maven-2.2.1
	/Users/esrinivasan/develop/apache-maven-3.3.3
	
#### 2. There are two version of JDK installed  (1.7 and 1.8).


#### esrinivasan is my machine name. You can replace it with whatever path you have on your machine.
