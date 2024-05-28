# Title: Managing offsets in Kafka Connect
How to keep/preserve topic offset position while recreating a new Kafka Connector ?

Environment Setup:

- AMQ Streams 2.6.0
- Debezium 2.4.0.Final
- MySQL 8.0

Prerequisites:

- Install AMQ Streams Operator from OperatorHub
- Install CamelK Operator from OperatorHub
- Install Service Registry Operator 


# I. Kafka Cluster Creation / Deploying Kafka Connect and Kafka Connector

Create the 'dbz-mysql' namespace: `oc new-project dbz-mysql`:

```
oc new-project dbz-mysql
```

Create Kafka clusters using Kafka CR YAML configuration

```
    oc create -f - <<EOF
    apiVersion: kafka.strimzi.io/v1beta2
    kind: Kafka
    metadata:
      name: my-cluster
    spec:
      kafka:
        config:
          offsets.topic.replication.factor: 1
          transaction.state.log.replication.factor: 1
          transaction.state.log.min.isr: 2
          default.replication.factor: 1
          min.insync.replicas: 1
          inter.broker.protocol.version: '3.6'
        storage:
          type: jbod
          volumes:
            - deleteClaim: false
              id: 0
              size: 1Gi
              type: persistent-claim
        listeners:
          - name: plain
            port: 9092
            type: internal
            tls: false
          - name: tls
            port: 9093
            type: internal
            tls: true
        version: 3.6.0
        replicas: 3
      entityOperator:
        topicOperator: {}
        userOperator: {}
      zookeeper:
        storage:
          deleteClaim: false
          size: 1Gi
          type: persistent-claim
        replicas: 3
    EOF
```

## Build and deploy Debezium connectors using KafkaConnect Custom Resource (CR)

- Install Kafka connect CR with Debezium plugins through AMQ Streams:

```
oc create -f - <<EOF
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaConnect
metadata:
  name: debezium-connect
  annotations:
    strimzi.io/use-connector-resources: "true"
spec:
  bootstrapServers: 'my-cluster-kafka-bootstrap:9093'
  build:
    output:
      image: >-
        image-registry.openshift-image-registry.svc:5000/dbz-mysql/dbz-mysql-connect:latest
      type: docker
    plugins:
      - artifacts:
          - type: tgz
            url: >-
              https://repo1.maven.org/maven2/io/debezium/debezium-connector-mysql/2.4.0.Final/debezium-connector-mysql-2.4.0.Final-plugin.tar.gz
        name: debezium-mysql-connector
  config:
    config.storage.replication.factor: -1
    config.storage.topic: debezium-connect-configs
    group.id: debezium-connect-cluster
    offset.storage.replication.factor: -1
    offset.storage.topic: debezium-connect-offsets
    status.storage.replication.factor: -1
    status.storage.topic: debezium-connect-status
  replicas: 1
  tls:
    trustedCertificates:
      - certificate: ca.crt
        secretName: my-cluster-cluster-ca-cert
  version: 3.6.0
EOF
```

TIP: Enable `use-connector-resources` to instantiate Kafka connectors through specific custom resources:
`oc annotate kafkaconnects2is my-connect-cluster strimzi.io/use-connector-resources=true`

NOTE: Multiple instances attempting to use the same internal topics will cause unexpected errors, so you must change the values of these properties for each instance.


### Check

```
oc get kc debezium-connect -o yaml | yq '.status.connectorPlugins'
```

## Deploy pre-populated MySQL instance

#### Configure credentials for the database:

```
oc new-app \
    -e MYSQL_ROOT_PASSWORD=debezium \
    -e MYSQL_USER=mysqluser \
    -e MYSQL_PASSWORD=mysqlpw \
    -e MYSQL_DATABASE=inventory \
    mysql:8.0-el9
```

```
oc rsh `oc get pods -l deployment=mysql -o name` mysql -u mysqluser -pmysqlpw inventory
```
```
CREATE TABLE products
(
    id INT PRIMARY KEY NOT NULL,
    name VARCHAR(100),
    model VARCHAR(100),
    price INT
);


INSERT INTO products VALUES (1, 'LenovoT41', 'Lenovo T 41', 3);
INSERT INTO products VALUES (2, 'LenovoT41', 'Lenovo UT 41', 45);
INSERT INTO products VALUES (3, 'DELL', 'DELL 41', 45);
```

## Kafka Connector CR: Create KC with MYSQL Connector:

```
oc create -f - <<EOF
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaConnector
metadata:
  labels:
    strimzi.io/cluster: debezium-connect
  name: mysql-connector
spec:
  class: io.debezium.connector.mysql.MySqlConnector
  tasksMax: 1
  autoRestart:
    enabled: true
  config:
    database.hostname: mysql
    database.port: 3306
    database.user: root
    database.password: debezium
    database.server.id: 184057
    database.whitelist: inventory
    database.names: inventory
    include.schema.changes: false
    schema.history.internal.kafka.topic: schemahistory.fullfillment
    schema.history.internal.kafka.bootstrap.servers: 'my-cluster-kafka-bootstrap:9092'
    topic.prefix: mysql
    topic.creation.default.replication.factor: 1
    topic.creation.default.partitions: 1
EOF
```

#### Check Status:

```
oc get kctr
NAME              CLUSTER             CONNECTOR CLASS                              MAX TASKS   READY
mysql-connector   dbz-mysql-connect   io.debezium.connector.mysql.MySqlConnector   1           True
```

```
oc get kctr mysql-connector -o yaml | yq '.status'
```

```
status:
  conditions:
  - lastTransitionTime: "2023-10-24T12:12:59.267139132Z"
    status: "True"
    type: Ready
  connectorStatus:
    connector:
      state: RUNNING
      worker_id: 10.131.0.22:8083
    name: mysql-connector
    tasks:
    - id: 0
      state: RUNNING
      worker_id: 10.131.0.22:8083
    type: source
  observedGeneration: 1
  tasksMax: 1
  topics:
  - mysql.inventory.products
```

# How to keep/preserve topic offset position while recreating a new Kafka Connector ?
# Moving Kafka Connect Connector to a New Cluster

If you are running Kafka Connect 3.6.0 (AMQ Streams 2.6), you can use the `/connectors/{connector}/offsets` REST endpoints to retrieve offsets in the existing Connect cluster and set them in their new Connect clusters.

## Steps:

1. **Stop the Connector in the Original Cluster**
    - Endpoint: `PUT /connectors/{name}/stop`
    - Action: Stop the connector in the original Kafka Connect cluster.
    - Add `spec.state:stopped` to kafkaConnector CR.

- Check with:

```
oc rsh debezium-connect-connect-0 curl localhost:8083/connectors/mysql-connector/status
```
```
{"name":"mysql-connector","connector":{"state":"STOPPED","worker_id":"debezium-connect-connect-0.debezium-connect-connect.dbz-mysql.svc:8083"},"tasks":[],"type":"source"}%
```



2. **Get Offsets**
    - Endpoint: `GET /connectors/{name}/offsets`
    - Action: Retrieve offsets for the connector in the original cluster.

```
oc rsh debezium-connect-connect-0 curl localhost:8083/connectors/mysql-connector/offsets
```
```
{"offsets":[{"partition":{"server":"mysql"},"offset":{"ts_sec":1716802895,"file":"binlog.000002","pos":1424}}]}
```

3. **Delete the Connector**
    - Endpoint: `DELETE /connectors/{name}`
    - Action: Delete the connector from the original Kafka Connect cluster.

```
oc delete kafkaConnector mysql-connector
kafkaconnector.kafka.strimzi.io "mysql-connector" deleted
```

4. **Create the Connector in the New Cluster**
    - Action: Set up the connector in the new Kafka Connect cluster.
5. **Stop the Connector in the New Cluster**
    - Endpoint: `PUT /connectors/{name}/stop`
    - Action: Stop the connector in the new Kafka Connect cluster.
    - Stop it (`state:stopped`)
```
oc create -f - <<EOF
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaConnect
metadata:
  name: debezium-connect-new
  annotations:
    strimzi.io/use-connector-resources: "true"
spec:
  bootstrapServers: 'my-cluster-kafka-bootstrap:9093'
  build:
    output:
      image: >-
        image-registry.openshift-image-registry.svc:5000/dbz-mysql/dbz-mysql-connect-new:latest
      type: docker
    plugins:
      - artifacts:
          - type: tgz
            url: >-
              https://repo1.maven.org/maven2/io/debezium/debezium-connector-mysql/2.4.0.Final/debezium-connector-mysql-2.4.0.Final-plugin.tar.gz
        name: debezium-mysql-connector
  config:
    config.storage.replication.factor: -1
    config.storage.topic: debezium-connect-new-configs
    group.id: debezium-connect-new-cluster
    offset.storage.replication.factor: -1
    offset.storage.topic: debezium-connect-new-offsets
    status.storage.replication.factor: -1
    status.storage.topic: debezium-connect-new-status
  replicas: 1
  tls:
    trustedCertificates:
      - certificate: ca.crt
        secretName: my-cluster-cluster-ca-cert
  version: 3.6.0
EOF
```

```
oc create -f - <<EOF
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaConnector
metadata:
  labels:
    strimzi.io/cluster: debezium-connect-new
  name: mysql-connector-new
spec:
  state: stopped
  class: io.debezium.connector.mysql.MySqlConnector
  tasksMax: 1
  autoRestart:
    enabled: true
  config:
    database.hostname: mysql
    database.port: 3306
    database.user: root
    database.password: debezium
    database.server.id: 184057
    database.whitelist: inventory
    database.names: inventory
    include.schema.changes: false
    schema.history.internal.kafka.topic: schemahistory.fullfillment
    schema.history.internal.kafka.bootstrap.servers: 'my-cluster-kafka-bootstrap:9092'
    topic.prefix: mysql
    topic.creation.default.replication.factor: 1
    topic.creation.default.partitions: 1
EOF
```
Check the status:
```
oc rsh debezium-connect-new-connect-0 curl localhost:8083/connectors/mysql-connector-new/status
{"name":"mysql-connector-new","connector":{"state":"STOPPED","worker_id":"debezium-connect-new-connect-0.debezium-connect-new-connect.dbz-mysql.svc:8083"},"tasks":[],"type":"source"}%
```



6. **Set Offsets**
    - Endpoint: `PATCH /connectors/{name}/offsets`
    - Action: Set the connector's offset in the new Kafka Connect cluster, reusing the output obtained from the original cluster.

```
oc rsh debezium-connect-new-connect-0 curl localhost:8083/connectors/mysql-connector-new/offsets
```
```
{"offsets":[]}%
```

```
oc exec -i debezium-connect-new-connect-0 -- curl -X PATCH \
    -H "Accept:application/json" \
    -H "Content-Type:application/json" \
    http://localhost:8083/connectors/mysql-connector-new/offsets -d @- <<'EOF'
{
   "offsets":[
      {
         "partition":{
            "server":"mysql"
         },
         "offset":{
            "ts_sec":1716802895,
            "file":"binlog.000002",
            "pos":1424
         }
      }
   ]
}
EOF
```

```
{"message":"The Connect framework-managed offsets for this connector have been altered successfully. However, if this connector manages offsets externally, they will need to be manually altered in the system that the connector uses."}%
```

```
oc rsh debezium-connect-new-connect-0 curl localhost:8083/connectors/mysql-connector-new/offsets
```
```
{"offsets":[{"partition":{"server":"mysql"},"offset":{"ts_sec":1716802895,"file":"binlog.000002","pos":1424}}]}%
```


7. **Restart the Connector**
    - Endpoint: `PUT /connectors/{name}/resume`
    - Action: Restart the connector in the new Kafka Connect cluster.

     Change the `state` in kafka Connector CR to `state: running`

