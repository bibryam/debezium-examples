##Debezium Demo

```
oc login --token=*** --server=***

oc new-project dbz-demo
```

###Create database
```

oc apply -f postgresql.yaml 
oc rsh $(oc get pods -o name -l app=postgresql)
psql -U postgresql sampledb
CREATE TABLE CUSTOMER
(
   ID bigint,
   SSN char(25),
   NAME varchar(64),
   CONSTRAINT CUSTOMER_PK PRIMARY KEY(ID)
);
CREATE TABLE ADDRESS
(
   ID bigint,
   STREET char(25),
   ZIP char(10),
   CUSTOMER_ID bigint,
   CONSTRAINT ADDRESS_PK PRIMARY KEY(ID),
   CONSTRAINT CUSTOMER_FK FOREIGN KEY (CUSTOMER_ID) REFERENCES CUSTOMER (ID)
);
INSERT INTO CUSTOMER (ID,SSN,NAME) VALUES (13, 'CST01002','Joseph Smith');
INSERT INTO CUSTOMER (ID,SSN,NAME) VALUES (11, 'CST01003','Nicholas Ferguson');
INSERT INTO CUSTOMER (ID,SSN,NAME) VALUES (12, 'CST01004','Jane Aire');
INSERT INTO ADDRESS (ID, STREET, ZIP, CUSTOMER_ID) VALUES (10, 'Main St', '12345', 10);
select * from CUSTOMER;
\q
exit

```

###Create Kafka cluster

```
oc apply -f operatorgroup.yaml

oc apply -f subscription.yaml 

oc apply -f kafka-ephemeral-single.yaml 

oc apply -f kafka-connect-s2i-single-node-kafka.yaml
```

###Download and build a Debezium connector
 
```
mkdir -p plugins && cd plugins
curl https://repo1.maven.org/maven2/io/debezium/debezium-connector-postgres/1.2.0.Final/debezium-connector-postgres-1.2.0.Final-plugin.tar.gz | tar xz;

curl https://repo1.maven.org/maven2/io/debezium/debezium-connector-mysql/1.2.0.Final/debezium-connector-mysql-1.2.0.Final-plugin.tar.gz | tar xz;

oc start-build my-connect-cluster-connect --from-dir=. --follow

rm -rf plugins

oc exec -i my-cluster-kafka-0 -c kafka -- curl -i -X PUT -H "Accept:application/json" -H "Content-Type:application/json" http://my-connect-cluster-connect-api:8083/connectors/postgresql-connector/config/ -d @- <<'EOF'
{
"name":"postgresql-connector",
"connector.class":"io.debezium.connector.postgresql.PostgresConnector",
"tasks.max":"1",
"database.hostname":"postgresql",
"database.port":"5432",
"database.user":"postgresql",
"database.password":"postgresql",
"database.server.name":"dbserver1",
"database.dbname":"sampledb",
"plugin.name":"pgoutput"
}
}
EOF
```


###Random queries
 
```
oc exec -i my-cluster-kafka-0 -c kafka -- curl -X GET -H "Accept:application/json" -H "Content-Type:application/json" http://my-connect-cluster-connect-api:8083/connector-plugins

oc exec -i my-cluster-kafka-0 -c kafka -- curl -X GET -H "Accept:application/json" -H "Content-Type:application/json" http://my-connect-cluster-connect-api:8083/connectors

oc exec -i my-cluster-kafka-0 -c kafka -- curl -X GET -H "Accept:application/json" -H "Content-Type:application/json" \
http://my-connect-cluster-connect-api:8083/connectors/postgresql-connector/status

oc exec -i my-cluster-kafka-0 -c kafka -- curl -X GET -H "Accept:application/json" -H "Content-Type:application/json"  http://my-connect-cluster-connect-api:8083/connectors/postgresql-connector/config

oc exec -i my-cluster-kafka-0 -c kafka -- curl -X POST -H "Accept:application/json" -H "Content-Type:application/json" http://my-connect-cluster-connect-api:8083/connectors/postgresql-connector/restart


oc exec -i my-cluster-kafka-0 -c kafka -- curl -X PUT -H "Accept:application/json" -H "Content-Type:application/json" http://my-connect-cluster-connect-api:8083/connectors/postgresql-connector/pause

oc exec -i my-cluster-kafka-0 -c kafka -- curl -X PUT -H "Accept:application/json" -H "Content-Type:application/json" http://my-connect-cluster-connect-api:8083/connectors/postgresql-connector/resume

oc exec -i my-cluster-kafka-0 -c kafka -- curl -X PUT -H "Accept:application/json" -H "Content-Type:application/json" http://my-connect-cluster-connect-api:8083/connectors/postgresql-connector/resume

oc exec -i my-cluster-kafka-0 -c kafka -- curl -X DELETE -H "Accept:application/json" -H "Content-Type:application/json" \
http://my-connect-cluster-connect-api:8083/connectors/postgresql-connector

```

###CR based queries

```
oc annotate kafkaconnects2is my-connect-cluster strimzi.io/use-connector-resources=true

oc apply -f - << EOF
apiVersion: kafka.strimzi.io/v1alpha1
kind: KafkaConnector
metadata:
  name: postgresql-connector
  labels:
    strimzi.io/cluster: my-connect-cluster
spec:
  class: io.debezium.connector.postgresql.PostgresConnector
  tasksMax: 1
  config:
    database.hostname: postgresql
    database.port: 5432
    database.user: postgresql
    database.password: postgresql
    database.server.name: dbserver1
    database.dbname: sampledb
    plugin.name: pgoutput
EOF
```


###Viewing a create event
```
oc exec -it my-cluster-kafka-0 -- /opt/kafka/bin/kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --from-beginning \
  --property print.key=true \
  --topic dbserver1.public.customer
```  

###Change data and observe events
```  
oc rsh $(oc get pods -o name -l app=postgresql)
psql -U postgresql sampledb

SELECT * FROM CUSTOMER;

UPDATE CUSTOMER SET NAME='Anne Marie' WHERE id=13;
DELETE FROM CUSTOMER WHERE id=13;
INSERT INTO CUSTOMER (ID,SSN,NAME) VALUES (15, 'CST01005','Alexander Bell');
```  
