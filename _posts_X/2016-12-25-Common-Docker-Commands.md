---
layout: post
title: Common-Docker-Commands
tags: [Docker]
categories: [Docker]
image:
  background: triangular.png
---

Some useful commonly used docker commands when working on OSX with VirtualBox.

1) This ensures the boot2docker VM is up and running.
{% highlight yaml %}
docker-machine start default
{% endhighlight %}

<hr>
<br/>

2) This is the command that needs to be executed once for every new terminal window opened, so as to establish connection with the boot2docker vm. 

{% highlight yaml %}
 eval “$(docker-machine env defaut)”
{% endhighlight %}
	
<hr>
<br/>

3) This command returns the IP address of the 'default' VM running boot2docker

{% highlight yaml %}
docker-machine ip default
{% endhighlight %}

<hr>
<br/>

4) Returns a list of VM with boot2docker install and running on VirtualBox.

{% highlight yaml %}
docker-machine ls

Sample Output:

NAME        ACTIVE   DRIVER       STATE     URL                         SWARM   ERRORS
default     *        virtualbox   Running   tcp://192.168.99.100:2376           

{% endhighlight %}

<hr>
<br/>

5) Regenerates the TLS certificates needed to communicated with the 'boot2docker' VM.  

{% highlight yaml %}
docker-machine regenerate-certs default     /// To regenerate the TLS certs for docker
{% endhighlight %}

<hr>
<br/>

6) Lists all available docker images 

{% highlight yaml %}
 docker images
{% endhighlight %}

<hr>
<br/>

7) Looks the docker process running with a particular container id.

{% highlight yaml %}
docker ps -a | grep <container -id >
{% endhighlight %}

<hr>
<br/>

8) Removes a running docker container 

{% highlight yaml %}
docker rm <container id>
{% endhighlight %}

<hr>
<br/>

9) Removes a particular docker image

{% highlight yaml %}
docker rmi  <imageid>
{% endhighlight %}

<hr>
<br/>

10) Kills a particular docker container

{% highlight yaml %}
docker kill <container-id>
{% endhighlight %}

<hr>
<br/>
 
11) Connects to the docker container and gives access to the bash shell to execute commands on the container 

{% highlight yaml %}
docker exec -i -t <container id> /bin/bash
{% endhighlight %}

<hr>
<br/>

12) Runs a docker image and does not automatically terminate the container. Allows an easy way to keep the container running and later connect to the containers shell using "docker exec -it " command.

{% highlight yaml %}
docker run -d <image-name> tail -f /dev/null
{% endhighlight %}

<hr>
<br/>


13) This is not a docker command. This utility is part of the virtual box installation. Allows one to power-off a VM running on VirtualBox  

{% highlight yaml %}
VBoxManage controlvm <vm-name> poweroff
{% endhighlight %}


<hr>
<br/>

14) Again, not a docker command. A virtual box command to share a directory on the HOST OS (OSX: /Users/esrinivasan/..) with the guest operating system running on VirtualBox. The name of the shared folder is set as "/eks" here. 

{% highlight yaml %}
VBoxManage sharedfolder add default --name /eks --hostpath /Users/esrinivasan/develop/learning/Docker/dec-2016/docker-node/docker-volume
{% endhighlight %}

<hr>
<br/>
