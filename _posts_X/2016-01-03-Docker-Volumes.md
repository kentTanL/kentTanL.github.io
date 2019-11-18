---
layout: post
title: Docker - Volumes (Mac OSX)
description: ""
tags: [Java]
categories: [Java]
image:
  background: triangular.png
---

<figure class="half center">
<img src="/images/docker/docker.png" height="400px"></img>
</figure>


#How Docker Works ON OSX

On a Linux installation Docker runs directly on the host but on MAC OSX it runs in a slightly different manner. Instead of running directly on the host, docker runs on a small light weight scaled down Linux virtual machine named 'default'.  The docker daemon is running on the linux VM and you use the docker client to connect to it.


<figure class="half center">
<img src="/images/docker/mac_docker_host.svg" height="400px"></img>
</figure>

If you execute the following command in the shell you will see that the IP address returned is different from the HOST machine.

{% highlight yaml %}

$docker-machine ip default
190.121.110.2

{% endhighlight %}


#Overview of Volumes in OSX


1. Virtual Box APP is installed on OSX.
2. Boot2Docker is launched in a virtual machine named 'default' on VirtualBox.
3. "docker run" command run the docker image in a container on the "default" VM.

Volume Mounting :

A directory present on the OSX (host machine) is first mounted to a directory of the same name(need not be of the same name) on the VM.
This ensures access of the host(OSX) directory contents to the guest OS (boot2docker 'default' VM). 
Now when the "docker run" command is executed with the volume parameters, it is the "directory" on the guest OS (boot2docker 'default' VM ) which gets mounted onto the container.  
   

<figure class="center">
<img src="/images/docker/docker-shared.png" height="400px"></img>
</figure>


Example with different directory names:



<figure class="center">
<img src="/images/docker/docker-shared-2.png" height="400px"></img>
</figure>



#Volumes in OSX

Containers are nothing but images with a read-write layer on top of it. This read-write layer enables the process running in the container to read and write files during its run. Once the container exist, the data written to the file system is removed.

Volumes help bring in data persistence to containers. Data stored in volumes are persisted even after the container has exited. Volumes are nothing but a directory present on the HOST machine which is mounted to the container. Any information that the container writes to this mounted directory or in this case called VOLUME, persists independent of the container lifecycle.


Below is an example of how to mount volume while starting the container.

The -v command param is asking docker to make the 'docker-shared' directory present in the current location and expose it as 'mounted-volume' at root location of container VM.

{% highlight yaml %}

docker run -v ./docker-volume:/mounted-volume --name=docker-volume-demo my-image

{% endhighlight %}


This would be a straightforward task in Linux installations but in OSX, since Docker is running on a Linux VM, we would have to create a mount pipe between the HOST and container through the Linux VM.


#Steps(With VitualBox as provider) :

1 . Open up the Virtualbox UI and find the 'default' machine.

2 . Click on 'Shared folders'.

3 . Add a new shared folder. Set "Folder Path" to a directory on OSX that you would like to use as Volume. Provide a name for the folder.
    In this example, I have provided "docker-shared" as the name for the folder.

4 . Enable 'Auto Mount' and 'Make Permanent'.

<figure class="center">
<img src="/images/docker/vbox-shared.png" height="400px"></img>
</figure>


5 . Log into the running 'default' machine using the following command. This should log you into the 'default' Linux VM's shell.

{% highlight yaml %}
$docker-machine ssh default

Note: Make sure that the boot2docker[default virtual machine (vm)] is already running.

Use the following command or start the "default" VM via the VirtualBox application.

$ VBoxManage start default

Ensure the terminal connects to the docker boot2dock virtual machine

$ eval "$(docker-machine env default)"


{% endhighlight %}

6 . Execute the following command and note down the GID and UID values.

{% highlight yaml %}
docker@default:~$ id
uid=1000(docker) gid=50(staff) groups=50(staff),100(docker)
{% endhighlight %}

7 . Create a directory named 'docker-shared' on the Linux VM

{% highlight yaml %}
docker@default:~$ mkdir docker-shared
{% endhighlight %}

8 . Execute the following command to mount the host directory previously shared to the VM

Syntax : sudo mount -t vboxsf -o defaults,uid=<uid>,gid=<gid> <mount_name> <local_dir>
 
<local_dir> Is the name of the directory created on the "default" VM.
<mount_name> Is the folder name provided in the shared folder settings of the VM.
 
{% highlight yaml %}
docker@default:~$ sudo mount -t vboxsf -o defaults,uid=1000,gid=50 docker-shared docker-shared
{% endhighlight %}

9 . Exit out of the Linux VM default machine.

{% highlight yaml %}
docker@default:~$ exit
{% endhighlight %}

10 . Run the container with the following command.

{% highlight yaml %}

$docker run -v ./docker-shared:/docker-shared --name=docker-volume-demo my-image

{% endhighlight %}


11 . View the mounted-volume.


{% highlight yaml %}
root@3af1249144e1:/# ls
bin  boot  dev  etc  home  lib  lib64  media  mnt  mounted-volume  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
root@3af1249144e1:/# cd docke-shared
root@3af1249144e1:/docker-shared# ls
root@3af1249144e1:/docker-shared# mkdir testdir
root@3af1249144e1:/docker-shared# ls
testdir
{% endhighlight %}












