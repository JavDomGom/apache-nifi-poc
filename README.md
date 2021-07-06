# apache-nifi-poc
Apache NiFi PoC

## Run Docker Compose

```bash
~$ docker-compose up
Creating network "apache-nifi-poc_internal" with driver "bridge"
Creating volume "apache-nifi-poc_zookeeper_data" with local driver
Creating volume "apache-nifi-poc_kafka_data" with local driver
Creating nifi      ... done
Creating zookeeper ... done
Creating kafka     ... done
Attaching to nifi, zookeeper, kafka
```

## Kafka topics

Inside Kafka container:

```bash
~$ docker exec -it kafka bash
```

Create a new topic:

```bash
~$ kafka-topics.sh --create --zookeeper zookeeper:2181 --replication-factor 1 --partitions 1 --topic topic-nifi
Created topic topic-nifi.
```

List existing topics:

```bash
~$ kafka-topics.sh --list --zookeeper zookeeper:2181
topic-nifi
```

### Kafka producer

Connect to topic using `kafka:9092` as broker:

```bash
~$ kafka-console-producer.sh --broker-list kafka:9092 --topic topic-nifi
>
```

At this time, the prompt expects you to enter a message and press enter, por example:

```bash
>Hello world 0!
>Hello world 1!
>Hello world 2!
>Blah, blah, blah...
```

All these messages are sent to Kafka.


### Kafka consumer

Connect to topic using `kafka:9092` as bootstrap server to consume all stored messages (*--from-beginning*):

```bash
~$ kafka-console-consumer.sh --bootstrap-server kafka:9092 --topic topic-nifi --from-beginning
Hello world 0!
Hello world 1!
Hello world 2!
Blah, blah, blah...
```

The consumer is left open waiting for new messages to come in. You can create all the consumers you need pointing to the same topic.


## Apache NiFi
If Apache NiFi has started successfully it will be available at the URL: http://localhost:8080/nifi
