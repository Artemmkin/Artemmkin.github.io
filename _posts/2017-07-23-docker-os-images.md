---
layout: post
title: >
  What are Docker OS images and why would I want to use them in my Dockerfile?  
tags: [docker, VM, virtualization]
---
Docker is not new these days. Everybody knows how to run a docker container at least on a local machine, because it's so easy. You find an image on DockerHub, you run `docker run -d <image-name>` and that's it.

You probably also know how to build docker images, because all you need is create a `Dockerfile`, use 7-8 [commands](https://docs.docker.com/engine/reference/builder/) to describe how to package your application and all its dependencies, finally build the container image with `docker build` command and run it.

Let's look at this simple Dockerfile.
~~~yml
FROM ubuntu:14.04
COPY ./hello-world .
EXPOSE 8080
CMD [ "./hello-world" ]
~~~

It looks simple, right? Nothing special.


You specify the OS image, add you binary file, ... wait, it just struck me. Look again, at the Dockerfile.
 <!--break-->

What in the world the ubuntu OS does here? If a container needs an operating system, then how is it different from a virtual machine?

A virtual machine has an operating system and runs our applications inside. So how different is a docker container then? Is it some type of virtual machine?

**Was all that talk about ```a container represents a bundle of your application and its dependencies``` just a big lie? (☉_☉)**

To really understand the difference between the VMs and containers I offer you to look at a bit of history. I want to specifically look at how these technologies came into play and what problems they were meant to solve.

Let's first start by looking at how things were before people started using VMs. The traditional server model that was used for decades in IT looked like this.

![200x200](/public/img/docker/oldmodel.png)

Here we have some vendor hardware and an operating system which runs on top of it. The operating system is bound to the hardware by the drivers which are specific to that hardware. Finally, on top of the OS we run various types of applications and services.

This model has lots of limitations.

First, it leads to lots of underutilized hardware. In most cases, the applications and services that you run on your  server don't utilize even half of its computing resources. In fact, previous studies showed that the average utilization of the server's resources was about 5-10%. Which means that if you spend 10 thousand dollars on a server hardware, you waste 9 thousand dollars worth of computing power. Plus, you have to spend a lot of money on the servers maintenance.

But why do we don't utilize all of our computing resources? Why can't we just run more applications on our servers? (・ω・)b

Well, it's not that simple as it might seem. There are some challenges that we face at OS level which include running multiple instances of an application or running different versions of the same application at the same time.

Most of the applications you see have some dependencies that need to be provided in order for them to run. And the problem we often see is that two applications may depend on the same library but require different versions of that library. For example, this might be the case when we want to run different versions of the same application.  Installing two versions of the same library system-wide and telling each of our applications which one they need to use would be a mess and in most cases is not even possible.

We can see how this problem is addressed in some programming languages. For instance, in python, you use virtual environments to create a separate isolated python runtime environment with all the dependencies specific to each application. This way we can run different python applications without breaking each applications' dependencies.

Although, the problem with application's dependencies is solved in some programming languages, it's not solved for all  applications that we use. We need to have some technology which would allow us to provide isolation of runtime environments for any application.

Besides, what if we decide to increase our computing resource utilization by running multiple instances of the same application on one server?

We find many problems here as well. For example, running multiple instances of nginx would require changing its init script, as well as its configs because we can't bind more than one application to the same port. As you can see, it takes quite a bit of work.

I think you get the idea that we have quite a few things to think about at the OS level of our server model. What do we do? 🤔


A bunch of people came up with this idea that if we can't run multiple applications in one OS, let's just figure out a way to run more OSes on our physical servers. Thus, the hardware virtualization came into play.

![200x200](/public/img/docker/hard-virt.png)

The idea was that a special piece of software called Hypervisor abstracts the underlying hardware (virtualizes it) to provide the same hardware to several operating systems. Operating systems which run on top of Hypervisor talk to the virtual hardware presented by the Hypervisor using the generic drivers. Thus, we could have multiple OSes of different sorts running on the same server at the same time each using a piece of computing resources that are there.

Thanks to hardware virtualization, the OS became portable. We'd seen before how the operating system was bound to the vendor hardware. But now operating systems can't see the server's underlying hardware, instead they see the generic drivers and the generic hardware provided by the Hypervisor. This fact makes the process of distributing and running various OSes on any hardware very easy. All we need is a proper hypervisor running on that hardware or host OS.

Hardware virtualization allowed us to run in isolation multiple OSes on the same physical server and made the movement of an OS form one machine to another much easier.

Did it solve our problem with computing resource utilization? It sort of did. We can now run more instances our applications on a physical server by running more instances of operating systems.

But it doesn't really solve our initial problem with running multiple applications on the same OS. It seems more like a workaround.

What we really need is a technology that would allow us to run our applications in isolation on one OS.

And this is where containers come in.

Containers are an abstraction at the OS layer. With the help of containers, we can bundle the application code and all its dependencies into one standardized package and then run it in isolation from other system processes.

![200x200](/public/img/docker/container.png)


Isolation means everything here. For instance, filesystem isolation means that each container gets its own filesystem which solves the problem with version conflicts. Network isolation means each instance gets its own IP address and thus two containerized applications are free to bind to the same port, without us having to deal with any init scripts.

This really solves our problem with resource utilization. We can now run multiple instances of applications (even of the same type) in our OS and use our computing resources to the maximum.  

Since container technology is based on the features built into the Linux kernel, like cgroups and namespaces, we can only run containers on Linux distributions. Although, with the use of virtual machines, we can run containers on pretty much any OS (check how Docker works on Windows or OS X).

The Docker itself is a software container platform which allows us to create and manage containers.

Docker provides a standard format for creating container images. This makes it extremely easy to run and distribute our applications through various OSes. For instance, we can now build a container image with our application on CentOS and then run that same image on Ubuntu without having to change anything, because differences in OS distributions are abstracted away. The only thing we need is to have Docker installed.

But I want to note that having containers doesn't mean we don't VMs. They are still widely used and help us decrease hardware costs. They are often used in cases when we need to provide computing resources to different people. Virtual machines allows us to allocate the exact number of computing resources to meet the client's requirements, and if there's some resources left we can allocate it to another client. This is how public clouds work.

In fact, VMs and container nicely work together (＾＾)ｂ

![200x200](/public/img/docker/container-vm.png)


As for using VMs to provide isolation for our applications, VMs seem a bit too heavy. They are big in size (we're usually talking about gigabytes here) because they run the full OS which includes things like the kernel and hardware drivers.

On the other hand, containers don't need all that software the OS usually have. All they need is an application you want to run plus its dependencies. As a result, containers are much smaller in size (MBs against GBs) which means they're easier to build and distribute.  They are also start and stop quicker because there's no OS boot process required.

So containers are obviously a better choice for running applications in isolation. Although, VMs are still used for running and distributing applications when a higher level of security and isolation is required.

**This is all great, but I still don't get it why we have the ubuntu image in the Dockerifle? If a container provides isolation to the application and its dependencies, and uses the host operating system's kernel, then why do we need to specify an OS image in our Dockerfile** (◔_◔)???

Let's make things even more confusing before we answer that question.

If I build the image from the Dockerfile above and look at its size, here's what I will see.

![200x200](/public/img/docker/im-size.png)

Ubuntu image is only 188MB. 🤔

Have you seen ubuntu OS image of that size? If you try to search for ubuntu 14.04, you'll see that its size is about 1GB.

So why is ubuntu container is 5 times smaller than a normal ubuntu image?  

As we talked earlier, containers don't need an OS as they share the kernel and execute instructions on the host directly. So that ubuntu container images is stripped down image of a real operating system which doesn't include kernel, but does have some libs and utilities specific to Ubuntu distro.

This means that we actually don't need ubuntu container image to run my GO binary. If I try to build another container image (v2.0) for my test application from `scratch`, it should still work.
~~~yml
FROM scratch
COPY ./hello-world .
EXPOSE 8080
CMD [ "./hello-world" ]
~~~
`scratch` is a reserved docker image name which just means no-op.

![200x200](/public/img/docker/scratch.png)

Noticed how it skipped the first step and went straight to the next?


Now if I look at the image size of my new image, I'll see that it became significantly smaller - only 5 MB against 194 MB! And it still works as before.

<script type="text/javascript" src="https://asciinema.org/a/V5jj9QPoERF8YAEKcVCY5JRWH.js" id="asciicast-V5jj9QPoERF8YAEKcVCY5JRWH" async></script>

But why people still use OS container images anyway? One of the most common reasons for this is a package manager. When you write a Dockerfile to build your image, it's much easier to install required dependencies inside the container using a package manager in the RUN command than directly copying all the packages from your local machine to the container.

Another reason to use an OS image is it facilitates troubleshooting. For instance, OS image may have a shell installed, this way I can run a virtual terminal inside the container and then issue commands from inside.

Let's see if can I check internet connectivity from inside the containers that I've built.

<script type="text/javascript" src="https://asciinema.org/a/5Li4vveWKiL3IjB84415OGh0x.js" id="asciicast-5Li4vveWKiL3IjB84415OGh0x" async></script>

As you saw, I wasn't able to run a shell inside the version 2 of my container  which was built from scratch. But I could run shell and ping inside the version 1 of my container, because the parent container (ubuntu) has all the required binaries in it.

So using OS images when building your application container images is a common practice. But you should remember to choose the OS container image wisely to avoid bloating your application container image.

For further reading, I recommend checking out the [Alpine](https://docs.docker.com/samples/alpine/) Docker image which is one of the most popular these days. It's very lightweight (4-5 MB), it has a package manager and many useful utilities preinstalled.
