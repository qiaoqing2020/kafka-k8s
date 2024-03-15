# Kafka ACLs

- [Kafka ACLs](#kafka-acls)
  - [Descriçāo](#descriçāo)
  - [Ambiente](#ambiente)
  - [Namespace](#namespace)
  - [Configurando o Kafka para ACLs](#configurando-o-kafka-para-acls)
    - [Broker](#broker)
    - [Client Admin](#client-admin)
  - [Confluent Kafka](#confluent-kafka)
    - [Service Account (kind: ServiceAccount)](#service-account-kind-serviceaccount)
    - [Headless Service (kind: Service)](#headless-service-kind-service)
    - [StatefulSet (kind: StatefulSet)](#statefulset-kind-statefulset)
    - [Implantaçāo](#implantaçāo)
    - [Verifique a comunicação entre os brokers](#verifique-a-comunicação-entre-os-brokers)
  - [Testando as ACLs no Kafkfa](#testando-as-acls-no-kafkfa)

## Descriçāo

Implantando e executando a Versão Comunitária do Kafka empacotada com o download da Comunidade Confluent e configurada para usar [autenticaçāo SASL/PLAIN](https://docs.confluent.io/platform/current/kafka/authentication_sasl/authentication_sasl_plain.html) e [ACLS](https://docs.confluent.io/platform/current/kafka/authorization.html).

## Ambiente

| Technology | Version |
| --- | --- |
| Minikube | v1.29.0 |
| Docker | v23.0.5 |
| Kubernetes | v1.26.1 |
| [cp-kafka](https://hub.docker.com/r/confluentinc/cp-kafka) | 7.5.0 |

## Namespace

Este [arquivo yaml](./00-namespace.yaml) define um [namespace](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) para executar o Kafka em um cluster Kubernetes.
Ele isola os recursos do Kafka dentro de um _namespace_ dedicado para uma melhor organização e gerenciamento.

## Configurando o Kafka para ACLs

Apache Kafka® inclui um _framework_ de autorização (Authorizer) configurado usando a propriedade `authorizer.class.name` no arquivo de configuração do Kafka broker.

### Broker

1. Habilite o mecanismos SASL/PLAIN no arquivo para o `CONTROLLER`.

    ```yaml
    - name: KAFKA_LISTENER_NAME_CONTROLLER_PLAIN_SASL_JAAS_CONFIG
      value: org.apache.kafka.common.security.plain.PlainLoginModule required username="admin" password="admin-secret" user_admin="admin-secret" user_kafkaclient1="kafkaclient1-secret"; 
    - name: KAFKA_SASL_MECHANISM_CONTROLLER_PROTOCOL
      value: PLAIN
    - name: KAFKA_CONTROLLER_ENABLED_MECHANISMS
      value: PLAIN
    ```

2. Atualize o valor da entrada `security.protocol.map` para `SASL_PLAINTEXT` no _listener_ do `CONTROLLER`:

   ```yaml
   - name: KAFKA_LISTENER_SECURITY_PROTOCOL_MAP
     value: "CONTROLLER:SASL_PLAINTEXT,SASL:SASL_PLAINTEXT"
   ```

3. Configure o `authorizer` e `super.user`

   ```yaml
   - name: KAFKA_SUPER_USERS
    value: User:admin
   - name: KAFKA_AUTHORIZER_CLASS_NAME
     value: org.apache.kafka.metadata.authorizer.StandardAuthorizer
   - name: KAFKA_ALLOW_EVERYONE_IF_NO_ACL_FOUND
     value: "false"
   ```

### Client Admin

1. Crie um [ConfigMap](https://kubernetes.io/docs/concepts/configuration/configmap/) baseado no arquivo `sasl_client.properties`:

    ```bash
    kubectl create configmap kafka-admin --from-file sasl_admin.properties -n kafka
    kubectl describe configmaps -n kafka kafka-admin 
    ```

    Saída:

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

2. Monte o ConfigMap como um `volume`:

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

Este [arquivo yaml](01-kafka-local.yaml) implanta um cluster Kafka dentro de um _namespace_ chamado `kafka`. Ele define vários recursos do Kubernetes necessários para configurar o Kafka de maneira distribuída.

Aqui está uma explicação do que este arquivo faz:

### Service Account (kind: ServiceAccount)

Uma [Service Account](https://kubernetes.io/docs/concepts/security/service-accounts/) chamada `kafka` é criada no _namespace_ `kafka`. Contas de serviço (Service Accounts) são usadas para controlar permissões e acesso a recursos dentro do cluster.

### Headless Service (kind: Service)

Um [headless Service](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services) chamado `kafka-headless` é criado no _namespace_ `kafka`.

Ele expõe as portas `9092` (para comunicação SASL_PLAINTEXT) e `9093`.

### StatefulSet (kind: StatefulSet)

Um [StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) hamado `kafka-headless` é criado no _namespace_ `kafka` com três réplicas.

Ele gerencia os pods do Kafka e garante que eles tenham nomes de host e armazenamento estáveis.

Cada pod está associado ao serviço `kafka-headless` e à conta de serviço `kafka`. Os pods usam a imagem Docker do Confluent Kafka (versão 7.5.0). No momento da escrita, esta é a versão mais recente da Confluent.

### Implantaçāo

Implemente o Kafka usando os seguintes comandos:

```bash
kubectl apply -f 00-namespace.yaml
kubectl apply -f 01-kafka.yaml
```

### Verifique a comunicação entre os brokers

Agora deve haver três nós (brokers) Kafka, cada um em execução em pods separados dentro do seu cluster. A resolução de nomes para o headless Service e os três pods dentro do StatefulSet é configurada automaticamente pelo Kubernetes conforme são criados, permitindo a comunicação entre os brokers. Consulte a documentação relacionada para obter mais detalhes sobre esse recurso.

Você pode verificar os logs do primeiro pod com o seguinte comando:

```bash
kubectl logs kafka-0
```

A resoluçāo de nomes para os três pods pode demorar mais tempo do que o pod a iniciar, entāo, você pode ver erros `UnknownHostException`` nos logs durante a inicializaçāo:

```bash
WARN [RaftManager nodeId=2] Error connecting to node kafka-1.kafka-headless.kafka.svc.cluster.local:29093 (id: 1 rack: null) (org.apache.kafka.clients.NetworkClient) java.net.UnknownHostException: kafka-1.kafka-headless.kafka.svc.cluster.local         ... 
```

Eventualmente, cada pod irá resolver os nomes e iniciar com uma mensagem afirmando que o broker foi `unfenced`:

```bash
INFO [Controller 0] Unfenced broker: UnfenceBrokerRecord(id=1, epoch=176) (org.apache.kafka.controller.ClusterControlManager)
```

## Testando as ACLs no Kafkfa

Para testar as ACLs, vamos utilizar dois clients: `admin` e `non-admin`. Abra dois terminais e faça o deploy do
[admin client](./03-kafka-admin.yaml) em um e do [non-admin](./02-kafka-client.yaml) no outro.

Comandos:

```bash
kubectl apply -f 02-kafka-client.yaml
kubectl apply -f 03-kafka-admin.yaml
```

Vamos usar a operação `CREATE` como um exemplo para fazer a investigação e resolver erros de permissão no Kafka.

Primeiro, criamos um tópico a partir do nosso cliente `admin`:
  
```bash
kubectl exec -it kafka-admin -- bash -c "kafka-topics --bootstrap-server kafka-0.kafka-headless.kafka.svc.cluster.local:9092 -create --topic kafka-admin --command-config /etc/kafka/secrets/sasl_admin.properties"
```

Saída:

```bash
Created topic kafka-admin
```

Em seguida, tentamos criar um tópico a partir do cliente `non-admin`:
  
```bash
kubectl exec -it kafka-client -- bash -c "kafka-topics --bootstrap-server kafka-0.kafka-headless.kafka.svc.cluster.local:9092 -create --topic kafka-client --command-config /etc/kafka/secrets/sasl_client.properties"
```

Saída:

```bash
Error while executing topic command : Authorization failed.
```
  
Este cliente não tem permissão para criar tópicos no Kafka. Investigando os logs do Kafka vemos a seguinte entrada:

```bash
kafka-0 kafka [2024-03-13 20:10:17,259] INFO Principal = User:kafkaclient1 is Denied operation = CREATE from host = 10.244.0.9 on resource = Topic:LITERAL:kafka-admin for request = CreateTopics with resourceRefCount = 1 based on rule DefaultDeny (kafka.authorizer.logger)
```

:warning: Veja que esta entrada é um `INFO` e não um `ERROR`.

Crie a seguinte ACL a partir do `admin`:

```bash
kubectl exec -it kafka-admin -- bash -c "kafka-acls --bootstrap-server kafka-0.kafka-headless.kafka.svc.cluster.local:9092 --add --allow-principal User:kafkaclient1 --operation Create --allow-host 10.244.0.9 --cluster --command-config /etc/kafka/secrets/sasl_admin.properties"
```

Saída:

```bash
Adding ACLs for resource `ResourcePattern(resourceType=CLUSTER, name=kafka-cluster, patternType=LITERAL)`: 
        (principal=User:kafkaclient1, host=10.244.0.9, operation=CREATE, permissionType=ALLOW)
```

Tentamos criar um tópico a partir do cliente `non-admin` novamente:

```bash
kubectl exec -it kafka-client -- bash -c "kafka-topics --bootstrap-server kafka-0.kafka-headless.kafka.svc.cluster.local:9092 -create --topic kafka-client --command-config /etc/kafka/secrets/sasl_client.properties"
```

Saída:

```bash
Created topic kafka-client.
```
