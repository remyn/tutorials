+++
date = "2016-08-22T15:33:35+12:00"
draft = false
title = "Starting with kafka"
slug = "kafka-start"
tags = ["docker","kafka"]
image = "kafka.png"
comments = false # set false to hide Disqus
share = false # set false to hide share buttons
menu= "main" # set "main" to add this content to the main menu
author = "remyn"
featured = false
description = "A simple start with kafka."

+++

A simple start with kafka, setting up a cluster with 3 nodes.
<!--more-->

Kafka is a distributed messaging system, that decouples publication and consumption of the messages.
Once a message is published, you can consume it and replay as many times as you want.
You can read the [quickstart](http://kafka.apache.org/documentation.html#quickstart) of Kafka to apprehend the basic concept. It is very well explained.


We are going to set up a basic Kafka cluster ready to use, for `development` perspective. We are going to use containers.
In the context of development we don't want to persist indefinitely the messages. 
It is why, in this configuration as soon the containers are stopped, the messages disappear.
If for some reason, you don't want this behavior, you should configure the kafka container against a specific volume to persist the messages behind the container lifecycle.


## Create a Kafka cluster

First, if it is not the case, you can create a virtual machine to handle our Kafka cluster - named kafka-cluster (using docker-machine).
Refer to [Docker start]({{< relref "docker-start.md" >}}), if necessary.

    docker-machine start kafka-cluster
    docker-machine env kafka-cluster
     (don't forget to execute the eval from the last line printed)
    export DOCKER_IP=$(docker-machine ip kafka-cluster)

The previous commands just ensure the virtual machine is running, and our docker client is connected to the docker daemon of the virtual machine. 

Now, we just need to create the cluster with `docker-compose`.

If you didn't get the [repository](https://github.com/remyn/tutorials-files), you can download directly the composition file [compose-kafka.yml](https://github.com/remyn/tutorials-files/raw/master/compose-kafka.yml).

    docker-compose -f compose-kafka.yml up -d

The composition file instructs docker to setup the following:

 * 1 x Zookeeper
 * 1 x Kafka node

You can check the logs and containers.
So by now we have Kafka operational.

### Scale to 3 nodes

But if we want to test replication of messages we will need more nodes.

If we want 3 nodes in total, just ask docker to scale the kafka container:

    docker-compose -f compose-kafka.yml scale kafka=3


By now, you have 3 nodes, that could replicate messages between them, simulating a realistic production environment.

So, you can experiment some messaging using some client library like [rdkafka-dotnet](https://github.com/ah-/rdkafka-dotnet).

## Create a Kafka shell

If you want to check that everything as been setup properly. We can use the default command line tools that is coming with Kafka.

To simplify the passage of parameters to the command line, the Kafka container already define easy to use variables and scripts.
It is why to use the command line we are going to start another kafka container.
If you didn't get the [repository](https://github.com/remyn/tutorials-files), you can download directly the script file [start-kafka-shell](https://github.com/remyn/tutorials-files/raw/master/start-kafka-shell.sh).

    docker-machine ssh kafka-cluster
    export HOST_IP=$(ifconfig eth0 | awk '/inet addr/{split($2,a,":"); print a[2]}')
    start-kafka-shell.sh $HOST_IP SHOST_IP:2181

The second command is just to retreive the IP Address of the Host, to be able to pass it to the Kafka container used for the shell.
The last command just start another kafka container, and open an SSH connection inside it. All the variables will be set up.

## Create a topic

First we need to create a Topic that will persist the messages.

    $KAFKA_HOME/bin/kafka-topics.sh --create --topic atopic --partition 4 --zookeeper $ZK --replication-factor 1

Check the topic exists:

    $KAFKA_HOME/bin/kafka-topics.sh --list --zookeeper $ZK

Get the details of the topic:

    $KAFKA_HOME/bin/kafka-topics.sh --describe --topic atopic --zookeeper $ZK

>Note: **$ZK** is a variable pointing to Zookeeper that has been set up properly.


## Produce some messages

So, now push some message into the topic:

    $KAFKA_HOME/bin/kafka-console-producer.sh --topic atopic --broker-list= $(broker-list.sh)

>Note: **broker-list.sh** is a script that retreive automatically the IP address and port of the nodes available.

Every line that you enter is pushed as a message into the topic.


## Consume messages

If you want to see the message consumed in the same time, you can open another console, with the following command:

    $KAFKA_HOME/bin/kafka-console-consumer.sh --topic atopic --zookeeper=$ZK --from-beginning

>Notice: **from-begining** asks Kafka to replay the messages from the start of the topic when you start consuming the messages.

