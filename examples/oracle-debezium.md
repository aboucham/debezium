## Prerequisties:

- AMQ Streams Operator Installed - 2.5.0

## Deploy pre-populated Oracle DB instance using AWS:

List all VPC Security group ID:

```
aws ec2 describe-security-groups --query "SecurityGroups[*].GroupId" --output text
```

List all the `subnets` on specific `VPC` id:

```
aws ec2 describe-subnets --query "Subnets[*].[SubnetId, CidrBlock, AvailabilityZone]" --output table
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

Make the Oracle instance Public and reachable:

To make the endpoint of your RDS DB instance public and accessible from the internet, follow these steps:

- Modify the Security Group:

You need to modify the security group `--vpc-security-group-ids sg-xxxxxxx` associated with your RDS DB instance to allow inbound traffic on the database port from your IP address or a range of IP addresses.

```
aws ec2 authorize-security-group-ingress \
    --group-id sg-xxxxxxx \
    --protocol tcp \
    --port 1521 \
    --cidr 0.0.0.0/0
```

 Replace YOUR_SECURITY_GROUP_ID with the ID of the security group associated with your RDS instance `--vpc-security-group-ids sg-xxxxxxx`, PORT_NUMBER with the appropriate database port, and YOUR_IP_ADDRESS with your public IP address.

- Modify the DB Instance Public Accessibility: By default, the DB instance is set to not be publicly accessible. You can modify this setting to enable public accessibility:
  
```
aws rds modify-db-instance \
    --db-instance-identifier my-oracle-instance \
    --publicly-accessible
```

NOTE: After making these changes, it might take a few minutes for the changes to propagate and for the endpoint to become publicly accessible.

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

Create namespace `dbz-oracle`:

```
oc new-project dbz-oracle
```

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
        image-registry.openshift-image-registry.svc:5000/dbz-oracle/dbz-oracle-connect:latest
      type: docker
    plugins:
      - artifacts:
          - type: tgz
            url: >-
              https://repo1.maven.org/maven2/io/debezium/debezium-connector-oracle/2.4.1.Final/debezium-connector-oracle-2.4.1.Final-plugin.tar.gz
        name: debezium-oracle-connector
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
  version: 3.5.0
EOF
```

## Check

```
oc get kc debezium-connect -o yaml | yq '.status.connectorPlugins'
```

## kafka Connector CR:
### Create KC with Oracle Connector:

1. Create secret for database user and password:

```
oc create secret generic debezium-secret-oracledb --from-literal=username=admin --from-literal=password=mypassword123 -n dbz-oracle
```

2. Create `KafkaConnector` named `oracle-connector`:

```
oc create -f - <<EOF
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaConnector
metadata:
  labels:
    strimzi.io/cluster: debezium-connect
  name: oracle-connector
spec:
  class: io.debezium.connector.oracle.OracleConnector
  tasksMax: 1
  autoRestart:
    enabled: true
  config:
    database.hostname: my-oracle-instance.xxxxxxxxx
    database.port: 1521
    database.dbname: oracledb
    database.user: ${secrets:dbz-oracle/debezium-secret-oracledb:username}
    database.password: ${secrets:dbz-oracle/debezium-secret-oracledb:password}
    topic.prefix: cdc
    topic.creation.default.replication.factor: 1
    topic.creation.default.partitions: 1
    table.include.list: "ORACLEDB.PRODUCTS"
    schema.history.internal.kafka.bootstrap.servers: my-cluster-kafka-bootstrap:9092
    schema.history.internal.kafka.topic: cdc.oracledb.schema.history
    schema.history.internal.store.only.captured.tables.ddl: true
    schema.history.internal.store.only.captured.databases.ddl: true
    poll.interval.ms: 100
    max.batch.size: 8192
    max.queue.size: 32768
EOF
```

## Check Status:

```
$ oc get kctr
NAME              CLUSTER             CONNECTOR CLASS                              MAX TASKS   READY
oracle-connector   debezium-connect   io.debezium.connector.oracle.OracleConnector   1           True
```
```
oc get kctr oracle-connector -o yaml | yq '.status'

status:
  conditions:
  - lastTransitionTime: "2023-10-24T12:12:59.267139132Z"
    status: "True"
    type: Ready
  connectorStatus:
    connector:
      state: RUNNING
      worker_id: 10.131.0.22:8083
    name: oracle-connector
    tasks:
    - id: 0
      state: RUNNING
      worker_id: 10.131.0.22:8083
    type: source
  observedGeneration: 1
  tasksMax: 1
  topics:
  - cdc.inventory.products
```

Delete Oracle instance in AWS via the command line typically involves interacting with the AWS CLI (Command Line Interface) and RDS-specific commands.:

```
aws rds delete-db-instance --db-instance-identifier my-oracle-instance --skip-final-snapshot
```
