# How to deploy a Kogito Job Service that uses Kafka using the Kogito Operator

## 1. Start Minikube

In order to have a local Kubernetes cluster, you can use a Minikube instance that can be executed using the following command.
```
minikube start --cpus 8 --memory 16384
```


## 2. Setup a Kafka cluster with Strimzi

We need first of all to create a namespace for the Strimzi operator:

```
kubectl create namespace kafka
```

Than we can deploy it:

```
kubectl create -f 'https://strimzi.io/install/latest?namespace=kafka' -n kafka
```

And we can create our Kafka cluster instance:

```
kkubectl apply -f ./resources/kubernetes/kafka.yaml -n kafka
```

And wait for Kubernetes to deploy the required services

```
kubectl wait kafka/my-cluster --for=condition=Ready --timeout=300s -n kafka 
```

After a while, when all the needed resources will be deployed and up and running, you could verify with the command in [Appendix A](#appendix-a) that there are only system topics.

## 3. Install the Kogito Operator

Now it's time to install the Kogito Operator with the changes that you would like to test. 
For this reason I'm supposing that you will execute the next commands in the root folder of your branch of the Kogito Operator you are working on for developing your changes.

In order to understand better what you are doing with the following commands, you can refer to the Operator SDK [official documentation](https://sdk.operatorframework.io/docs/olm-integration/quickstart-bundle/).

```
make deploy-operator-on-ocp image=quay.io/kiegroup/kogito-operator:1.30.0
```

When the Kogito Operator will be available, we have to declare tha the Kafka instance is a relevant infrastructure component for the Kogito services that we will deploy with the KogitoOperator using the [KafakInfra](https://docs.jboss.org/kogito/release/latest/html_single/#_kogito_operator_dependencies_on_third_party_operators) CRD. I prepared a manifest for the KafkaInfra, you can find in the [resources folder](resources/kubernetes/kafka-infra.yml), you only need to apply it:

```
kubectl apply -f ./resources/kubernetes/kogito-infra.yml -n kogito-jobsservice
```

## 4. Deploy your Kogito service

It's finally time to deploy our Kogito Jobs service that produces/consumes messages using Kafka.

The command to deploy the service is:

```
kubectl apply -f ./resources/kubernetes/kogito-supportingservice.yml
```

When the Kogito Jobs service will be up and running, you could verify with the command in [Appendix A](#appendix-a) that in addition to the ```system``` topics there will be also the ones needed by the service (in particular my-custom-kogito-job-service-job-request-events).


## Appendix A
```
kubectl exec -n kafka -it my-cluster-kafka-0 -c kafka -- bin/kafka-topics.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --list
```

