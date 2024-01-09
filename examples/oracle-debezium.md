## Prerequisties:

- AMQ Streams Operator Installed - 2.5.0

## Deploy pre-populated Oracle DB instance using AWS:

List all VPC Security group ID:

```
aws ec2 describe-security-groups --query "SecurityGroups[*].GroupId" --output text
```

List all the `subnets` on specific `VPC` id:
```
aws ec2 describe-subnets --filters Name=vpc-id,Values=vpc-xxxxx --query "Subnets[*].[SubnetId,CidrBlock,AvailabilityZone]" --output table
```

Choose two `subnets` ids in two different `AvailabilityZone`:

```
aws rds create-db-subnet-group \
    --db-subnet-group-name YOUR_DB_SUBNET_GROUP_NAME \
    --db-subnet-group-description "Description for your DB subnet group" \
    --subnet-ids subnet-xxxxxxxx subnet-xxxxxxxxxx
```

Create Oracle DB instance:

```
aws rds create-db-instance \
    --db-instance-identifier my-oracle-instance \
    --db-instance-class db.m5.large \
    --db-subnet-group-name YOUR_DB_SUBNET_GROUP_NAME \
    --engine oracle-se2 \
    --engine-version 19.0.0.0.ru-2020-10.rur-2020-10.r1 \
    --master-username admin \
    --master-user-password mypassword123 \
    --allocated-storage 100 \
    --db-name oracledb \
    --vpc-security-group-ids sg-xxxxxxx \
    --backup-retention-period 7 \
    --no-multi-az \
    --license-model license-included
```

Connect to your Oracle DB:

```
sqlplus admin/mypassword123@my-oracle-instance.xxxxxxxxx:1521/oracledb
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

## kafka Connect CR - Build - Oracle Instances:
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
        name: debezium-oracle-connector
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
oc get kc debezium-connect -o yaml | yq '.status.connectorPlugins'
```

## kafka Connector CR:
### Create KC with Oracle Connector:

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
oc get kctr mysql-connector -o yaml | yq '.status'

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
