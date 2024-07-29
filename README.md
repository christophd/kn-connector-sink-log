# Knative connector log-sink

Knative eventing connector based on [Apache Camel Kamelets](https://camel.apache.org/camel-kamelets/).
The connector project creates a container image that is pushed into a registry so the image can be referenced in a Kubernetes deployment.

## Build the container image

The project uses Quarkus in combination with Apache Camel, Kamelets and Maven as a build tool.

You can use the following Maven commands to build the container image.

```shell
./mvnw package -Dquarkus.container-image.build=true
```

The container image uses the project version and image group defined in the Maven POM.
You can customize the image group with `-Dquarkus.container-image.group=my-group`.

By default, the container image looks like this:

```text
quay.io/openshift-knative/kn-connector-sink-log:1.0-SNAPSHOT
```

The project leverages the Quarkus Kubernetes and container image extensions so you can use Quarkus properties and configurations to customize the resulting container image.

See these extensions for details:
* https://quarkus.io/guides/deploying-to-kubernetes
* https://quarkus.io/guides/container-image

## Push the container image

```shell
./mvnw package -Dquarkus.container-image.build=true -Dquarkus.container-image.push=true
```

Pushes the image to the image registry defined in the Maven POM.
The default registry is [quay.io](https://quay.io/).

You can customize the registry with `-Dquarkus.container-image.registry=localhost:5001` (e.g. when connecting to local Kind cluster).

In case you want to connect with a local cluster like Kind or Minikube you may also need to set `-Dquarkus.container-image.insecure=true`.

## Kubernetes manifest

The build produces a Kubernetes manifest in (`target/kubernetes/kubernetes.yml`).
This manifest holds all resources required to run the application on your Kubernetes cluster.

The Kubernetes manifest includes:

* Service
* Deployment
* Trigger

You can customize the Kubernetes resources in [src/main/kubenretes/kubernetes.yml](src/main/kubernetes/kubernetes.yml).
This is being used as a basis and Quarkus will generate the final manifest in `target/kubernetes/kubernetes.yml` during the build.

## Deploy to Kubernetes

You can deploy the application to Kubernetes with:

```shell
./mvnw package -Dquarkus.kubernetes.deploy=true
```

This connects to the current Kubernetes cluster that you are connected with (e.g. via `kubectl config set-context --current --namespace $1`).

You may change the target namespace with `-Dquarkus.kubernetes.namespace=my-namespace`.

## Kamelet sink pipe

The sink consumes events from the Knative broker.
It uses a Pipe resource as the central piece of code to define how the Knative events are consumed and where the events get forwarted to.

The Pipe is a YAML file located in [src/main/resources/camel/kn-connector-sink-log.yaml](src/main/resources/camel/kn-connector-sink-log.yaml)

_kn-connector-sink-log.yaml_
```yaml
apiVersion: camel.apache.org/v1
kind: Pipe
metadata:
  name: kn-connector-sink-log
spec:
  source:
    ref:
      kind: Broker
      apiVersion: eventing.knative.dev/v1
      name: default
    properties:
      type: ""
  sink:
    ref:
      kind: Kamelet
      apiVersion: camel.apache.org/v1
      name: log-sink
```

This connector uses the [log-sink](https://camel.apache.org/camel-kamelets/log-sink.html) Kamelet and consumes events from the Knative broker. 

The Pipe always references a Knative broker as a source and connects to a Kamelet as a sink.

The name of the broker is always `default` because the Knative Trigger resource is responsible for connecting the application to the Knative broker.
The Trigger decides when to call the application as it provides the events to the application based on the trigger configuration.

This way the same container image can be used with different brokers and events (e.g. by adding filter criteria to the Trigger).
It is only a matter of configuring the Trigger resource that connects the application with the Knative broker.

You can find a sample Trigger in `src/main/kubernetes/kubernetes.yml`

```yaml
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  annotations:
    eventing.knative.dev/creator: kn-connectors
  labels:
    eventing.knative.dev/connector: log-sink
    eventing.knative.dev/broker: default
  name: kn-connector-sink-log
spec:
  broker: default
  subscriber:
    ref:
      apiVersion: v1
      kind: Service
      name: kn-connector-sink-log
    uri: /events
```

## Configuration

Each Kamelet defines a set of properties.
The user is able to customize these properties when running a connector deployment.

### Environment variables

You can customize the properties via environment variables on the deployment:

* CAMEL_KAMELET_LOG_SINK_SHOWHEADERS=false
* CAMEL_KAMELET_LOG_SINK_MULTILINE=false

You can set the environment variable on the running deployment:

```shell
kubectl set env deployment/kn-connector-sink-log CAMEL_KAMELET_LOG_SINK_MULTILINE=false
```

The environment variables that overwrite properties on the Kamelet sink follow a naming convention:

* CAMEL_KAMELET_{{NAME}}_{{ANY_PROPERTY_NAME}}

The name represents the name of the Kamelet sink as defined in the [Kamelet catalog](https://camel.apache.org/camel-kamelets/).

### ConfigMap and secrets

You may also mount a configmap/secret with some `application.properties`:

_application.properties_
```properties
# Kamelet log-sink defined properties
camel.kamelet.log-sink.showHeaders=true

# any other Kamelet sink property
camel.kamelet.log-sink.any-other-prop=value
```

## CloudEvent attributes

By default, the connector receives all events on the Knative broker.

You may want to specify filters on the CloudEvent attributes so that the connector selectively consumes events from the broker.
Just configure the Knative trigger to filter based on attributes:

_Knative trigger_
```yaml
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  annotations:
    eventing.knative.dev/creator: kn-connectors
  labels:
    eventing.knative.dev/connector: log-sink
    eventing.knative.dev/broker: default
  name: kn-connector-sink-log
spec:
  broker: default
  filter:
    attributes:
      type: dev.knative.connector.event.timer
  subscriber:
    ref:
      apiVersion: v1
      kind: Service
      name: kn-connector-sink-log
    uri: /events
```

The trigger for example filters the events by its type `ce-type=dev.knative.connector.event.timer`.

## Dependencies

The required Camel dependencies need to be added to the Maven POM before building and deploying.
You can use one of the Kamelets available in the [Kamelet catalog](https://camel.apache.org/camel-kamelets/) as a source or sink in this connector.

Typically, the Kamelet is backed by a Quarkus Camel extension component dependency that needs to be added to the Maven POM.
The Kamelets in use may list additional dependencies that we need to include in the Maven POM.

## Custom Kamelets

Creating a new kn-connector project is very straightforward.
You may copy one of the sample projects and adjust the reference to the Kamelets.

Also, you can use the Camel JBang kubernetes export functionality to generate a Maven project from a given Pipe YAML file.

```shell
camel kubernetes export my-pipe.yaml --runtime quarkus --dir target
```

This generates a Maven project that you can use as a starting point for the kn-connector project.

The connector is able to reference all Kamelets that are part of the [default Kamelet catalog](https://camel.apache.org/camel-kamelets/).

In case you want to use a custom Kamelet, place the `kamelet.yaml` file into `src/main/resources/kamelets`.
The Kamelet will become part of the built container image and you can just reference the Kamelet in the Pipe YAML file as a source or sink.

## More configuration options

For more information about Apache Camel Kamelets and their individual properties see https://camel.apache.org/camel-kamelets/.

For more detailed description of all container image configuration options please refer to the Quarkus Kubernetes extension and the container image guides:

* https://quarkus.io/guides/deploying-to-kubernetes
* https://quarkus.io/guides/container-image
