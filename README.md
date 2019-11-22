# Please do not do this

You want to have replication on a specific DC, but not on another

That's okay, but you need to have the same `KAFKA_INTER_BROKER_LISTENER_NAME` on all brokers.

If not, you'll hit https://github.com/apache/kafka/blob/trunk/core/src/main/scala/kafka/cluster/Broker.scala#L80


```
kafka.common.BrokerEndPointNotAvailableException: End point with listener name REPLICATION not found for broker 3
kafka-2         | 	at kafka.cluster.Broker.$anonfun$node$1(Broker.scala:53)
```

Here are a few commands to test this

```
docker-compose up -d

docker-compose ps
```

Please note that I am forcing `min.insync.replicas=3` to make sure wait for all replication to happen.

```
docker-compose exec kafka-client kafka-topics --zookeeper zookeeper:2181 --create --topic demo --partitions 3 --replication-factor 3 --config min.insync.replicas=3

docker-compose exec kafka-client kafka-topics --zookeeper zookeeper:2181 --topic demo --describe

docker-compose exec kafka-client kafka-producer-perf-test \
        --topic demo \
        --num-records 1000 \
        --record-size 100 \
        --throughput -1 \
        --producer-props \
            ack=all \
            linger.ms=0 \
            bootstrap.servers=kafka-1:9092,kafka-2:9092,kafka-3:9092 \
            compression.type=zstd

for id in 1 2 3; do
    echo Broker $id
    docker-compose exec zookeeper zookeeper-shell localhost:2181 get /brokers/ids/$id | grep endpoint | jq .
done
```

Please DO NOT DO THIS. Please simplify as much as you can your deployment.

(Be nice to your fellow support team)

# This is a strange dc2

My guess, is that this dc2 is of lower quality than your dc1, so you probably want to have all leaders on dc1.

Let's do that by creating a topic with specific replica assignment.

```
docker-compose exec kafka-client kafka-topics --zookeeper zookeeper:2181 --create --topic demo-no-leader-on-dc2 --replica-assignment "1:2:3,2:1:3,1:2:3"
docker-compose exec kafka-client kafka-topics --zookeeper zookeeper:2181 --topic demo-no-leader-on-dc2 --describe
```

# See it live

[![asciicast](https://asciinema.org/a/283026.svg)](https://asciinema.org/a/283026?autoplay=true)