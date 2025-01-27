## Prerequisties:

- AMQ Streams Operator Installed - 2.5.0

To deploy Oracle DB, you could use the free version provided by oracle `container-registry.oracle.com/database/free:23.3.0.0` or AWS:

## Deploy pre-populated Oracle DB instance using Free version:

```
oc apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: oracle-database-deployment
  namespace: default
spec:
  selector:
    matchLabels:
      app: oracle-database
  replicas: 1
  template:
    metadata:
      labels:
        app: oracle-database
    spec:
      containers:
        - name: oracle-database-container
          image: container-registry.oracle.com/database/free:23.3.0.0
          env:
            - name: ORACLE_PWD
              value: secret
          resources:
            limits:
              cpu: 2
              memory: 4Gi
          ports:
            - containerPort: 1521
          readinessProbe:
            tcpSocket:
              port: 1521
            initialDelaySeconds: 15
            periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: oracle-database-service
  namespace: default
spec:
  selector:
    app: oracle-database
  ports:
    - name: oracle-database-port
      protocol: TCP
      port: 1521
      targetPort: 1521
EOF
```

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
    --allocated-storage 100 \
    --engine oracle-se2 \
    --engine-version 19.0.0.0.ru-2023-10.rur-2023-10.r1 \
    --master-username admin \
    --master-user-password mypassword123 \
    --db-name oracledb \
    --publicly-accessible \
    --vpc-security-group-ids sg-xxxxxxx \
    --db-subnet-group-name YOUR_DB_SUBNET_GROUP_NAME \
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

How to get Oracle DB instance status and to get an Endpoint Address:

```
aws rds describe-db-instances --db-instance-identifier my-oracle-instance --query "DBInstances[*].[DBInstanceIdentifier,DBInstanceStatus,Endpoint.Address,PubliclyAccessible]" --output table
```
Once the database instance is fully provisioned and ready to use, the status will change to `available`.
e.g:
```
------------------------------------------------------------------------------------------------------------
|                                            DescribeDBInstances                                           |
+--------------------+------------+---------------------------------------------------------------+--------+
|  my-oracle-instance|  available |  my-oracle-instance.xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx  |  True  |
+--------------------+------------+---------------------------------------------------------------+--------+
```
Replace the endpoint address in the sqlplus below:

Connect to your Oracle DB:

```
sqlplus admin/mypassword123@my-oracle-instance.xxxxxxxxx:1521/oracledb
```

```
curl -O https://raw.githubusercontent.com/aboucham/debezium/main/examples/inventory.sql
sqlplus admin/mypassword123@my-oracle-instance.xxxxxxxxx:1521/oracledb @inventory.sql
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
      - name: debezium-oracle-connector
        artifacts:
          - type: zip
            url: https://maven.repository.redhat.com/ga/io/debezium/debezium-connector-oracle/2.7.3.Final-redhat-00003/debezium-connector-oracle-2.7.3.Final-redhat-00003-plugin.zip
          - type: zip
            url: https://maven.repository.redhat.com/ga/io/debezium/debezium-scripting/2.7.3.Final-redhat-00003/debezium-scripting-2.7.3.Final-redhat-00003.zip
          - type: jar
            url: https://repo1.maven.org/maven2/org/codehaus/groovy/groovy/3.0.11/groovy-3.0.11.jar
          - type: jar
            url: https://repo1.maven.org/maven2/org/codehaus/groovy/groovy-jsr223/3.0.11/groovy-jsr223-3.0.11.jar
          - type: jar
            url: https://repo1.maven.org/maven2/org/codehaus/groovy/groovy-json/3.0.19/groovy-json-3.0.11.jar
          - type: jar
            url: https://repo1.maven.org/maven2/com/oracle/database/jdbc/ojdbc8/21.6.0.0/ojdbc8-21.6.0.0.jar
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
  version: 3.8.0
EOF
```
NOTE: Ensure that jdbc driver has the same version as Oracle DB `ojdbc8-19.21.0.0.jar`

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
  config:
    database.hostname: oracle-database-service.default.svc.cluster.local
    database.port: 1521
    database.dbname: FREEPDB1
    database.user: SYSTEM
    database.password: secret
    topic.prefix: cdc
    topic.creation.default.partitions: 1
    topic.creation.default.replication.factor: 1
    schema.history.internal.kafka.topic: cdc.oracledb.schema.history
    schema.history.internal.kafka.bootstrap.servers: 'my-cluster-kafka-bootstrap:9092'
    table.include.list: 'ORACLEDB.ADMIN.*'
  tasksMax: 1
EOF
```

   - 'storeConfiguration.jdbcConnectionUrl=jdbc:oracle:thin:SYSTEM/secret@oracle-database-service.default.svc.cluster.local:1521/FREEPDB1'

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
aws rds describe-db-instances --db-instance-identifier my-oracle-instance
```
If the DB instance has been successfully deleted, you should receive an error indicating that the specified DB instance doesn't exist. If the DB instance still exists, the command will return details about the DB instance, including its status:
```
An error occurred (DBInstanceNotFound) when calling the DescribeDBInstances operation: DBInstance my-oracle-instance not found.
```
