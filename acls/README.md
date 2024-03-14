# Kafka ACLS

- [Kafka ACLS](#kafka-acls)
  - [Description](#description)
  - [Environment](#environment)
  - [Namespace](#namespace)
  - [SASL Authentication](#sasl-authentication)
    - [Broker](#broker)
    - [Client](#client)
  - [Configuring Kafka for ACLs](#configuring-kafka-for-acls)
    - [Broker](#broker-1)
    - [Admin Client](#admin-client)
  - [Confluent Kafka](#confluent-kafka)
    - [Service Account (kind: ServiceAccount)](#service-account-kind-serviceaccount)
    - [Headless Service (kind: Service)](#headless-service-kind-service)
    - [StatefulSet (kind: StatefulSet)](#statefulset-kind-statefulset)
    - [Deploy](#deploy)
    - [Verify communication across brokers](#verify-communication-across-brokers)
  - [Testing Kafkfa ACLs](#testing-kafkfa-acls)

## Description

Deploying and running the Community Version of Kafka packaged with the Confluent Community download and configured to use [SASL/PLAIN authentication](https://docs.confluent.io/platform/current/kafka/authentication_sasl/authentication_sasl_plain.html) and [ACLS](https://docs.confluent.io/platform/current/kafka/authorization.html).

## Environment

| Technology | Version |
| --- | --- |
| Minikube | v1.29.0 |
| Docker | v23.0.5 |
| Kubernetes | v1.26.1 |
| [cp-kafka](https://hub.docker.com/r/confluentinc/cp-kafka) | 7.5.0 |

## Namespace

This [yaml file](./00-namespace.yaml) defines a [namespace](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) for running Kafka in a Kubernetes cluster.
It isolates Kafka resources within a dedicated namespace for better organization and management.

## SASL Authentication

We need to configure brokers and clients to use `SASL` authentication. Refer to [Kafka Broker and Controller Configurations for Confluent Platform](https://docs.confluent.io/platform/current/installation/configuration/broker-configs.html) page for detailed explanation of the configurations used here.

### Broker

1. Enable SASL/PLAIN mechanism in the `server.properties` file of every broker.

    ```yaml
    # List of enabled mechanisms, can be more than one
    - name: KAFKA_SASL_ENABLED_MECHANISMS
      value: PLAIN
    
    # Specify one of of the SASL mechanisms
    - name: KAFKA_SASL_MECHANISM_INTER_BROKER_PROTOCOL
      value: PLAIN
    ```

2. Tell the Kafka brokers on which ports to listen for client and interbroker `SASL` connections. Configure `listeners`, and `advertised.listeners`:

    ```yaml
    - command:
    ...
    export KAFKA_ADVERTISED_LISTENERS=SASL://${POD_NAME}.kafka-headless.kafka.svc.cluster.local:9092
    ...
    
    env:
    ...
    - name: KAFKA_LISTENER_SECURITY_PROTOCOL_MAP
      value: "CONTROLLER:PLAINTEXT,SASL:SASL_PLAINTEXT"
    - name: KAFKA_LISTENERS
      value: SASL://0.0.0.0:9092,CONTROLLER://0.0.0.0:29093
    ```

3. Configure JAAS for the Kafka broker listener as follows:

    ```yaml
    - name: KAFKA_LISTENER_NAME_SASL_PLAIN_SASL_JAAS_CONFIG
      value: org.apache.kafka.common.security.plain.PlainLoginModule required username="admin" password="admin-secret" user_admin="admin-secret" user_kafkaclient1="kafkaclient1-secret"; 
    ```

### Client

1. Create a [ConfigMap](https://kubernetes.io/docs/concepts/configuration/configmap/) based on the `sasl_client.properties` file:

    ```bash
    kubectl create configmap kafka-client --from-file sasl_client.properties -n kafka
    kubectl describe configmaps -n kafka kafka-client 
    ```

    Output:

    ```bash
    configmap/kafka-client created
    Name:         kafka-client
    Namespace:    kafka
    Labels:       <none>
    Annotations:  <none>

    Data
    ====
    sasl_client.properties:
    ----
    sasl.mechanism=PLAIN
    security.protocol=SASL_PLAINTEXT
    sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required \
      username="kafkaclient1" \
      password="kafkaclient1-secret";

    BinaryData
    ====

    Events:  <none>
    ```

2. Mount the ConfigMap as a `volume`:

    ```yaml
    ...
    volumeMounts:
        - mountPath: /etc/kafka/secrets/
          name: kafka-client
    ...
    volumes:
    - name: kafka-client
      configMap: 
        name: kafka-client
    ```

## Configuring Kafka for ACLs

Apache Kafka® includes a pluggable authorization framework (Authorizer), configured using the `authorizer.class.name` configuration property in the Kafka broker configuration file.

### Broker

1. Enable SASL/PLAIN mechanism for the `CONTROLLER`.

    ```yaml
    - name: KAFKA_LISTENER_NAME_CONTROLLER_PLAIN_SASL_JAAS_CONFIG
      value: org.apache.kafka.common.security.plain.PlainLoginModule required username="admin" password="admin-secret" user_admin="admin-secret" user_kafkaclient1="kafkaclient1-secret"; 
    - name: KAFKA_SASL_MECHANISM_CONTROLLER_PROTOCOL
      value: PLAIN
    - name: KAFKA_CONTROLLER_ENABLED_MECHANISMS
      value: PLAIN
    ```

2. Update the `security.protocol.map` of the `CONTROLLER` to use `SASL_PLAINTEXT`

   ```yaml
   - name: KAFKA_LISTENER_SECURITY_PROTOCOL_MAP
     value: "CONTROLLER:SASL_PLAINTEXT,SASL:SASL_PLAINTEXT"
   ```

3. Configure the `authorizer` and `super.user`

   ```yaml
   - name: KAFKA_SUPER_USERS
    value: User:admin
   - name: KAFKA_AUTHORIZER_CLASS_NAME
     value: org.apache.kafka.metadata.authorizer.StandardAuthorizer
   - name: KAFKA_ALLOW_EVERYONE_IF_NO_ACL_FOUND
     value: "false"
   ```

### Admin Client

1. Create a [ConfigMap](https://kubernetes.io/docs/concepts/configuration/configmap/) based on the `sasl_client.properties` file:

    ```bash
    kubectl create configmap kafka-admin --from-file sasl_admin.properties -n kafka
    kubectl describe configmaps -n kafka kafka-admin 
    ```

    Output:

    ```bash
    configmap/kafka-admin created
    Name:         kafka-admin
    Namespace:    kafka
    Labels:       <none>
    Annotations:  <none>
    
    Data
    ====
    sasl_admin.properties:
    ----
    sasl.mechanism=PLAIN
    security.protocol=SASL_PLAINTEXT
    sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required \
      username="admin" \
      password="admin-secret";
    
    BinaryData
    ====

    Events:  <none>
    ```

2. Mount the ConfigMap as a `volume`:

    ```yaml
    ...
    volumeMounts:
        - mountPath: /etc/kafka/secrets/
          name: kafka-admin
    ...
    volumes:
    - name: kafka-admin
      configMap: 
        name: kafka-admin
    ```

## Confluent Kafka

This [yaml file](01-kafka.yaml) deploys a Kafka cluster within a Kubernetes namespace named `kafka`. It defines various Kubernetes resources required for setting up Kafka in a distributed manner.

Here's a breakdown of what this file does:

### Service Account (kind: ServiceAccount)

A [Service Account](https://kubernetes.io/docs/concepts/security/service-accounts/) named `kafka` is created in the `kafka` namespace. Service accounts are used to control permissions and access to resources within the cluster.

### Headless Service (kind: Service)

A [headless Service](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services) named `kafka-headless` is defined in the `kafka` namespace.

It exposes ports `9092` (for SASL_PLAINTEXT communication).

### StatefulSet (kind: StatefulSet)

A [StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) named `kafka` is configured in the `kafka` namespace with three replicas.

It manages Kafka pods and ensures they have stable hostnames and storage.

Each pod is associated with the headless service `kafka-headless` and the service account `kafka.` The pods use the Confluent Kafka Docker image (version 7.5.0). At the time of writing, this is the latest Confluent release.

### Deploy

You can deploy Kafka using the following commands:

```bash
kubectl apply -f 00-namespace.yaml
kubectl apply -f 01-kafka.yaml
```

Check if the Pods are `Running`:

```bash
kubectl get pods
```

Output:

```bash
NAME      READY   STATUS    RESTARTS   AGE
kafka-0   1/1     Running   0          61s
kafka-1   1/1     Running   0          92s
kafka-2   1/1     Running   0          2m33s
```

### Verify communication across brokers

There should now be three Kafka brokers each running on separate pods within your cluster. Name resolution for the headless service and the three pods within the StatefulSet is automatically configured by Kubernetes as they are created, allowing for communication across brokers. See the related documentation for more details on this feature.

You can check the first pod's logs with the following command:

```bash
kubectl logs kafka-0
```

The name resolution of the three pods can take more time to work than it takes the pods to start, so you may see `UnknownHostException warnings`` in the pod logs initially:

```bash
WARN [RaftManager nodeId=2] Error connecting to node kafka-1.kafka-headless.kafka.svc.cluster.local:29093 (id: 1 rack: null) (org.apache.kafka.clients.NetworkClient) java.net.UnknownHostException: kafka-1.kafka-headless.kafka.svc.cluster.local         ...
```

But eventually each pod will successfully resolve pod hostnames and end with a message stating the broker has been unfenced:

```bash
INFO [Controller 0] Unfenced broker: UnfenceBrokerRecord(id=1, epoch=176) (org.apache.kafka.controller.ClusterControlManager)
```

## Testing Kafkfa ACLs

To test the ACLs, we will deploy two clients: `admin` and `non-admin`. Open two terminal windows and deploy the
[admin client](./03-kafka-admin.yaml) in one and the [non-admin](./02-kafka-client.yaml) in the another one.

Commands:

```bash
kubectl apply -f 02-kafka-client.yaml
kubectl apply -f 03-kafka-admin.yaml
```

Let's use the operation `CREATE` as an example of how to troubleshoot and solve permission errors in Kafka.

First, create a topic from the `admin` client:
  
```bash
kubectl exec -it kafka-admin -- bash -c "kafka-topics --bootstrap-server kafka-0.kafka-headless.kafka.svc.cluster.local:9092 -create --topic kafka-admin --command-config /etc/kafka/secrets/sasl_admin.properties"
```

Output:

```bash
Created topic kafka-admin
```

Second, try to create a topic from the `non-admin` client:
  
```bash
kubectl exec -it kafka-client -- bash -c "kafka-topics --bootstrap-server kafka-0.kafka-headless.kafka.svc.cluster.local:9092 -create --topic kafka-client --command-config /etc/kafka/secrets/sasl_client.properties"
```

Output:

```bash
Error while executing topic command : Authorization failed.
```
  
Our client does not have permission. Looking into the Kafka logs it's possible to see the following entry:

```bash
kafka-0 kafka [2024-03-13 20:10:17,259] INFO Principal = User:kafkaclient1 is Denied operation = CREATE from host = 10.244.0.9 on resource = Topic:LITERAL:kafka-admin for request = CreateTopics with resourceRefCount = 1 based on rule DefaultDeny (kafka.authorizer.logger)
```

:warning: Note that in the log this is an `INFO` entry and not an `ERROR`.

Create the ACL from the `admin` client:

```bash
kubectl exec -it kafka-admin -- bash -c "kafka-acls --bootstrap-server kafka-0.kafka-headless.kafka.svc.cluster.local:9092 --add --allow-principal User:kafkaclient1 --operation Create --allow-host 10.244.0.9 --cluster --command-config /etc/kafka/secrets/sasl_admin.properties"
```

Output:

```bash
Adding ACLs for resource `ResourcePattern(resourceType=CLUSTER, name=kafka-cluster, patternType=LITERAL)`: 
        (principal=User:kafkaclient1, host=10.244.0.9, operation=CREATE, permissionType=ALLOW)
```

Try to create a topic from the `non-admin` client again:
  
```bash
kubectl exec -it kafka-client -- bash -c "kafka-topics --bootstrap-server kafka-0.kafka-headless.kafka.svc.cluster.local:9092 -create --topic kafka-client --command-config /etc/kafka/secrets/sasl_client.properties"
```

Output:

```bash
Created topic kafka-client.
```
