# How to Run

This is a step-by-step guide how to run the example:

## Installation

* The example is implemented in Java. See
   https://www.java.com/en/download/help/download_options.xml. 
   If you want to compile the example and create your own images, you need to install a JDK (Java
   Development Kit). After the installation you should be able to execute
   `java` and `javac` on the command line. If you don't plan to bake your own images, java is not needed.

* The example run in Docker Containers. You need to install Docker
  Community Edition, see https://www.docker.com/community-edition/
  . You should be able to run `docker` after the installation.

* The example need a lot of RAM. You should configure Docker to use 4
  GB of RAM. Otherwise Docker containers might be killed due to lack
  of RAM. On Windows and macOS you can find the RAM setting in the
  Docker application under Preferences/ Advanced.

* A running Kubernetes cluster is needed to run the Strimzi Kafka Operator (https://strimzi.io/) and our order processing application.
  
* After installing Docker you should also be able to run
  `docker-compose`. If this is not possible, you might need to install
  it separately. See https://docs.docker.com/compose/install/ .

## Build

**If you want to build your own docker images of order process application, continue with this section, if not then skip to [Running Strimzi Kafka operator](#running-strimzi-kafka-operator) section.**   
Change to the directory `microservice-kafka` and run `./mvnw clean
package` or `mvnw.cmd clean package` (Windows). This will take a while:

```
./mvnw clean package -Dmaven.test.skip=true
```

If this does not work:

* Ensure that `settings.xml` in the directory `.m2` in your home
directory contains no configuration for a specific Maven repo. If in
doubt: delete the file.

* Skip the tests: `./mvnw clean package -Dmaven.test.skip=true` or
  `mvnw.cmd clean package -Dmaven.test.skip=true` (Windows).

## Bake the Images

**If you want to build your own docker images of order process application, continue with this section, if not then skip to [Running Strimzi Kafka operator](#running-strimzi-kafka-operator) section.**    
First you need to build the Docker images. Change to the directory
`docker` and run `docker-compose build`. This will download some base
images, install software into Docker images and will therefore take
its time:

```
docker-compose build
```

Afterwards the Docker images should have been created. They have the prefix
`mskafka`:

```
docker images 
REPOSITORY                                              TAG                 IMAGE ID            CREATED             SIZE
mskafka_invoicing                                       latest              1fddb3132141        43 seconds ago      214MB
mskafka_shipping                                        latest              7340d766ea6f        46 seconds ago      214MB
mskafka_order                                           latest              0f9848e55054        49 seconds ago      215MB
mskafka_kafkacat                                        latest              461e8b02bb99        12 days ago          113MB
mskafka_postgres                                        latest              2b2f4f035d6d        12 days ago          269MB
```

## Push the images to Repository

To push the images baked above to your repository, first tag the images :

```
docker image tag mskafka_order vishnuhdadhich/mskafka_order:latest
```

Then, login to your registry with `docker login` and push the images :

```
docker push vishnuhdadhich/mskafka_order:latest
```

## Running Strimzi Kafka operator

To run the Strimzi Kafka operator in your K8s cluster, we will use OperatorHub.io, 
see https://operatorhub.io/operator/strimzi-cluster-operator . 
Install Operator Lifecycle Manager (OLM), a tool to help manage the Operators running on your cluster. Platforms like OpenShift / OKD will have it pre-installed.

```
$ curl -sL https://github.com/operator-framework/operator-lifecycle-manager/releases/download/0.10.0/install.sh | bash -s 0.10.0
```

Install the operator by running the following command:

```
kubectl create -f https://operatorhub.io/install/strimzi-cluster-operator.yaml
```

This Operator will be installed in the "operators" namespace and will be usable from all namespaces in the cluster. After install, checkout the custom resource definitions (CRDs) introduced by this operator to start using it.

## Create the Kafka Cluster and a Topic

The CRDs for Kafka cluster and a topic have been included in this repo. To apply them :

```
kubectl apply -f kafka.yaml
kubectl apply -f topic.yaml
```

Check if your Kafka pods are running by `kubectl get pods` command.

## Test the cluster 

Commands to test the cluster :

Produce some message :
```
kubectl run kafka-producer -ti --image=strimzi/kafka:0.11.4-kafka-2.1.0 --rm=true --restart=Never -- bin/kafka-console-producer.sh --broker-list my-cluster-kafka-bootstrap:9092 --topic order
```

Consume the message produced above :
```
kubectl run kafka-consumer -ti --image=strimzi/kafka:0.11.4-kafka-2.1.0 --rm=true --restart=Never -- bin/kafka-console-consumer.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --topic order --from-beginning
```

Use `ctrl+c` to come back to your console.

## Deploy the Order processing application

```
kubectl apply -f k8s-specifications.yaml
```

If any of the pods is not running, you can look at its logs using
e.g.  `kubectl logs <podname>`. The name of the pod is
given in the last column of the output of `kubectl get pods`. Looking at the
logs even works after the container has been
terminated. If the log says that the container has been `killed`, you
need to increase the RAM assigned to Docker to e.g. 4GB.
  
If you need to do more trouble shooting open a shell in the container
using e.g. `kubectl exec -it <podname> bash`.

## Access the UI

You can now go to http://public-ipaddress:31000/ and enter an order. That will
create a shipping and an invoice in the other two microservices.

You can terminate all containers using `kubectl delete -f k8s-specifications.yaml`.