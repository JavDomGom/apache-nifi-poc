# apache-nifi-poc
Apache NiFi PoC

## Run Docker Compose

```bash
~$ docker-compose up
Creating network "apache-nifi-poc_internal" with driver "bridge"
Creating volume  "apache-nifi-poc_nifi_data" with local driver
Creating volume  "apache-nifi-poc_nifi_registry_data" with local driver
Creating volume  "apache-nifi-poc_kafka_0_data" with local driver
Creating volume  "apache-nifi-poc_kafka_1_data" with local driver
Creating volume  "apache-nifi-poc_kafka_2_data" with local driver
Creating nifi          ... done
Creating nifi-registry ... done
Creating kafka-1       ... done
Creating kafka-0       ... done
Creating kafka-2       ... done
Attaching to nifi-registry, nifi, kafka-2, kafka-1, kafka-0
```

## Kafka topics

Inside any Kafka container, e.g. `kafka-0`:

```bash
~$ docker exec -it kafka-0 bash
```

Create a new replicated topic:

```bash
/$ kafka-topics.sh --create --bootstrap-server kafka:9092 --partitions 3 --replication-factor 3 --topic topic-poc
Created topic topic-poc.
```

List existing topics:

```bash
/$ kafka-topics.sh --list --bootstrap-server kafka:9092
topic-poc
```

Describe topic:

```bash
/$ kafka-topics.sh --describe --bootstrap-server kafka:9092 --topic topic-poc
Topic: topic-poc	TopicId: P3xSUW4kRsKCXjxuG_yvbA	PartitionCount: 3	ReplicationFactor: 3	Configs:
	Topic: topic-poc	Partition: 0	Leader: 2	Replicas: 2,0,1	Isr: 2,0,1
	Topic: topic-poc	Partition: 1	Leader: 0	Replicas: 0,1,2	Isr: 0,1,2
	Topic: topic-poc	Partition: 2	Leader: 1	Replicas: 1,2,0	Isr: 1,2,0
```

If you need to delete existing topic:

```bash
/$ kafka-topics.sh --bootstrap-server kafka:9092 --topic topic-poc --delete
```

To obtain consumers details and the offset of each partition:

```bash
/$ kafka-consumer-groups.sh --bootstrap-server kafka-0:9092 --group group-a --describe

GROUP    TOPIC      PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG  CONSUMER-ID                 HOST         CLIENT-ID
group-a  topic-poc  2          17              17              0    consumer-group-a-1-e23de045 /172.28.0.6  consumer-group-a-1
group-a  topic-poc  1          29              29              0    consumer-group-a-1-73c8e32d /172.28.0.5  consumer-group-a-1
group-a  topic-poc  0          27              27              0    consumer-group-a-1-65f1fbb8 /172.28.0.7  consumer-group-a-1
```


### Kafka consumer

Create 3 consumers on 3 separate consoles, all members of the same group `group-a`. These consumers stay open waiting for new messages to arrive.

#### Consumer 0

Inside conteniner `kafka-0` connect to topic `topic-poc` using `kafka-0:9092` as bootstrap server:

```bash
~$ docker exec -it kafka-0 bash
/$ kafka-console-consumer.sh --bootstrap-server kafka-0:9092 --topic topic-poc --from-beginning --group group-a
```

#### Consumer 1

Inside conteniner `kafka-1` connect to topic `topic-poc` using `kafka-1:9192` as bootstrap server:

```bash
~$ docker exec -it kafka-1 bash
/$ kafka-console-consumer.sh --bootstrap-server kafka-1:9192 --topic topic-poc --from-beginning --group group-a
```

#### Consumer 2

Inside conteniner `kafka-2` connect to topic `topic-poc` using `kafka-2:9292` as bootstrap server:

```bash
~$ docker exec -it kafka-2 bash
/$ kafka-console-consumer.sh --bootstrap-server kafka-2:9292 --topic topic-poc --from-beginning --group group-a
```


### Kafka producer

Inside any Kafka container, e.g. `kafka-0`:

```bash
~$ docker exec -it kafka-0 bash
/$ kafka-console-producer.sh --broker-list kafka-0:9092,kafka-1:9192,kafka-2:9292 --topic topic-poc
>
```

At this time, the prompt expects you to enter a message and press enter, por example:

```bash
>aaa
>bbb
>ccc
>ddd
...
```

Each message will be sent to the same Kafka topic `topic-poc`, in the other hand the 3 consumers from the same group `group-a` will consume the messages, balancing the reading load and avoiding reading duplicate messages.

<p align="center"><img src="img/kafka_cluster_00.gif"></p>
<br>


## Apache NiFi
If Apache NiFi has started successfully it will be available at the URL: http://localhost:8080/nifi

### Configure TLS
```bash
~$ docker exec -it nifi-0 bash
nifi@nifi-0:/opt/nifi/nifi-current$ ../nifi-toolkit-current/bin/tls-toolkit.sh standalone -n nifi-0,nifi-1 -C 'CN=admin, OU=NIFI' -o /tmp/ssl
```

```bash
nifi@nifi-0:/opt/nifi/nifi-current$ cd /tmp/ssl/
nifi@nifi-0:/tmp/ssl$ ls -lrt
total 24
-rw------- 1 nifi nifi 1233 Aug  3 17:25  nifi-cert.pem
-rw------- 1 nifi nifi 1675 Aug  3 17:25  nifi-key.key
drwx------ 2 nifi nifi 4096 Aug  3 17:25  nifi-0
drwx------ 2 nifi nifi 4096 Aug  3 17:25  nifi-1
-rw------- 1 nifi nifi   43 Aug  3 17:25 'CN=admin_OU=NIFI.password'
-rw------- 1 nifi nifi 3501 Aug  3 17:25 'CN=admin_OU=NIFI.p12'
```

Copy cert and password files from container to local:
```bash
~$ docker cp nifi-0:/tmp/ssl/CN=admin_OU=NIFI.p12 .
~$ docker cp nifi-0:/tmp/ssl/CN=admin_OU=NIFI.password .
```

```bash
nifi@nifi-0:/tmp/ssl$ cd /opt/nifi/nifi-current/conf
nifi@nifi-0:/opt/nifi/nifi-current/conf$ cp nifi.properties nifi.properties_bak
nifi@nifi-0:/opt/nifi/nifi-current/conf$ cp keystore.p12 keystore.p12_bak
nifi@nifi-0:/opt/nifi/nifi-current/conf$ cp truststore.p12 truststore.p12_bak
nifi@nifi-0:/opt/nifi/nifi-current/conf$ cp authorizers.xml authorizers.xml_bak
nifi@nifi-0:/opt/nifi/nifi-current/conf$ mv authorizations.xml authorizations.xml_bak
nifi@nifi-0:/opt/nifi/nifi-current/conf$ mv users.xml users.xml_bak
```

```bash
nifi@nifi-0:/opt/nifi/nifi-current/conf$ cp /tmp/ssl/nifi-0/* .
```

Edit `authorizers.xml` file.
Here:

```xml
<property name="Initial Admin Identity"></property>
```

add `CN=JavDomGom, OU=Apache NiFi` into `property` tag as follows:

```xml
<property name="Initial Admin Identity">CN=admin, OU=NIFI</property>
```

Save and close.

Now edit `nifi.properties`:
```properties
nifi.web.proxy.host=nifi-0:8443,nifi-1:8443
nifi.cluster.is.node=true
nifi.cluster.node.protocol.port=8082
nifi.cluster.flow.election.max.wait.time=1 min
```

Go to `bin` directory:

```bash
nifi@nifi-0:/opt/nifi/nifi-current/conf$ cd ../bin/
```

Stop NiFi:

```bash
nifi@nifi-0:/opt/nifi/nifi-current/bin$ ./nifi.sh stop
```

Start NiFi and enter into container again:

```bash
~$ docker start nifi-0
~$ docker exec -it nifi-0 bash
```

```bash

```

## Apache NiFi Registry
If Apache NiFi Registry has started successfully it will be available at the URL: http://localhost:18080/nifi-registry
