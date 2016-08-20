+++
date = "2016-08-18T14:30:58+12:00"
draft = false
title = "Load balance containers"
slug = "load-balance"
tags = ["docker","mesos", "marathon", "traefik"]
image = "docker-mesos-marathon.png"
comments = false # set false to hide Disqus
share = false # set false to hide share buttons
menu= "main" # set "main" to add this content to the main menu
author = "remyn"
featured = false
description = "A simple way to load balance containers inside a cluster."
+++

A simple way to load balance containers inside a cluster.
<!--more-->



First start or setup a virtual machine that will act as our cluster. If necessary, refer to [Docker start]({{< relref "docker-start.md" >}}).

In the following, we will use the virtual machine created before named `docker-host`. And we assume, you have set up the environment variable to connect to the docker daemon of the virtual machine.

## Create the cluster

We are going to create another environment variable containing the IP address of our virtual machine.

It will be useful for our cluster creation. For that purpose, we use docker-machine to tell us what is the IP address:

    export DOCKER_IP=$(docker-machine ip docker-host)

Now, we just need to create the cluster with `docker-compose`.

If you didn't get the [repository](https://github.com/remyn/tutorials-files), you can download directly the composition file [compose-cluster.yml](https://github.com/remyn/tutorials-files/raw/master/compose-cluster.yml).

    docker-compose -f compose-cluster.yml up -d

The composition file instruct docker to setup the following:

 * 1 x Zookeeper
 * 1 x Mesos server
 * 1 x Mesos agent
 * 1 x Marathon
 * 1 x Traefik

The first time, it will be a little longer because it needs to pull the container images from the public repository of docker.

list the containers

    docker ps

You can also check the logs of the containers to be sure, everything has been set up properly: 

    docker logs docker_demo_zk_1
    docker logs docker_demo_master_1
    docker logs docker_demo_slave_1
    docker logs docker_marathon_1
    docker logs docker_demo_traefik_1


## Mesos, Marathon, Traefik

Mesos manage your computing resources - typically cpu, ram. Every agent reports the resources available.
You can have more details at the [mesos website](http://mesos.apache.org/).

Marathon will schedule the containers against the resources.
You can have more details at the [marathon website](https://mesosphere.github.io/marathon/).

Traefik is a dynamic reverse proxy. It will redirect the HTTP traffic to the appropriate container. It will act as a load balancer.
You can find more information on the [website](http://traefik.io/).

Now there are all up and running you can check there web interfaces:

    "C:\Program Files (x86)\Mozilla Firefox\firefox.exe" http://$(echo $DOCKER_IP):5050 for Mesos
    "C:\Program Files (x86)\Mozilla Firefox\firefox.exe" http://$(echo $DOCKER_IP):8080 for Marathon
    "C:\Program Files (x86)\Mozilla Firefox\firefox.exe" http://$(echo $DOCKER_IP):8088 for Traefik


_That's it !_ We have a cluster running locally simulated by containers. But it will be the same principle, if you have several instances of mesos agent as full EC2 instances for examples. In our current local cluster an full EC2 instance is simulated by a agent container. 


## Load balancing

Inside the composition file, we also started a simple rest application, named `whoami` that just reply to a GET pushing back its local IP. It is handy to check load balancing.
We have 2 instances (I mean - container)  of the same application. 

    curl -H Host:whoami.docker.localhost http://$(echo $DOCKER_IP)

If you execute the previous command several time, you will see that each request is handle by the 2 deployed containers alternatively.

>Notice, that we don't communicate directly with the container here. We simply send the request to the HOST IP address.
>Traefik will automatically redirect the request to the container that can handle the request. It balances automatically between the number of containers running.
>Traefik updates in real time the configuration. No need to stop the service, update the configuration file and restart it when you scale the containers up or down.
>Just start more or stop containers and Traefik will automatically notice the difference.


## Creating our own asp.net core container

It is easy to use some already created container. What about creating our own one.

_Let's do it!_
For the purpose of this exercise, we are going to create a simple ASP.NET core (WEBAPI) that will take a json payload on a POST - just to have more fun - and reply back with the IP address of the host. Once again it is handy to test load balancing.

### Return the IP Address of the container
You can get the solution of the [project](https://remyn.github.com/tutorials-files/payrun). Or alternatively, just create your own ASP.NET Core and in one of of your controller insert the IP address:

    var request = UriHelper.GetEncodedUrl(HttpContext.Request);
    HttpContext.Response.Headers.Add("From HOST", GetIP());

    private string GetIP()
    {
        return String.Join(",", NetworkInterface.GetAllNetworkInterfaces()
                     .SelectMany(i => i.GetIPProperties().UnicastAddresses)
                     .Select(i => i.Address)
                     .Where(i => i.AddressFamily == System.Net.Sockets.AddressFamily.InterNetwork && !IPAddress.IsLoopback(i))
                     .Select(i => i.ToString()));
    }


So now we have an application that is able to handle some HTTP Request and reply back adding the IP Address of the host into the header.
We can debug it, run it on our windows computer.

### Create the container

Here, we have an ASP.NET Core application. I mean it is a cross platform application that can be self hosted (using kestrel). It can be executed on Windows or Linux.
We can create an container that we can deploy either on Windows or on Linux. It is a deployment choice not a compilation one.

_So let's create the container._

First, if it is not already the case, put a dockerfile - named dockerfile without extension - into the project folder, that looks like the following:

    FROM microsoft/dotnet:latest
    COPY . /app
    WORKDIR /app
    RUN ["dotnet", "restore"]
    RUN ["dotnet", "build"]
    EXPOSE 5000/tcp
    ENTRYPOINT ["dotnet", "run", "--server.urls", "http://*:5000"]

In summary, it is an instruction file for Docker to build the container image. The file could be read as:

* Start with the official Microsoft .Net Core image.
* Copy the all the project into the folder app of the container.
* Move to this folder.
* Execute dotnet restore => Will load all the depnedencies of the project.
* Execute dotnet build => I think it is clear enough!
* The container will expose the port 5000 other TCP. Our application is listening on this port.
* When the container start, execute dotnet run --server.urls http://*:5000. In other words, run our application listening from any IP address on the port 5000.

To build the container, simply:

    cd to-your-project-folder
    docker build -t payrun

**What is happening here?**
First we are using the windows docker client that you have installed with Chocolatey in the [previous article]({{< relref "docker-start.md" >}}).
Then, as we have already setup the environment variables to connect with the docker daemon of our host, docker following the instructions of `dockerfile`, transfer the project files to the linux box and create a container named `payrun`.

You can check the container images with:

    docker images

Now you should have a container image - an imuutable artifact - that can be deploy anywhere on any host Windows or Linux. 


### Deploy the container

Let's use our container. To manually start our container, simply:

    docker run -t -d --name payrun_1 -p 5000:5000 payrun

With the previous command line, we ask docker to start the container named payrun, give the instance name payrun_1, match the port 5000 of the host to the port 5000 of the container.
And by the way, give back the control (with the -d) parameter.


Contol it is running:

    docker ps

Check its logs:

    docker logs payrun_1



### Test our application

Our application should be reachable on the port 5000 inside the container.
Let's try a direct call to the application. We deployed the container by matching the port 5000 of the host to the same number inside the container. So a request to the host IP using the port 5000, should end up inside the container.

    curl -X POST -i -H "Content-Type: application/json" -d "{ 'Name':'Payrun name', 'Schedule': 'Schedule name' }" http://$(echo $DOCKER_IP):5000/payruns 

_Voila!_  You should see the answer with the header containing the IP address of the container.


### Checking the dynamicc proxy

According to what we said before, Traefik should have seen the new container, registered it automatically and be ready to transfer requests to it.

First check its registration, got to the Traefik web interface - port 8088 of your host. You should see under the docker tab, the service `payrun_1` registered with a frontend rule named `Host:payrun_1.docker.localhost`.

>Note: as we didn't specifically assigned a label to the container Traefik automatically give it one based on convention.

Let's try it.

    curl -X POST -i -H "Content-Type: application/json" -H "Host:payrun_1.docker.localhost" -d "{ 'Name':'Payrun name', 'Schedule': 'Schedule name' }" http://$(echo $DOCKER_IP):80/payruns

>2 things to notice here:

>* We are using the port 80, not the port 5000.
>* We are incorporating into the header the frontend rule.

Because we have Traefik listening on the port 80 of the cluster IP Address. We just need to send the request to the `cluster`, mentioning into the header the `service` that need to handle the request.
Traefik will automatically redirect the request to the appropriate `container`. We don't need to know on which instance is the container,  what is the IP address of the container or even the port.

Sometimes there is awesome **magic** tool!


## Scale our container

What about scaling up or down? Marathon will manage that for us.
We can ask Marathon to ensure that 3 instances of our application is running at all times.
Create a deployment file `marathon-payrun.json`, declariung the cpu, ram usage and the container parameters, taht looks like:

    {
        "id": "marathon-payrun",
        "cpus": 0.1,
        "mem": 64.0,
        "instances": 3,
        "container": {
            "type": "DOCKER",
            "docker": {
            "image": "payrun",
            "network": "BRIDGE",
            "portMappings": [
                { "containerPort": 5000, "hostPort": 0, "protocol": "tcp" }
            ]
            }
        },
        "healthChecks": [
            {
            "protocol": "HTTP",
            "portIndex": 0,
            "path": "/api/values",
            "gracePeriodSeconds": 5,
            "intervalSeconds": 20,
            "maxConsecutiveFailures": 3
            }
        ],
        "labels": {
            "traefik.weight": "1",
            "traefik.protocol": "http",
            "traefik.frontend.rule" : "Host:payrun.marathon",
            "traefik.frontend.priority" : "10"
        }
    }

Now, create a deployment on marathon using this file:

    curl -X POST http://$(echo $DOCKER_IP):8080/v2/apps -d @marathon-payrun.json -H "Content-type: application/json"

The previous command use the API interface, but the same action could be done thru the web interface.

* Go on the Marathon web interface (port 8000) and check the deployment and the instances.
* Go on the Traefik web interface (port 8088), on the Marathon tab, check the new 3 instances are registered.
* Go on the Mesos web interface (port 5050) and check the resources used (3 x 0.1 cpu and 3 x 64 MB).


Did you notice, the frontend rule label specified inside the marathon file : `traefik.frontend.rule" : "Host:payrun.marathon`.
So want to use this new `service` deployed? 

    curl -X POST -i -H "Content-Type: application/json" -H "Host:payrun.marathon" -d "{ 'Name':'Payrun name', 'Schedule': 'Schedule name' }" http://$(echo $DOCKER_IP):80/payruns

As previously, we are sending the request to the port 80 of the host.Traefik will do the job to know how to reach and load balance the service. Repeat the previous command several times and check the IP Address of the container to see that each instance is used alternatively.


Want more instance? Just scale the deployment from the Marathon interface or from its API. As long as Mesos reports enough resources, you will have what you want.

Finally, you check what your services are really consuming by checking the containers:

    docker stats 

