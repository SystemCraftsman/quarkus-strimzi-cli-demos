Quarkus Strimzi Demo (with Strimzi Kafka CLI)
========================

This project illustrates how you can interact with Apache Kafka on Kubernetes (Strimzi) using MicroProfile Reactive Messaging.

## Kafka cluster (on OpenShift)

First you need a Kafka cluster on your OpenShift.

Create a namespace first:

```bash
 oc new-project kafka-quickstart
```

Then you will need Strimzi Kafka CLI by running the following command (you will need Python3 and pip):

```bash
 sudo pip install strimzi-kafka-cli
```

See [here](https://github.com/systemcraftsman/strimzi-kafka-cli) for other details about Strimzi Kafka CLI.

After installing Strimzi Kafka CLI run the following command to install the operator on `kafka-quickstart` namespace:

```bash
 kfk operator --install -n kafka-quickstart
```

When the operator is ready to serve, run the following command in order to create a Kafka cluster:

```bash
 kfk clusters --create --cluster my-cluster -n kafka-quickstart
```

A `vim` interface will pop-up. 
If you like you can change the broker and zookeeper replicas to 1 but I suggest you to leave them as is if your Kubernetes cluster have enough resources.
Save the cluster configuration file and respond `Yes` to make Strimzi CLI apply the changes.

Wait for the 3 broker and 3 zookeeper pods running in ready state in your cluster:

```bash
 oc get pods -n kafka-quickstart -w
```

When all pods are ready, create your `prices` topic to be used by the application:

```bash
 kfk topics --create --topic prices --partitions 10 --replication-factor 2 -c my-cluster -n kafka-quickstart
```

Check your topic is created successfully by describing it natively:

```bash
kfk topics --describe --topic prices -c my-cluster -n kafka-quickstart --native
```

## Deploy the application

The application can be deployed to OpenShift using: 

```bash
 ./mvnw clean package -DskipTests
```

This will take a while since the s2i build will run before the deployment.
Be sure the application's pod is running in ready state in the end. 
Run the following command to get the URL of the `Prices` page: 

```bash
echo http://$(oc get routes -n kafka-quickstart -o json | jq -r '.items[0].spec.host')/prices.html 
```

Copy the URL to your browser, and you should see a fluctuating price.

## Anatomy

In addition to the `prices.html` page, the application is composed by 3 components:

* PriceGenerator
* PriceConverter
* PriceResource

We generate (random) prices in `PriceGenerator`.
These prices are written in `prices` Kafka topic that we recently created. 
`PriceConverter` reads from the prices Kafka topic and apply some magic conversion to the price. 
The result is sent to an in-memory stream consumed by a JAX-RS resource `PriceResource`.
The data is sent to a browser using server-sent events.

![](https://quarkus.io/guides/images/kafka-guide-architecture.png)

The interaction with Kafka is managed by MicroProfile Reactive Messaging.
The configuration is located in the application configuration.

## Running and deploying in native

You can compile the application into a native binary using:

```bash
mvn clean install -Pnative
```

or deploy with:

```bash
 ./mvnw clean package -Pnative -DskipTests
```

This demo is based on the following resources:

- https://quarkus.io/guides/kafka
- https://github.com/quarkusio/quarkus-quickstarts/tree/main/kafka-quickstart