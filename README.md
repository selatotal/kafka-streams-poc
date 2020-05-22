# kafka-streams-poc
Kafka Streams POC

This POC was created using instructions of [Kafka Streams Tutorial](https://kafka.apache.org/25/documentation/streams/tutorial).

## Setup

* **Install Kafka**

Create a Kafka User to execute Kafka
```bash
sudo useradd kafka -m
sudo passwd kafka
sudo adduser kafka sudo
```
Change to Kafka User
```bash
su -l kafka
```
Download Kafka
```bash
wget https://downloads.apache.org/kafka/2.5.0/kafka_2.12-2.5.0.tgz
```
Extract file
```bash
mkdir ~/kafka && cd ~/kafka
tar -xvzf ~/kafka_2.12-2.5.0.tgz --strip 1
```
Configure server
```bash
nano ~/kafka/config/server.properties

    Add the following line to enable delete topics

delete.topic.enable=true
```
You can use a docker-compose file in the repository root to start a Kafka Instance, using the following command:
```bash
docker-compose up
```
Create systemd for zookeeper
```bash
sudo nano /etc/systemd/system/zookeeper.service

[Unit]
Requires=network.target remote-fs.target
After=network.target remote-fs.target

[Service]
Type=simple
User=kafka
ExecStart=/home/kafka/kafka/bin/zookeeper-server-start.sh /home/kafka/kafka/config/zookeeper.properties
ExecStop=/home/kafka/kafka/bin/zookeeper-server-stop.sh
Restart=on-abnormal

[Install]
WantedBy=multi-user.target
```
Create systemd for kafka
```bash
sudo nano /etc/systemd/system/kafka.service

[Unit]
Requires=zookeeper.service
After=zookeeper.service

[Service]
Type=simple
User=kafka
ExecStart=/bin/sh -c '/home/kafka/kafka/bin/kafka-server-start.sh /home/kafka/kafka/config/server.properties > /home/kafka/kafka/kafka.log 2>&1'
ExecStop=/home/kafka/kafka/bin/kafka-server-stop.sh
Restart=on-abnormal

[Install]
WantedBy=multi-user.target
```
Start Kafka
```bash
sudo systemctl start kafka

    Check if OK
sudo journalctl -u kafka
```
* **Create Topics**

Create streams-plaintext-input topic
```bash
~/kafka/bin/kafka-topics.sh --create \
    --bootstrap-server localhost:9092 \
    --replication-factor 1 \
    --partitions 1 \
    --topic streams-plaintext-input
```
Create streams-wordcount-output topic
```bash
~/kafka/bin/kafka-topics.sh --create \
    --bootstrap-server localhost:9092 \
    --replication-factor 1 \
    --partitions 1 \
    --topic streams-pipe-output
```
Check if topis were created
```bash
~/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe
```

## Projects
This POC has 3 projects, as described below:

### Pipe
This project read messages from streams-plaintext-input and output the same message to streams-pipe-output topic.
To test, you should execute:
```bash
gradle run
```
After that, you can use kafka commands to publish and check messages, as below:
```bash
To publish:
~/kafka/bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic streams-plaintext-input

To listen:
~/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic streams-pipe-output --from-beginning --formatter kafka.tools.DefaultMessageFormatter --property print.key=true --property print.value=true --property key.deserializer=org.apache.kafka.common.serialization.StringDeserializer
```

### Line Split
This project read messages from streams-plaintext-input and output the message splitted by words to streams-linesplit-output topic.

Before you run, you need to create output topic in kafka, using the command below:
```bash
~/kafka/bin/kafka-topics.sh --create \
    --bootstrap-server localhost:9092 \
    --replication-factor 1 \
    --partitions 1 \
    --topic streams-linesplit-output
```
To test, you should execute:
```bash
gradle run
```
After that, you can use kafka commands to publish and check messages, as below:
```bash
To publish:
~/kafka/bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic streams-plaintext-input

To listen:
~/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic streams-linesplit-output --from-beginning --formatter kafka.tools.DefaultMessageFormatter --property print.key=true --property print.value=true --property key.deserializer=org.apache.kafka.common.serialization.StringDeserializer
```

### Wordcount
Count the number of each word sent to topic.
Before you run, you need to create output topic in kafka.
As this topic is a changelog stream, you should create with log compaction enable, using the command below:
```bash
~/kafka/bin/kafka-topics.sh --create \
    --bootstrap-server localhost:9092 \
    --replication-factor 1 \
    --partitions 1 \
    --topic streams-wordcount-output \
    --config cleanup.policy=compact
```
To test, you should execute:
```bash
gradle run
```
After that, you can use kafka commands to publish and check messages, as below:
```bash
To publish:
~/kafka/bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic streams-plaintext-input

To listen:
~/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 \
            --topic streams-wordcount-output \
            --from-beginning \
            --formatter kafka.tools.DefaultMessageFormatter \
            --property print.key=true \
            --property print.value=true \
            --property key.deserializer=org.apache.kafka.common.serialization.StringDeserializer \
            --property value.deserializer=org.apache.kafka.common.serialization.LongDeserializer
```
