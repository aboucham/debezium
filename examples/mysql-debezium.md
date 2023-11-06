## Prerequisties:

- AMQ Streams Operator Installed - 2.5.0

## Deploy pre-populated MySQL instance
## Configure credentials for the database

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

## kafka Connect CR - Build - SQL Instances:
## Kafka connect CR ######

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
              https://repo1.maven.org/maven2/io/debezium/debezium-connector-mongodb/2.4.0.Final/debezium-connector-mongodb-2.4.0.Final-plugin.tar.gz
        name: debezium-mongodb-connector
      - artifacts:
          - type: tgz
            url: >-
              https://repo1.maven.org/maven2/io/debezium/debezium-connector-postgres/2.4.0.Final/debezium-connector-postgres-2.4.0.Final-plugin.tar.gz
        name: debezium-postgres-connector
      - artifacts:
          - type: tgz
            url: >-
              https://repo1.maven.org/maven2/io/debezium/debezium-connector-mysql/2.4.0.Final/debezium-connector-mysql-2.4.0.Final-plugin.tar.gz
        name: debezium-mysql-connector
  config:
    config.storage.replication.factor: -1
    config.storage.topic: dbz-mysql-connect-configs
    group.id: connect-cluster
    offset.storage.replication.factor: -1
    offset.storage.topic: dbz-mysql-connect-offsets
    status.storage.replication.factor: -1
    status.storage.topic: dbz-mysql-connect-status
  replicas: 1
  tls:
    trustedCertificates:
      - certificate: ca.crt
        secretName: my-cluster-cluster-ca-cert
  version: 3.5.0
EOF
```

## Check

```
oc get kc dbz-mysql-connect -o yaml | yq e 'status.connectorPlugins'
```

## kafka Connector CR:
### Create KC with MYSQL Connector:

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

## Check Status:

```
$ oc get kctr
NAME              CLUSTER             CONNECTOR CLASS                              MAX TASKS   READY
mysql-connector   dbz-mysql-connect   io.debezium.connector.mysql.MySqlConnector   1           True
```
```
oc get kctr mysql-connector -o yaml | yq read - 'status'

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
