# rtp-demo-spring-boot



## Manual Setup

#### OC and Minishift

Follow the Minishift install instructions according to your environment:

https://docs.okd.io/latest/minishift/getting-started/preparing-to-install.html

Install the most recent `oc` binary:

https://github.com/openshift/origin/releases

Ensure that Minishift and `oc` versions are aligned:
```
$ oc version
oc v3.11.0+0cbc58b
kubernetes v1.11.0+d4cacc0
features: Basic-Auth

Server https://192.168.64.6:8443
kubernetes v1.11.0+d4cacc0
```

#### Start the Minishift VM and Enable Admin User

Start minishift with enough resources:
```
$ minishift start --cpus 4 --disk-size 100GB --memory 12GB
```

Once the Kubernetes cluster is running, login as admin user:
```
$ oc login -u system:admin
```

Enable the admin user so that you can login to the console as u:admin, p:admin
```
$ minishift addon apply admin-user
```

You should be able to login to the web console with user:admin, pass:admin



#### Strimzi Cluster and Kafka Topics

Create a new project for the Strimzi cluster:
```
$ oc new-project kafka-cluster
```

Apply the Strimzi installation file:
```
$ oc apply -f kafka/strimzi-cluster-operator-0.8.2.yaml -n kafka-cluster
```

Provision an ephemeral Kafka cluster:
```
$ oc apply -f kafka/kafka-ephemeral.yaml -n kafka-cluster
```

Note: The above files are local versions of these Strimzi project examples, modified to use the project name 'kafka-cluster':
- https://github.com/strimzi/strimzi-kafka-operator/releases/download/0.8.2/strimzi-cluster-operator-0.8.2.yaml
- https://github.com/strimzi/strimzi-kafka-operator/blob/master/examples/kafka/kafka-ephemeral.yaml


Watch the deployment until all Kafka pods are created and running:
```
$ oc get pods -n kafka-cluster -w
NAME                                          READY     STATUS    RESTARTS   AGE
my-cluster-entity-operator-5d7cd7774c-x8sg7   3/3       Running   0          33s
my-cluster-kafka-0                            2/2       Running   0          57s
my-cluster-kafka-1                            2/2       Running   0          57s
my-cluster-kafka-2                            2/2       Running   0          57s
my-cluster-zookeeper-0                        2/2       Running   0          1m
my-cluster-zookeeper-1                        2/2       Running   0          1m
my-cluster-zookeeper-2                        2/2       Running   0          1m
strimzi-cluster-operator-56d699b5c5-ch9r2     1/1       Running   0          2m
```

Create the topics for the rtp demo application:
```
$ oc apply -f kafka/creditor-ctms.yaml -n kafka-cluster
$ oc apply -f kafka/creditor-acks.yaml -n kafka-cluster
$ oc apply -f kafka/creditor-pre-validation.yaml -n kafka-cluster
$ oc apply -f kafka/creditor-post-validation.yaml -n kafka-cluster
```

Confirm on each Kafka broker that the topics were replicated.
```
$ oc exec -it my-cluster-kafka-0 -c kafka -- bin/kafka-topics.sh --zookeeper localhost:2181 --list
creditor-acks
creditor-ctms
creditor-post-validation
creditor-pre-validation
```
```
$ oc exec -it my-cluster-kafka-1 -c kafka -- bin/kafka-topics.sh --zookeeper localhost:2181 --list
creditor-acks
creditor-ctms
creditor-post-validation
creditor-pre-validation
```
```
$ oc exec -it my-cluster-kafka-2 -c kafka -- bin/kafka-topics.sh --zookeeper localhost:2181 --list
creditor-acks
creditor-ctms
creditor-post-validation
creditor-pre-validation
```


```
$ mvn fabric8:deploy -Popenshift
```




oc create configmap rtp-creditor-intake-config \
            --from-literal=BOOTSTRAP_SERVERS="${bootstrap}" \
            --from-literal=CONSUMER_TOPIC=creditor-ctms \
            --from-literal=GROUP_ID=rtp-creditor-intake-app \
            --from-literal=SECURITY_PROTOCOL=PLAINTEXT \
            --from-literal=AUTO_OFFSET_RESET=earliest \
            --from-literal=ENABLE_AUTO_COMMIT=true


oc create configmap rtp-creditor-validation-config \
            --from-literal=BOOTSTRAP_SERVERS="${bootstrap}" \
            --from-literal=CONSUMER_TOPIC=creditor-pre-validation \
            --from-literal=GROUP_ID=rtp-creditor-validation-app \
            --from-literal=SECURITY_PROTOCOL=PLAINTEXT \
            --from-literal=AUTO_OFFSET_RESET=earliest \
            --from-literal=ENABLE_AUTO_COMMIT=true

