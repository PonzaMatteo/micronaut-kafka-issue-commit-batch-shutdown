# micronaut-kafka-issue-commit-batch-shutdown

## Requirements

- [kcat](https://docs.confluent.io/platform/current/app-development/kafkacat-usage.html)
- gradle

Sample app for reproducing issue: https://github.com/micronaut-projects/micronaut-kafka/issues/607

The app listen to the topic `testing-kafka-shutdown` and log one message and waits for 1s.

In my local environment shutting down the application after a few seconds produce an offset
of 500 messages. Looks like the whole batch of messages is committed before the individual
messages can possibly be processed even tough the offset strategy of the consumer
is `OffsetStrategy.SYNC_PER_RECORD`

### 0. Setup

Build the docker layers for the example application:

```bash
./gradlew dockerfile buildLayers
```

Start kafka container:

```bash
docker compose up kafka -d
```

Add 1000 messages to test topic:

```bash
seq 1000 | xargs -I _ echo 'Message#_' | kcat -P -t "testing-kafka-shutdown" -b localhost:29092   
```

Start the application in docker (log 1 message and sleep for 1 seconds)

```bash
docker compose up example-kafka -d 
```

Stop it after a few seconds:

```bash
docker stop example-kafka
```

Check messages processed logs:

```bash
docker logs example-kafka | grep 'Processing message'
```

List offset for consumer `myGroupId`

```bash
docker exec -it kafka kafka-consumer-groups --bootstrap-server localhost:9092 --group myGroupId --describe
```

If you want to re-try the experiment can reset the offset with:

```bash
docker exec -it kafka kafka-consumer-groups --bootstrap-server localhost:9092 --group myGroupId --reset-offsets --to-earliest --topic testing-kafka-shutdown --execute
```