#Khelg 2015:1 docker asp.net lab

In this lab we will explore some core container concepts with the help of asp.net vNext and boot2docker. Because of the limited docker support in Windows at this time, please clone this project somewhere into your home directory on your computer.

## Prerequisites

* ASP.NET vNext tools - either [Visual Studio 2015 CTP](http://www.visualstudio.com/en-us/downloads/visual-studio-2015-downloads-vs.aspx) or [KRE](https://github.com/aspnet/home) with [your favorite editor](http://www.omnisharp.net/) (optionally [yeoman](http://yeoman.io/) with aspnet-generator )
* [boot2docker for windows](https://github.com/boot2docker/windows-installer/releases)
* [VirtualBox](https://www.virtualbox.org/)
* You also need to [disable Hyper-V for Windows](http://www.hanselman.com/blog/SwitchEasilyBetweenVirtualBoxAndHyperVWithABCDEditBootEntryInWindows81.aspx) to run VirtualBox. 
* [cmder](https://bliker.github.io/cmder/) - Optional. But recommended for a sane terminal experience

## Starting up

Open up your terminal and run `boot2docker start` to create a virtual machine that will run docker. 
When done, you will be logged into the machine, but if that doesn't happen, run `boot2docker ssh` to login.

Run `docker ps` inside the virtual machine and you should get an empty list of docker containers

Run `docker run hello-world` and you will see the output from a process running inside an app. Note that running `docker ps` after `docker run hello-world` still returns an empty list. _A docker container will stop running when the process inside exits_

Run `docker ps --all` to see all containers that have run. The `hello-world` has a status with `exited(0)`, which is standard for processes that exit without errors.

To see the output of a container - running or not - you can run `docker logs <image_id_or_name>`.

## The ASP.NET image

Now download the microsoft ASP.NET image by typing `docker pull microsoft/aspnet`. You'll notice that running this image will return immediately since it is only meant to be the base for containers

We can run a command in the container by sending it as a parameter after the image name, and when we supply the `-it` flag we get that command ran in an **i**nteractive **t**erminal. Thus, to get a shell inside the container, simply run `docker run -it microsoft/aspnet /bin/bash`. You'll notice that inside this image we have access to the asp.net vNext runtime in form of the commands `k` and `kpm`. 

If you open up another terminal with boot2docker and run `docker ps` you should be able to see the microsoft/aspnet container running - the name of the container is randomly generated, but if you want to set it yourself you can supply a `--name <name>` parameter to `docker run`

When you are done playing in the container then type `exit` (or press Ctrl+D) to return  to the docker console.

## Running ASP.NET projects inside the container

From the docker terminal, cd into the `src\AspNet.Docker` directory of this project. Your home folder is mounted in the under `/c/Users`.

Now we want to bring our code into the container to try to run it from there. To do that, we'll make use of docker's _volume mounting_, where we can specify a folder on the host filesystem the should be accessible inside the docker host as a folder. We do this with the `-v` flag, which should be supplied as `-v FOLDER_ON_HOST_MACHINE:FOLDER_IN_CONTAINER`. You can get the name of the directory your standing in simply from the  `$PWD` variable. A UNIX standard place for putting a server is under the `/srv` directory. 

Run a bash shell inside the container where you mount `src/AspNet.Docker` under `/srv/aspnet`and run the following commands inside that folder to run your project in a Kestrel server. __N.B. the Kestrel implementation is not quite complete and won't support Ctrl+C after it has started. To kill it you must run `sudo killall mono` from another boot2docker terminal__

`kpm restore`

`k kestrel`

Now you should be able to make changes in your code repo while and see them reflected in your container immediately.

## Customizing the image with a Dockerfile ##

Up until this point we have been doing great just playing around inside the Microsoft ASP.NET container. However, containers by default are not allowed to open any network ports, so we need to modify it to open up port 5004 that kestrel is listening on.

Create a file called `Dockerfile` in `src/AspNet.Docker` write the following

```
FROM microsoft/aspnet

EXPOSE 5004
```


The Dockerfile is the recipe from which docker containers are built. In this case we will create a container that is based on `microsoft/aspnet`, but with the change that it listens to port 5004. 

To build this container simply run `docker build -t <yourname>/<imagename> .` inside the directory with the `Dockerfile`. The `-t` flag supplies the name of the image, as you might have guessed. 

After the build has finished, you can run  `docker images` to see what images you have on your system - you should be able to see the image you just built in that list

If you now run a server the same way you did before, you should be able to use another boot2docker terminal to connect to the kestrel server.

To see what ports your containers are exposing, you can use the `docker port <container>`. To get all the details of the container - including its IP address - use `docker inspect <container>` ( The output is quite large so you might want to pipe it to `grep IPAddress`) Once you have the addresss, you should be able to run `curl <ip_address>:5004` and see the HTML of the ASP.NET welcome page.

To get the port forwarded even further, we can supply the `-p` flag to `docker run` to map ports between the host and the container - not entirely unlike the `-v` flag. Invoke the option with `-v HOST_PORT:CONTAINER_PORT`.

If you run the docker image with the exposed port, and forward it to the host, you can then run `boot2docker ip` from a terminal  and open up that `<boot2docker_ip>:<forwarded_port>` in your web browser to see the asp.net welcome page

## Automating build and run using the Dockerfile ##

So far we have have only used to Dockerfile to expose a port, but we can of course use it for so much more. The natural step is that the container should both install all dependencies via `kpm restore` and also start the server with `k kestrel`

So let's start adding to the Dockerfile until it does. You can see the full Dockerfile reference see https://docs.docker.com/reference/builder/ . The commands you will most likely use are: `ADD`, `WORKDIR`,`RUN` and `ENTRYPOINT`

Once you are done (or think you are), just run `docker build -t <yourname>/aspnet` again and see how it works. Note that you can tag different images you create just by appending `:<tag>` to the image name. 

To run your images, simply use `docker run` like before, but remember to remove the `-v` since your Dockerfile now should add files to the server dir as part of the build. However, you should still use -v when you are developing and want to test out your code in the container.

__N.B. again, due to issues with the kestrel server - it _must_ have a terminal running to start properly, so you still have to run it with the `-it` flag__ _(After it has been started you can kill the docker process that started it though and it will keep running)_

## Building a (more) generic base image for all asp.net vNext projects ##

There's a special Dockerfile command called `ONBUILD` that prefixes other commands that ONLY runs if the image is used as a base image for another image (i.e. when specified in `FROM`). Can you create an image that makes is possible for us to only write `FROM <image>` in our Dockerfile and get the server installed and up and running?

## Bonus meals ##

* Share data between containers: https://docs.docker.com/userguide/dockervolumes/

* Link containers together: https://docs.docker.com/userguide/dockerlinks/

* Docker on Azure with powershell: http://blogs.technet.com/b/privatecloud/archive/2014/07/17/configuring-docker-on-azure-with-powershell-dsc.aspx

* Docker on Azure with xplat-cli: http://azure.microsoft.com/en-us/documentation/articles/virtual-machines-docker-with-xplat-cli/

* Distributed containers with Service discovery: https://medium.com/@dan.ellis/you-dont-need-1mm-for-a-distributed-system-70901d4741e1
