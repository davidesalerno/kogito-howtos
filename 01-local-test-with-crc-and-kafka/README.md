# How to deploy a Kogito service locally that uses Kafka

## 1. Get a Kogito service using Kafka
You can find a good candidate for this in the Kogito Examples and in particular the [process-kafka-quickstart-quarkus](https://github.com/kiegroup/kogito-examples/tree/stable/kogito-quarkus-examples/process-kafka-quickstart-quarkus) one.

## 2. Build and push the Kogito service image on your docker registry

If you would like to use the mentioned Kogito example, you can find the changes needed to build an image [here](https://github.com/davidesalerno/kogito-examples/commit/65ed558d7e17a4a3e5063a56131a82968ffc8509).

I simply added the Quarkus Jib and Kubernetes extensions with the following commands:
```
quarkus extension add 'container-image-jib'
quarkus extension add 'quarkus-kubernetes'
```
Then I built an image
```
quarkus build -Dquarkus.container-image.build=true -DskipTests
```

Finally I pushed it on the docker registry (Quay.io) making it **public available**:
```
docker push quay.io/dsalerno/process-kafka-quickstart-quarkus:1.22.1.Final
```
## 3. Start CRC
 
I suppose you are able to start RedHat Openshift Local (formerly CodeReady Containers aka CRC), this is a mandatory requirement, so please refer to the [official documentation](https://developers.redhat.com/products/openshift-local/overview) to have it availble on your workstation.


Use the following command in order to have got enough resources:

```
crc start --memory 16384 --cpus 8
```

Setup your environment variables:

```
eval $(crc oc-env)
https://api.crc.testing:6443
```
and login to the OC web console ```https://console-openshift-console.apps-crc.testing/``` and create a project that we will use as a target for our test. 

Don't forget to select this as your default project for the OC cli in this way:
```
oc project <project name>
```

## 3. Setup a Kafka cluster with Strimzi

Using the OC web console, using the left sidebar got to Operators -> OperatorHub and install the Strimzi Operator using the default values (more details are available [here](https://docs.openshift.com/container-platform/4.10/operators/user/olm-installing-operators-in-namespace.html#olm-installing-from-operatorhub-using-web-console_olm-installing-operators-in-namespace) if needed).

Once the Strimzi Operato is up and running, we can proceed with the setup of the Kafka cluster using the deployment ```kafka.yml``` you can fin in the [resources](resources/kubernetes/kafka.yaml) folder.

```
oc apply -f ./resources/kubernetes/kafka.yaml
```

After a while, when all the needed resources will be deployed and up and running, you could verify with the command in [Appendix A](#appendix-a) that there are only system topics.

## 4. Build and Install your Kogito Operator

Now it's time to build and install the Kogito Operator with the changes that you would like to test. For this reason I'm supposing that you will execute the next commands in the root folder of your branch of the Kogito Operator you are working on for developing your changes.

In order to understand better what you are doing with the following commands, you can refer to the Operator SDK [official documentation](https://sdk.operatorframework.io/docs/olm-integration/quickstart-bundle/).

```
export USERNAME=<container-registry-username>
export VERSION=0.0.1
export IMG=quay.io/$USERNAME/kogito-operator:v$VERSION // location where your operator image is hosted
export BUNDLE_IMG=quay.io/$USERNAME/kogito-operator-bundle:v$VERSION // location where your bundle will be hosted
source source ~/.venvs/cekit/bin/activate // <-- you have to refer to a Python virtual env with CeKit available
make
make bundle
make bundle-build bundle-push
operator-sdk bundle validate $BUNDLE_IMG
operator-sdk run bundle $BUNDLE_IMG --install-mode=AllNamespaces --timeout=15m --namespace=openshift-operators
```

When the Kogito Operator will be available, we have to declare tha the Kafka instance is a relevant infrastructure component for the Kogito services that we will deploy with the KogitoOperator using the [KafakInfra](https://docs.jboss.org/kogito/release/latest/html_single/#_kogito_operator_dependencies_on_third_party_operators) CRD. I prepared a manifest for the KafkaInfra, you can find in the [resources folder](resources/kubernetes/kafka-infra.yml), you only need to apply it:

```
oc apply -f ./resources/kubernetes/kafka-infra.yml
```

## 5. Deploy your Kogito service

It's finally time to deploy our Kogito service that produces/consumes messages using Kafka.

If you would like to use the example I suggested at the beginning of this guide ([1](#1-get-a-kogito-service-using-kafka)), you can use the ```kogito-runtime.yml``` in the [resources folder](resources/kubernetes/kogito-runtime.yml) otherwhise you have to create a similar one based on your Kogito service paying particular attention to have into the ```spec``` this:

```
infra:
    - my-kafka-infra
```

The command to deploy the service is:

```
oc apply -f ./resources/kubernetes/kogito-runtime.yml
```

When the Kogito service will ne up and running, you could verify with the command in [Appendix A](#appendix-a) that in addition to the ```system``` topics there will be also the ones needed by the service (in case you used the suggested example, processedtravellers and travellers).


## Appendix A
```
oc exec -it my-cluster-kafka-0 -c kafka -- bin/kafka-topics.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --list
```

