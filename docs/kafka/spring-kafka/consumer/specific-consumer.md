# 특정 Topic-Partition에 바인딩하는 Consumer
이번 시간에는 특정 topic-partition 조합에 대해서만 수신하는 consumer에 대해 배워보겠습니다.
기본적으로 하나의 consumer group 내에 여러개의 consumer instance가 존재하는 경우 각각의 instance는 특정 topic-partition을 할당받습니다.

대부분의 경우에는 하나의 topic에 대한 수신 로직은 동일할거라 판단됩니다.
하지만 요구사항에 따라 특정 partition으로 들어오는 레코드에 대해 다르게 처리하는 로직을 구현할 수 있습니다.

<br/>

## Consumer 설정 및 구현
특이 설정이 없으므로 지난 시간에 만든 containerFactory로 진행하도록 하겠습니다.

먼저 특정 partition에 시작 offset을 지정해주는 방법입니다.

```java
@KafkaListener(containerFactory = "basicListenerContainerFactory", groupId = groupId,
		topicPartitions = @TopicPartition(topic = "test-topic-mp", partitionOffsets = @PartitionOffset(partition = "0", initialOffset = "0")))
public void listen(@Payload String message,
		@Header(KafkaHeaders.RECEIVED_PARTITION_ID) int partition,
		@Header(KafkaHeaders.RECEIVED_MESSAGE_KEY) String messageKey) {
    System.out.println("[group.id : " + groupId + ", key : " + messageKey + ", partition: " + partition + " consumed : " + message);
    // handle business
}
```

위 코드에서 특이한 점은 두 가지가 있습니다.
첫 번째는 `@TopicPartition`과 `@PartitionOffset` 어노테이션입니다.
topicPartitions 파라미터값을 보시면 이해하기에는 어렵지 않을거라 생각됩니다. 
`topicPartitions = @TopicPartition(topic = "test-topic-mp", partitionOffsets = @PartitionOffset(partition = "0", initialOffset = "0"))` 

두 번째는 레코드의 header와 payload 부분을 `@Header`와 `@Payload` 어노테이션을 활용하여 레코드에 대한 특정 정보를 쉽게 가져왔다는 점 입니다.
Producer를 구현하면서 사용했던 `Message<?>`를 수신해서 header와 payload를 구분해서 가져오는 방법도 추후에 배우겠지만
레코드 내에 특정 데이터를 쉽게 가져와서 핸들링하는 부분은 상당히 유용합니다.

<br/>

이번에는 위 방법과 다르게 여러 개 partition 중 특정 몇 개의 partition만 수신하는 방법입니다.
```java
@KafkaListener(containerFactory = "basicListenerContainerFactory", groupId = groupId,
		topicPartitions = @TopicPartition(topic = "test-topic-mp", partitions = { "1", "2" }))
public void listen2(@Payload String message,
		@Header(KafkaHeaders.RECEIVED_PARTITION_ID) int partition,
		@Header(KafkaHeaders.RECEIVED_MESSAGE_KEY) String messageKey) {
    System.out.println("[group.id : " + groupId + ", key : " + messageKey + ", partition: " + partition + " consumed : " + message);
    // handle business
}
```
이번에는 `patitionOffsets` 대신 `partitions`라는 파라미터에 수신하고자 한 partition 숫자를 준 것을 확인할 수 있습니다.

<br/>

## 최종 모습
```java
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.annotation.PartitionOffset;
import org.springframework.kafka.annotation.TopicPartition;
import org.springframework.kafka.support.KafkaHeaders;
import org.springframework.messaging.handler.annotation.Header;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.stereotype.Component;

/**
 * 2강: 특정 파티션에 바인딩해서 수신
 */
@Component
public class LessonTwoSpecificPartitionConsumer {

    private final String groupId = "test-group-partition";

	@KafkaListener(containerFactory = "basicListenerContainerFactory", groupId = groupId,
			topicPartitions = @TopicPartition(topic = "test-topic-mp", partitionOffsets = @PartitionOffset(partition = "0", initialOffset = "0")))
	public void listen(@Payload String message,
			@Header(KafkaHeaders.RECEIVED_PARTITION_ID) int partition,
			@Header(KafkaHeaders.RECEIVED_MESSAGE_KEY) String messageKey) {
		System.out.println("[group.id : " + groupId + ", key : " + messageKey + ", partition: " + partition + " consumed : " + message);
		// handle business
	}

	@KafkaListener(containerFactory = "basicListenerContainerFactory", groupId = groupId,
			topicPartitions = @TopicPartition(topic = "test-topic-mp", partitions = { "1", "2" }))
	public void listen2(@Payload String message,
			@Header(KafkaHeaders.RECEIVED_PARTITION_ID) int partition,
			@Header(KafkaHeaders.RECEIVED_MESSAGE_KEY) String messageKey) {
		System.out.println("[group.id : " + groupId + ", key : " + messageKey + ", partition: " + partition + " consumed : " + message);
		// handle business
	}
}
```

<br/>

## Test 방법 
### Kafka CLI를 활용하여 수신 테스트 방법
여러 개의 partition으로 구성된 topic을 생성한 다음 여러 개의 레코드를 송신하고 어느 시점에 위 수신 메소드들이 실행되는지 확인해보세요.

팁: Console Producer 활용 방법 <br/>
$ `kafka-console-producer.sh --broker-list localhost:9092 --topic 토픽명`

## 참고 사항
* consumer group
* rebalancing
