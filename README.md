# PioneerEventStreamingDemo
- First of all, install Minikube and kubectl for working on Kubernetes. (Setting up the cluster itself is NOT in scope)
- Start docker container;
```
minikube start --memory=4096
```
- Can learn running pods with;
```
kubectl get po -A
```
## Installing Strimzi to Windows
- Download install files and unzip
- Create a new "kafka" namespace for the Strimzi Kafka Cluster Operator;
```
kubectl create namespace kafka
```
- Change the namespace fields of .yaml files to "kafka" manually
- Create a new "my-kafka-projects" namespace where will deploy kafka cluster;
```
kubectl create namespace my-kafka-projects
```
- Edit the "install/cluster-operator/060-Deployment-strimzi-cluster-operator.yaml" file and set the STRIMZI_NAMESPACE environment variable to the namespace "my-kafka-projects"
- Deploy the CRDs and role-based access control (RBAC) resources to manage the CRDs;
```
kubectl apply -f install/cluster-operator/ -n kafka
```
- Give permission to the Cluster Operator to watch the my-kafka-projects namespace. The commands create role bindings that grant permission for the Cluster Operator to access the Kafka cluster. 
```
kubectl apply -f install/cluster-operator/020-RoleBinding-strimzi-cluster-operator.yaml -n my-kafka-projects
kubectl apply -f install/cluster-operator/032-RoleBinding-strimzi-cluster-operator-topic-operator-delegation.yaml -n my-kafka-projects
kubectl apply -f install/cluster-operator/031-RoleBinding-strimzi-cluster-operator-entity-operator-delegation.yaml -n my-kafka-projects
```
- Create Kafka cluster;
```
kubectl apply -f examples/kafka/kafka-persistent-single.yaml -n my-kafka-projects
```
- Deploy;
```
kubectl wait kafka/my-cluster --for=condition=Ready --timeout=300s -n my-kafka-projects
```
- Pods can be seen with;
```
kubectl get po -A
```
- After deployment screen should be like this;
```
NAMESPACE           NAME                                         READY   STATUS    RESTARTS   AGE
kafka               strimzi-cluster-operator-594455675b-6lmhf    1/1     Running   0          15m
kube-system         coredns-74ff55c5b-p7dxr                      1/1     Running   0          17m
kube-system         etcd-minikube                                1/1     Running   0          18m
kube-system         kube-apiserver-minikube                      1/1     Running   0          18m
kube-system         kube-controller-manager-minikube             1/1     Running   0          18m
kube-system         kube-proxy-ktjx4                             1/1     Running   0          17m
kube-system         kube-scheduler-minikube                      1/1     Running   0          18m
kube-system         storage-provisioner                          1/1     Running   3          18m
my-kafka-projects   my-cluster-entity-operator-6f9c6fc89-xtmlf   3/3     Running   4          4m10s
my-kafka-projects   my-cluster-kafka-0                           1/1     Running   0          5m56s
my-kafka-projects   my-cluster-zookeeper-0                       1/1     Running   0          10m
```
## Kafka Connect Integration
- Below command installs connect pod to cluster;
```
kubectl apply -f examples/connect/kafka-connect-single-node-kafka.yaml -n my-kafka-projects
```
- Deleting can be necessary when "Error" or "CrashLoopBackOff" status.
```
kubectl -n my-kafka-projects delete pod <name>
```
- Can verify successful deployment with;
```
kubectl -n my-kafka-projects get deployments
```
- After deployment screen should be like this;
```
NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
my-cluster-entity-operator   1/1     1            1           23h
my-connect-cluster-connect   1/1     1            1           23h
```
- And running Kafka Connect can seen from pods;
```
NAMESPACE           NAME                                         READY   STATUS    RESTARTS   AGE
kafka               strimzi-cluster-operator-594455675b-6lmhf    1/1     Running   8          24h
kube-system         coredns-74ff55c5b-p7dxr                      1/1     Running   3          24h
kube-system         etcd-minikube                                1/1     Running   3          24h
kube-system         kube-apiserver-minikube                      1/1     Running   4          24h
kube-system         kube-controller-manager-minikube             1/1     Running   5          24h
kube-system         kube-proxy-ktjx4                             1/1     Running   2          24h
kube-system         kube-scheduler-minikube                      1/1     Running   2          24h
kube-system         storage-provisioner                          1/1     Running   24         24h
my-kafka-projects   my-cluster-entity-operator-6f9c6fc89-xtmlf   3/3     Running   44         23h
my-kafka-projects   my-cluster-kafka-0                           1/1     Running   5          6h9m
my-kafka-projects   my-cluster-zookeeper-0                       1/1     Running   14         24h
my-kafka-projects   my-connect-cluster-connect-dfd948dcb-6t7kz   1/1     Running   0          19m
```
## ksqlDB Integration
- Can follow https://docs.ksqldb.io/en/latest/operate-and-deploy/installation/installing/#start-the-ksqldb-server;
- Clone the ksqlDB repo;
```
git clone https://github.com/confluentinc/ksql.git
```
- Get in the cloned directory with "cd"
- Switch the correct branch;
```
git checkout 6.0.1-post
```
- Start ksqlDB stack with;
```
docker-compose up -d
```
- Status check and see all states are "Up";
```
docker-compose ps
```
- Screen should seen as below;
```
Name                      Command            State                 Ports
-------------------------------------------------------------------------------------------------
additional-ksqldb-server   /usr/bin/docker/run         Up      0.0.0.0:49157->8090/tcp
ksql_kafka_1               /etc/confluent/docker/run   Up      0.0.0.0:29092->29092/tcp, 9092/tcp
ksql_schema-registry_1     /etc/confluent/docker/run   Up      8081/tcp
ksql_zookeeper_1           /etc/confluent/docker/run   Up      2181/tcp, 2888/tcp, 3888/tcp
ksqldb-cli                 /bin/sh                     Up
primary-ksqldb-server      /usr/bin/docker/run         Up      0.0.0.0:8088->8088/tcp
```
- Start ksqlDB's interactive CLI with;
```
docker exec -it ksqldb-cli ksql http://primary-ksqldb-server:8088
```
## Kafka Producer & Consumer Setup
- Once the cluster, Kafka Connect and ksqlDB is running, can deploy a Kafka producer to send messages to a Kafka topic (the topic automatically created)
```
kubectl -n my-kafka-projects run kafka-producer -ti --image=strimzi/kafka:0.20.1-kafka-2.6.0 --rm=true --restart=Never -- bin/kafka-console-producer.sh --broker-list my-cluster-kafka-bootstrap:9092 --topic my-topic
```
- Command prompt appears and screen should be like this. Producer text can written here;
```
If you don't see a command prompt, try pressing enter.
>Hello, world!
```
- Deployment of a Kafka consumer that listening producer;
```
kubectl -n my-kafka-projects run kafka-consumer -ti --image=strimzi/kafka:0.20.1-kafka-2.6.0 --rm=true --restart=Never -- bin/kafka-console-consumer.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --topic my-topic --from-beginning
```
