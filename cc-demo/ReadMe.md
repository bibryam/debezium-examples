## Produce/Consume Demo 

### 1. Follow step 4 Start a database from openshift-demo folder

### 2. Expose database using LoadBalancer service
```
oc apply -f nodeport.yaml
```

### 3. Find the full path to the LoadBalancer 
```
oc get svc postgresql-expose -o=jsonpath='{..ingress[0].hostname}'
```

### 4. Start Debezium connector
```
config.json 
```

### 4. Start a Kafka client for change events
Run the following commands from openshift-demo
```
oc apply -f operatorgroup.yaml

oc apply -f subscription.yaml 

oc apply -f kafka-cluster.yaml 
```
Once Kafka broker is up, use its container and the client library in it

```  
oc cp config.properties my-cluster-kafka-0:/tmp
oc rsh my-cluster-kafka-0 cat /tmp/config.properties

oc exec -it my-cluster-kafka-0 -c kafka -- /opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server pkc-4yyd6.us-east1.gcp.confluent.cloud:9092 \
  --consumer.config /tmp/config.properties \
  --topic dbserver1.public.customer \
  --from-beginning \
  --max-messages 100 \
  --property print.key=true
 
```  


