## Building a Docker file

```
cd .\messaging\kafka\
docker build . -t aimvector/kafka:3.7.0
```

## Install Kafka

```
docker run --rm --name kafka -it aimvector/kafka:3.7.0 bash
```

# Check files
```
ls -l /kafka/bin/
cat /kafka/config/server.properties
```

We can use the `docker cp` command to copy the file out of our container:

```
docker cp kafka:/kafka/config/server.properties ./server.properties
docker cp kafka:/kafka/config/zookeeper.properties ./zookeeper.properties
```

Note: We'll need the Kafka configuration to tune our server and Kafka also requires
at least one Zookeeper instance in order to function. To achieve high availability, we'll run
multiple kafka as well as multiple zookeeper instances in the future

# Zookeeper

Let's build a Zookeeper image. The Apache folks have made it easy to start a Zookeeper instance the same way as the Kafka instance by simply running the `start-zookeeper.sh` script.

```
cd ./zookeeper
docker build . -t aimvector/zookeeper:3.7.0
cd ..
```

Create a kafka network and run 1 zookeeper instance

```
docker network create kafka
docker run -d `
--rm `
--name zookeeper-1 `
--net kafka `
-v ${PWD}/config/zookeeper-1/zookeeper.properties:/kafka/config/zookeeper.properties `
aimvector/zookeeper:3.7.0

docker logs zookeeper-1
```

# Kafka - 1

```
docker run -d `
--rm `
--name kafka-1 `
--net kafka `
-v ${PWD}/config/kafka-1/server.properties:/kafka/config/server.properties `
aimvector/kafka:3.7.0

docker logs kafka-1
```

# Kafka - 2

```
docker run -d `
--rm `
--name kafka-2 `
--net kafka `
-v ${PWD}/config/kafka-2/server.properties:/kafka/config/server.properties `
aimvector/kafka:3.7.0
```

# Kafka - 3

```
docker run -d `
--rm `
--name kafka-3 `
--net kafka `
-v ${PWD}/config/kafka-3/server.properties:/kafka/config/server.properties `
aimvector/kafka:3.7.0
```


# Topic

Create a Topic that allows us to store `Order` information. </br>
To create a topic, Kafka and Zookeeper have scripts with the installer that allows us to do so. </br>

Access the container:
```
docker exec -it zookeeper-1 bash
```
Create the Topic:
/kafka/bin/kafka-topics.sh \
--create \
--bootstrap-server kafka-1:9092,kafka-2:9092,kafka-3:9092 \
--replication-factor 1 \
--partitions 3 \
--topic Orders

Describe our Topic:
```
/kafka/bin/kafka-topics.sh \
--describe \
--topic Orders \
--bootstrap-server kafka-1:9092,kafka-2:9092,kafka-3:9092
```

# Simple Producer & Consumer

The Kafka installation also ships with a script that allows us to produce
and consume messages to our Kafka network: <br/>

We can then run the consumer that will receive that message on that Orders topic:

```
docker exec -it zookeeper-1 bash

/kafka/bin/kafka-console-consumer.sh \
--bootstrap-server kafka-1:9092,kafka-2:9092,kafka-3:9092 \
--topic Orders --from-beginning

```

With a consumer in place, we can start producing messages

```
docker exec -it zookeeper-1 bash

echo "New Order: 1" | \
/kafka/bin/kafka-console-producer.sh \
--broker-list kafka-1:9092,kafka-2:9092,kafka-3:9092 \
--topic Orders > /dev/null
```


Once we have a message in Kafka, we can explore where it got stored in which partition:

```
docker exec -it kafka-1 bash

apt install -y tree
tree /tmp/kafka-logs/


ls -lh /tmp/kafka-logs/Orders-*

/tmp/kafka-logs/Orders-0:
total 4.0K
total 8.0K
-rw-r--r-- 1 root root 10M Jun 28 18:23 00000000000000000000.index   
-rw-r--r-- 1 root root   0 Jun 28 18:23 00000000000000000000.log     
-rw-r--r-- 1 root root 10M Jun 28 18:23 00000000000000000000.timeindex
-rw-r--r-- 1 root root   8 Jun 28 18:23 leader-epoch-checkpoint      
-rw-r--r-- 1 root root  43 Jun 28 18:23 partition.metadata

/tmp/kafka-logs/Orders-1:
total 12K
-rw-r--r-- 1 root root 10M Jun 28 18:23 00000000000000000000.index   
-rw-r--r-- 1 root root  80 Jun 28 18:26 00000000000000000000.log     
-rw-r--r-- 1 root root 10M Jun 28 18:23 00000000000000000000.timeindex
-rw-r--r-- 1 root root   8 Jun 28 18:23 leader-epoch-checkpoint      
-rw-r--r-- 1 root root  43 Jun 28 18:23 partition.metadata 

/tmp/kafka-logs/Orders-2:
total 8.0K
-rw-r--r-- 1 root root 10M Jun 28 18:23 00000000000000000000.index   
-rw-r--r-- 1 root root   0 Jun 28 18:23 00000000000000000000.log     
-rw-r--r-- 1 root root 10M Jun 28 18:23 00000000000000000000.timeindex
-rw-r--r-- 1 root root   8 Jun 28 18:23 leader-epoch-checkpoint      
-rw-r--r-- 1 root root  43 Jun 28 18:23 partition.metadata
```

By seeing 0 bytes in partition 0 and 2, we know the message is sitting in partition 1 as it has 80 bytes. </br>
We can check the message with :

```
cat /tmp/kafka-logs/Orders-1/*.log
```