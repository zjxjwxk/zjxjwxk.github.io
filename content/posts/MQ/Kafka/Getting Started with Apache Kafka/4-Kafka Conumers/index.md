---
title: Kafka Consumer
date: 2024-11-18T18:41:00+08:00
tags:
- MQ
- Kafka
categories:
- MQ
ShowToc: true
preview: 200
---

## Kafka Consumer Group

- Consumers are typically done as a group.
- A single consumer will end up inefficient with large amounts of data.
- A consumer may never catch up.
- Every consumer should be on it's own machine, instance, pod.

The consumer group ID is the key so Kafka knows that messages should be distributed to both consumers without duplicating.

![image-20241118185205006](images/image-20241118185205006.png)

If we add one more consumer to this group, the last one will be idle, because one partition can't be share across consumers. One partition can only be assigned to one consumer. Instead, the partitions are the way of Kafka to scale. More partitions imply you can have more consumers in the same consumer group.

![image-20241118185601440](images/image-20241118185601440.png)

## Consuming from Kafka

Create a docker-compose file `docker-compose.yaml` containing 3 Zookeepers, 3 Kafka Brokers, and 1 Kafka REST Proxy:

```yaml
---
version: '3'
services:
  zookeeper-1:
    image: confluentinc/cp-zookeeper:7.4.1
    hostname: zookeeper-1
    container_name: zookeeper-1
    volumes:
      - ./zookeeper-1_data:/var/lib/zookeeper/data
      - ./zookeeper-1_log:/var/lib/zookeeper/log
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
      ZOO_MY_ID: 1
      ZOO_SERVERS: server.1=zookeeper-1:2888:3888;2181 server.2=zookeeper-2:2888:3888;2181 server.3=zookeeper-3:2888:3888;2181

  zookeeper-2:
    image: confluentinc/cp-zookeeper:7.4.1
    hostname: zookeeper-2
    container_name: zookeeper-2
    volumes:
      - ./zookeeper-2_data:/var/lib/zookeeper/data
      - ./zookeeper-2_log:/var/lib/zookeeper/log
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
      ZOO_MY_ID: 2
      ZOO_SERVERS: server.1=zookeeper-1:2888:3888;2181 server.2=zookeeper-2:2888:3888;2181 server.3=zookeeper-3:2888:3888;2181

  zookeeper-3:
    image: confluentinc/cp-zookeeper:7.4.1
    hostname: zookeeper-3
    container_name: zookeeper-3
    volumes:
      - ./zookeeper-3_data:/var/lib/zookeeper/data
      - ./zookeeper-3_log:/var/lib/zookeeper/log
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
      ZOO_MY_ID: 3
      ZOO_SERVERS: server.1=zookeeper-1:2888:3888;2181 server.2=zookeeper-2:2888:3888;2181 server.3=zookeeper-3:2888:3888;2181


  broker-1:
    image: confluentinc/cp-kafka:7.4.1
    hostname: broker-1
    container_name: broker-1
    volumes:
      - ./broker-1-data:/var/lib/kafka/data
    depends_on:
      - zookeeper-1
      - zookeeper-2
      - zookeeper-3
    ports:
      - 9092:9092
      - 29092:29092
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper-1:2181
      KAFKA_ADVERTISED_LISTENERS: HOST://localhost:9092,INTERNAL://broker-1:29092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: HOST:PLAINTEXT,INTERNAL:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: INTERNAL
      KAFKA_SNAPSHOT_TRUST_EMPTY: true

  broker-2:
    image: confluentinc/cp-kafka:7.4.1
    hostname: broker-2
    container_name: broker-2
    volumes:
      - ./broker-2-data:/var/lib/kafka/data
    depends_on:
      - zookeeper-1
      - zookeeper-2
      - zookeeper-3
      - broker-1
    ports:
      - 9093:9093
      - 29093:29093
    environment:
      KAFKA_BROKER_ID: 2
      KAFKA_ZOOKEEPER_CONNECT: zookeeper-1:2181
      KAFKA_ADVERTISED_LISTENERS: HOST://localhost:9093,INTERNAL://broker-2:29093
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: HOST:PLAINTEXT,INTERNAL:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: INTERNAL
      KAFKA_SNAPSHOT_TRUST_EMPTY: true

  broker-3:
    image: confluentinc/cp-kafka:7.4.1
    hostname: broker-3
    container_name: broker-3
    volumes:
      - ./broker-3-data:/var/lib/kafka/data
    depends_on:
      - zookeeper-1
      - zookeeper-2
      - zookeeper-3
      - broker-1
      - broker-2
    ports:
      - 9094:9094
      - 29094:29094
    environment:
      KAFKA_BROKER_ID: 3
      KAFKA_ZOOKEEPER_CONNECT: zookeeper-1:2181
      KAFKA_ADVERTISED_LISTENERS: HOST://localhost:9094,INTERNAL://broker-3:29094
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: HOST:PLAINTEXT,INTERNAL:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: INTERNAL
      KAFKA_SNAPSHOT_TRUST_EMPTY: true


  rest-proxy:
    image: confluentinc/cp-kafka-rest:7.4.1
    ports:
      - "8082:8082"
    depends_on:
      - zookeeper-1
      - zookeeper-2
      - zookeeper-3
      - broker-1
      - broker-2
      - broker-3
    hostname: rest-proxy
    container_name: rest-proxy
    environment:
      KAFKA_REST_HOST_NAME: rest-proxy
      KAFKA_REST_BOOTSTRAP_SERVERS: 'broker-1:29092,broker-2:29093,broker-3:29094'
      KAFKA_REST_LISTENERS: "http://0.0.0.0:8082"
```

Run composed containers:

```bash
docker compose up -d
```

### Create Topic

Create a topic `myorders` with 2 partitions:

```bash
kafka-topics.sh --create --bootstrap-server localhost:9092 --partitions 2 --topic myorders
```

### Run Consumer

Run 3 Kafka Consumers in different terminals lisitening to the same topic `myorders` and deserialize the messages by string key and double value seperated by `,`:

```bash
kafka-console-consumer.sh \
--bootstrap-server localhost:9092 \
--topic myorders --from-beginning \
--key-deserializer org.apache.kafka.common.serialization.StringDeserializer \
--value-deserializer org.apache.kafka.common.serialization.DoubleDeserializer \
--property print.key=true \
--property key.separator=, \
--group 1
```

Then, in another terminal, run another consumer with all the other same parameters but with `--group 2`.

We can see check the logs of broker 1:

```bash
docker logs broker-1
```

The following log can be found, which means there are three members in the group 1:

```
[2024-11-18 14:36:14,337] INFO [GroupCoordinator 1]: Stabilized group 1 generation 3 (__consumer_offsets-49) with 3 members (kafka.coordinator.group.GroupCoordinator)
[2024-11-18 14:36:14,346] INFO [GroupCoordinator 1]: Assignment received from leader console-consumer-340e7131-cbd4-4811-a62a-543fbac93948 for group 1 for generation 3. The group has 3 members, 0 of which are static. (kafka.coordinator.group.GroupCoordinator)
```

And in the logs of broker 2 (or 3), we can see that there is one consumer in the group 2:

```
[2024-11-18 14:38:49,987] INFO [GroupCoordinator 3]: Stabilized group 2 generation 1 (__consumer_offsets-0) with 1 members (kafka.coordinator.group.GroupCoordinator)
[2024-11-18 14:38:50,050] INFO [GroupCoordinator 3]: Assignment received from leader console-consumer-bebf5e94-8479-4ab8-b078-ced3f0d81d4c for group 2 for generation 1. The group has 1 members, 0 of which are static. (kafka.coordinator.group.GroupCoordinator)
```

### Send Messages

Run the following Java method to send messages to the topic `myorders`:

```java
public class Main {
	private static final Logger log = LoggerFactory.getLogger(Main.class);
	private static final String TOPIC = "myorders";

	public static void main(String[] args) throws InterruptedException {

		Properties props = new Properties();
		props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "http://localhost:9092,http://localhost:9093,http://localhost:9094");
		props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
		props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, DoubleSerializer.class.getName());
		KafkaProducer<String, Double> producer = new KafkaProducer<>(props);
		String stateString =
				"AK,AL,AZ,AR,CA,CO,CT,DE,FL,GA," +
						"HI,ID,IL,IN,IA,KS,KY,LA,ME, MD," +
						"MA,MI,MN,MS,MO,MT,NE,NV,NH,NJ," +
						"NM,NY,NC,ND,OH,OK,OR,PA,RI,SC," +
						"SD,TN,TX,UT,VT,VA,WA,WV,WI,WY";
		String[] stateArray = stateString.split(",");
		for (int i = 0; i < 25000; i++) {
			String key = stateArray[(int) Math.floor(Math.random()*(50))];
			double value = Math.floor(Math.random()* (10000-10+1)+10);
			ProducerRecord<String, Double> producerRecord =
					new ProducerRecord<>(TOPIC, key, value);

			log.info("Sending message with key " + key + " to Kafka");

			producer.send(producerRecord, (metadata, e) -> {
				if (metadata != null) {
					System.out.println(producerRecord.key());
					System.out.println(producerRecord.value());
					System.out.println(metadata.toString());
				}
			});
			Thread.sleep(5000);
		}
		producer.flush();
		producer.close();

		log.info("Successfully produced messages to " + TOPIC + " topic");

	}
}
```

We can see that the consumer in the group 2 read all messages:

```
MT,4138.0
LA,6868.0
KS,6318.0
MA,6184.0
NM,8817.0
NJ,6606.0
NV,9748.0
```

The consumer 1 in the group 1:

```
MT,4138.0
NV,9748.0
```

The consumer 2 (or 3) in the group 2:

```
LA,6868.0
KS,6318.0
MA,6184.0
NM,8817.0
NJ,6606.0
```

But consumer 3 (or 2) in the group 3 doesn't read any message. That's because there are only two partitions, which has been assigned to the other two consumers. So this consumer is idle. **One partition can only be assigned to one consumer.**

We can chek the detail of groups:

```bash
❯ kafka-consumer-groups.sh --all-groups --all-topics --bootstrap-server localhost:9092 --describe

GROUP           TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID                                           HOST            CLIENT-ID
1               myorders        0          13              13              0               console-consumer-340e7131-cbd4-4811-a62a-543fbac93948 /172.18.0.1     console-consumer
1               myorders        1          16              16              0               console-consumer-68f58739-69be-4f48-af94-590ee15add84 /172.18.0.1     console-consumer
GROUP           TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID                                           HOST            CLIENT-ID
2               myorders        1          16              16              0               console-consumer-bebf5e94-8479-4ab8-b078-ced3f0d81d4c /172.18.0.1     console-consumer
2               myorders        0          13              13              0               console-consumer-bebf5e94-8479-4ab8-b078-ced3f0d81d4c /172.18.0.1     console-consumer%
```

We can see that in the group 1, these two partitions are assigned to two different consumers. But in the group2, these two partitions are assigned to the same consumer.