# Getting started with JMS on Kubernetes using the Quarkus Qpid JMS extension

This guide shows you how to send and receive messages using [Apache
Qpid JMS](http://qpid.apache.org/components/jms/index.html) and
[ActiveMQ Artemis](https://activemq.apache.org/artemis/index.html) on
[Kubernetes](https://kubernetes.io/) using a
[Quarkus Qpid JMS extension](https://github.com/amqphub/quarkus-qpid-jms).
It uses the [AMQP 1.0](http://www.amqp.org/) message protocol to send
and receive messages.

## Overview

The example application has three parts:

* An AMQP 1.0 message broker, ActiveMQ Artemis

* A sender service exposing an HTTP endpoint that converts HTTP
  requests into AMQP 1.0 messages.  It sends the messages to a queue
  called `example/strings` on the broker.

* A receiver service that consumes AMQP messages from
  `example/strings`.  It exposes another HTTP endpoint that returns
  the messages as HTTP responses.

The sender and the receiver use the JMS API to perform messaging
operations.

## Prerequisites

* Optional: For native builds without using docker, [GraalVM](https://www.graalvm.org/) version [19.1.1](https://github.com/oracle/graal/releases/tag/vm-19.1.1)+ [installed](https://www.graalvm.org/docs/getting-started), with `GRAALVM_HOME` set and [native-image extension](https://www.graalvm.org/docs/reference-manual/aot-compilation/).

* [Install and run Minikube in your
  environment](https://kubernetes.io/docs/setup/minikube/)

## Steps

1. Change directory to the sender application, build it to prepare for
   creating a docker image to deploy.

   ```bash
   $ cd sender/
   $ mvn package -Pnative -Dnative-image.docker-build=true
   [Maven output]
   cd ../
   ```

1. Change directory to the receiver application, build it to prepare for
   creating a docker image to deploy.

   ```bash
   $ cd receiver/
   $ mvn package -Pnative -Dnative-image.docker-build=true
   [Maven output]
   cd ../
   ```

1. Configure your shell to use the Minikube Docker instance:

   ```bash
   $ eval $(minikube docker-env)
   $ echo $DOCKER_HOST
   tcp://192.168.39.67:2376
   ```

1. Create a broker deployment and expose it as a service:

   ```bash
   $ kubectl run messaging --generator=run-pod/v1 --image docker.io/ssorj/activemq-artemis
   pod/messaging created
   $ kubectl expose pod messaging --type=ClusterIP --port=5672
   service/messaging exposed
   ```

1. Change directory to the sender application, create a
   deployment, and expose it as a service:

   ```bash
   $ cd sender/
   $ docker build -f src/main/docker/Dockerfile.native -t quarkus-jms-sender .
   [Docker output]
   $ kubectl apply -f kubernetes.yml
   service/quarkus-jms-sender created
   deployment.apps/quarkus-jms-sender created
   ```

1. Change directory to the receiver application, create a
   deployment, and expose it as a service:

   ```bash
   $ cd ../receiver/
   $ docker build -f src/main/docker/Dockerfile.native -t quarkus-jms-receiver .
   [Docker output]
   $ kubectl apply -f kubernetes.yml
   service/quarkus-jms-receiver created
   deployment.apps/quarkus-jms-receiver created
   ```

1. Check that the deployments and pods are present.  You should see
   deployments and services for `broker`, `sender`, and `receiver`.

   ```bash
   $ kubectl get deployment
   $ kubectl get service
   ```

1. Save the `NodePort` URLs in local variables:

   ```bash
   $ sender_url=$(minikube service quarkus-jms-sender --url)
   $ receiver_url=$(minikube service quarkus-jms-receiver --url)
   ```

1. Use `curl` to test the readiness of the send and receive services:

   ```bash
   $ curl $sender_url/health/ready
   {
    "status": "UP",
    "checks": [
   ]

   $ curl $receiver_url/health/ready
   {
    "status": "UP",
    "checks": [
   ]
   ```

1. Use `curl` to send strings to the sender service:

   ```bash
   $ curl -X POST -H "content-type: text/plain" -d hello1 $sender_url/api/send
   OK
   $ curl -X POST -H "content-type: text/plain" -d hello2 $sender_url/api/send
   OK
   $ curl -X POST -H "content-type: text/plain" -d hello3 $sender_url/api/send
   OK
   ```

1. Use `curl` to receive the sent strings back from the receiver
   service:

   ```bash
   $ curl $receiver_url/api/receive
   hello1
   $ curl $receiver_url/api/receive
   hello2
   $ curl $receiver_url/api/receive
   hello3
   ```

## More resources

* [Quarkus Getting Started](https://quarkus.io/get-started/)
* [Quarkus Qpid JMS extension](https://github.com/amqphub/quarkus-qpid-jms)
* [Getting started with Minikube](https://kubernetes.io/docs/tutorials/hello-minikube/)
* [Apache ActiveMQ Artemis](https://activemq.apache.org/artemis/)
* [Artemis container image](https://cloud.docker.com/u/ssorj/repository/docker/ssorj/activemq-artemis)
* [Apache Qpid JMS](http://qpid.apache.org/components/jms/index.html)
* [What's New in JMS 2.0 by Nigel Deakin](https://www.oracle.com/technetwork/articles/java/jms20-1947669.html)
* [AMQP](https://www.amqp.org/)
