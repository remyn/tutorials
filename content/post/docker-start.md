+++
date = "2016-08-16T11:32:49+12:00"
draft = false
title = "Starting with docker"
slug = "docker-start"
tags = ["docker","docker-machine", "docker-compose"]
image = "docker-machine.png"
comments = false # set false to hide Disqus
share = false # set false to hide share buttons
menu= "main" # set "main" to add this content to the main menu
author = "remyn"
featured = false
description = "A simple start for docker, using docker-machine and dokcer-compose."
+++

docker-machine : a simple tool to manage our virtual machines hosting our containers.
<!--more-->

# Docker

For the unlucky user that doesn't have Windows 10, we can't have a native docker daemon. Instead, we are going to use docker from a linux box.
Here we are going to create a virtual machine with a lightweigth linux set up with a docker daemon.

All the following command should be enter inside a `git-bash` command line tool. If you have installed Git support for your VS2015 it should be installed.
Just start one instance as administrator and then go to your home folder:

    cd $HOME


You can get the file used in this tutorial by cloning the repository [`tutorials`](https://github.com/remyn/tutorials-files):

    git clone https://github.com/remyn/tutorials-files


## Get the tools

First thing, we will need to load the 2 docker tools : `docker-machine` and `docker-compose`.
You can used the scripts [get-docker-machine.sh](https://github.com/remyn/tutorials-files/raw/master/get-docker-machine.sh) and [get-docker-compose.sh](https://github.com/remyn/tutorials-files/raw/master/get-docker-compose.sh) to download them into a given folder (e.g. __docker-demo__).

    ./tutorials/get-docker-machine.sh docker-demo
    ./tutorials/get-docker-compose.sh docker-demo
    
>Note: If no folder is passed the default `docker-bin` is used.

Great, so by now you should have the tools into the folder docker-demo.

![docker-exe-in-folder](images/docker-exe-folder.png "docker exe in folder.")


## Create the virtual machine

Let's create a virtual machine with docker on it !
Here we are going to use HyperV but the same could be acheived with the VirtualBox driver. Use the one that you have already installed on your computer.

>Note: It is one or the other, because once activated HyperV from your Windows features, VirtualBox can't function anymore! 

    cd docker-demo
    docker-machine create --driver hyperv docker-host

With HyperV, if you prefer using a specific network interface (e.g. WIFI), you can indicate a virtual-switch with the --hyperv-virtual-switch parameter:

    docker-machine create --driver hyperv --hyperv-virtual-switch "External LAN" docker-host


_That's it !_ The virtual machine has been setup with a lightweight linux footprint. The latest docker daemon has been set up and it is running.

**List** the virtual machine:

    docker-machine ls

**Stop** the virtual machine:

    docker-machine stop docker-host

**Start** the virtual machine

    docker-machine start docker-host

**Remove** (delete) the virtual-machine:

    docker-machine rm docker-host

Here we can see that a simple executable suffice to manage the virtual machine. You can use another graphic tool to manage your virtual machine and push container into: https://kitematic.com/ .


## Remote control the virtual machine

To remote login into the virtual machine, we need to set up the environment variable that docker will use to communicate with the daemon (IP, port ...):

    docker-machine env docker-host

The previous command do just a print of the export command to create the environment variables. So to really create them, just execute the command by copying the last line printed:

    eval $("C:\Users\remyn\bin\docker-machine.exe" env docker-host)

**Remote login** with the virtual machine:

    docker-machine ssh docker-host


## Windows docker client

While we can't have a native docker daemon on windows (8.1 and before), we can have a client one to natively send command to a docker daemon into a linux machine.

From an administartor console, simply install first [Chocolatey](http://chocolatey.org).

Then to install the client docker, just type:

    choco install docker

Then, you can simply communicate with teh docker daemon of your previous virtual machine:

    docker ps
