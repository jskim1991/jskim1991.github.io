# Producer with Message Keys

이번에는 Message Key를 활용하여 토픽 내 특정 파티션에 송신하는 방법에 대해 알아보겠습니다.

## 목적
1. 특정 레코드를 하나의 파티션으로 보내 순차적으로 처리

## Producer 구현
### Producer 설정
지난 글에서 사용한 Producer와 설정 차이가 없어 동일한 설정과 동일한 `KafkaTemplate`으로 진행합니다.

### 코드 설명
Producer가 여러개의 파티션으로 구성된 토픽에 송신하는 경우, 레코드에 Message Key의 존재 여부에 따라 다르게 동작합니다.
- Message Key가 존재하는 경우: Kafka가 Message key를 기반으로 Hashing을 통해 파티션을 결정 
- Message Key가 없는 경우: Round-robin을 기반으로 파티션을 결정

그럼 동일한 Message key를 가진 레코드들은 항상 동일한 파티션으로 향하며, 그 레코드들에 대해서는 순차적으로 처리를 할 수 있습니다.

#### 랜덤 파티션으로 송신하는 경우
이전 글에서 사용하던 송신 부분 코드입니다.
파티션이 여러 개인 토픽일 때 아래 코드는 key를 넘겨주는 부분이 없으므로 송신할 때 어느 파티션으로 갈지 예측할 수 없습니다.
```java
public void sendMessageNoMessageKey(String message) {
    Message<String> record = MessageBuilder.withPayload(message)
                                 .setHeader(KafkaHeaders.TOPIC, topic)
                                 .build();
    kafkaTemplate.send(record);
}
```

#### 특정 파티션으로 송신하는 경우
`KafkaTemplate`에서 제공하는 메소드를 보면 Message key로 파티션을 보장하는 방법은 두 가지가 있습니다.
1. `send(String topic, K key, @Nullable V data)` 메소드에 직접 Message key를 넘기는 방법이 있으며,
2. `send(Message<?>)` 메소드를 활용하는 경우는 `KafkaHeaders.MESSAGE_KEY` 헤더 값에 key를 줄 수 있습니다.

첫 번째 메소드 하위 부분 보시면 파티션을 직접 설정하는 방법도 있습니다. 이 부분은 참고만 하셔도 될 거라 생각합니다.
개인적인 생각에는 key를 기반으로 파티션을 정하는 게 더 유용할 거라 판단됩니다. 
그 이유는 파티션 수는 변경될 수 있기 때문입니다 (Kafka client가 실행 중에는 파티션 증가만 가능합니다).

```java
public void sendMessage(String key, String message) {
    /* Message with same keys will be guaranteed to be in same partition by Kafka */
    // using key
    kafkaTemplate.send(topic, key, message);

    // using MESSAGE_KEY header in Message<?>
    Message<String> record = MessageBuilder.withPayload(message)
                                 .setHeader(KafkaHeaders.TOPIC, topic)
                                 .setHeader(KafkaHeaders.MESSAGE_KEY, key)
                                 .build();
    kafkaTemplate.send(record);


    /* 참고 - Assign partitions explicitly */
    // users can assign partition id (note: partition starts at 0)
    kafkaTemplate.send(topic, 1, key, message);

    // similarly, it is possible to set partition id in Message<?> headers
    Message<String> recordWithPartition = MessageBuilder.fromMessage(record)
                                              .setHeader(KafkaHeaders.PARTITION_ID, 1)
                                              .build();
    kafkaTemplate.send(recordWithPartition);
}
```


### 최종 모습
```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.support.KafkaHeaders;
import org.springframework.messaging.Message;
import org.springframework.messaging.support.MessageBuilder;
import org.springframework.stereotype.Component;

@Component
public class SimplePartitionProducer {

    // Create topic with multiple partitions
    // ./kafka-topics.sh --zookeeper localhost:2181 --create --topic 토픽명 --partitions 3 --replication-factor 1

    @Value("${kafka.topic-mp:sample-topic-mp}")
    private String topic;

    @Autowired
    @Qualifier("simpleProducerKafkaTemplate")
    private KafkaTemplate<String, String> kafkaTemplate;

    public void sendMessage(String key, String message) {

        /* Message with same keys will be guaranteed to be in same partition by Kafka */
        // using key
        kafkaTemplate.send(topic, key, message);

        // using Message<?> - set in Headers
        Message<String> record = MessageBuilder.withPayload(message)
                                    .setHeader(KafkaHeaders.TOPIC, topic)
                                    .setHeader(KafkaHeaders.MESSAGE_KEY, key)
                                    .build();
        kafkaTemplate.send(record);


        /* Assign partitions explicitly */
        // users can assign partition id (note: partition starts at 0)
        kafkaTemplate.send(topic, 1, key, message);

        // similarly, it is possible to set partition id in Message<?> headers
        Message<String> recordWithPartition = MessageBuilder.fromMessage(record)
                                                .setHeader(KafkaHeaders.PARTITION_ID, 1)
                                                .build();
        kafkaTemplate.send(recordWithPartition);
    }

    public void sendMessageNoMessageKey(String message) {
        Message<String> record = MessageBuilder.withPayload(message)
                                            .setHeader(KafkaHeaders.TOPIC, topic)
                                            .build();
        kafkaTemplate.send(record);
    }
}
```

## Test 방법 
### Test용 REST API를 만들어 송신하기 
```java
@RestController
@RequestMapping("producer")
public class ProducerController {

    @Autowired
    @Qualifier("simplePartitionProducer")
    private SimplePartitionProducer partitionProducer;
    
    @PostMapping("/simple-partition")
    public void sendUsingSimplePartitionProducer(@RequestParam String key, @RequestBody String message) {
        this.partitionProducer.sendMessage(key, message);
    }
    
    @PostMapping("/simple-partition-no-key")
    public void sendUsingSimplePartitionNoKey(@RequestBody String message) {
        this.partitionProducer.sendMessageNoMessageKey(message);
    }
}
```
localhost:8080/producer/simple-partition 호출 시 동일한 key를 가진 레코드는 항상 동일한 파티션에만 레코드가 쌓이는 것을 확인할 수 있습니다. 
localhost:8080/producer/simple-partition-no-key 호출 시 레코드가 랜덤한 파티션으로 가는 것을 확인할 수 있습니다.

## 테스트 결과
팁: kafka경로/bin 디렉토리에서 아래 실행 명령어를 사용하면 파티션별 쌓인 레코드 개수를 확인할 수 있습니다.
`./kafka-run-class.sh kafka.tools.GetOffsetShell --broker-list localhost:9092 --topic 토픽명`

예시:
```
./kafka-run-class.sh kafka.tools.GetOffsetShell --broker-list localhost:9092 --topic test-topic-mp
test-topic-mp:0:3
test-topic-mp:1:7
test-topic-mp:2:8
```