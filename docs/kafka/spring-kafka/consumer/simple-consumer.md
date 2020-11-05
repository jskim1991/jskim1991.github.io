# Simple Consumer

이번에는 spring-kafka를 활용하여 가장 기본적인 Consumer를 구현해보겠습니다.

비즈니스 요구사항을 충족하는 Kafka Consumer를 구현하기 위해서는 Consumer에 대한 설정과 기본 이론을 이해하면 많은 도움이 되기 때문에
코드 설명 중간중간에 공부하시면 좋은 주제를 팁으로 드리겠습니다.

<br/>

## Consumer 구현
### Consumer 설정
Consumer를 구현하려면 `consumerFactory`, `containerFactory`, `@KafkaListener`를 구현해서 사용해야 합니다.
쉽게 생각하면 consumerFactory로 Kafka Consumer 설정을 정의하고
consumerFactory를 활용하여 containerFactory를 만들고
containerFactory를 @KafkaListener 파라미터로 넘겨서 Kafka 레코드를 수신하는 Spring Boot App.을 만들 수 있습니다.

먼저, consumerFactory는 consumer 인스턴스를 생성하기 위해 사용하는 Kafka 설정/전략입니다.
기존에 Producer Tutorial에서 `KafkaTemplate`을 만들기 위해 Map에 Kafka Producer 설정을 담은 것과 같다고 생각하시면 됩니다.
그럼 containerFactory를 만들어보겠습니다.
※ 참고: 시간 되시면 auto.offset.reset 설정에 대해 배워보세요. 
```java
public ConsumerFactory<String, String> consumerFactory() {
    Map<String, Object> props = new HashMap<>();
    props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
    props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
    props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
    props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
    return new DefaultKafkaConsumerFactory<>(props);
}
```

<br/>

다음으로 containerFactory는 왜 사용할까요?

containerFactory를 활용하면 consumer를 수행하는 Spring Boot App.에 대한 다양한 customizing이 가능합니다.
앞으로 하나씩 배워가겠지만, consumer가 수신 처리를 실패하면 그 에러를 어떻게 대처하며, 
어떤 방식으로 몇 번 retry를 하는 등 다양한 옵션을 구현할 수 있습니다.
또 다른 장점은 한 개의 containerFactory를 Spring Bean으로 만들면 동일한 모델을 다른 consumer에도 재사용할 수 있습니다.
그럼 심플한 containerFactory를 bean으로 생성해보겠습니다. 
```java
@Bean("basicListenerContainerFactory")
public ConcurrentKafkaListenerContainerFactory<String, String> simpleKafkaListenerContainerFactory() {
    ConcurrentKafkaListenerContainerFactory<String, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
    factory.setConsumerFactory(this.consumerFactory());
    return factory;
}
```
※ 참고: containerFactory를 spring bean으로 등록하지 않으면 default container factory가 제공됩니다.

<br/>

마지막으로 containerFactory를 `@KafkaListener`에 넘겨서 consumer를 생성해보겠습니다.


어노테이션 파라미터 중 `containerFactory` 값을 basicListenerContainerFactory (containerFactory bean 이름)로 준 것을 확인할 수 있습니다.
이런 방식으로 한번 만든 containerFactory bean은 여러 @KafkaListener에서 지정하여 재사용할 수 있습니다.

KafkaListener 어노테이션에 넘길 수 있는 파라미터를 보면 매우 편리하게 되어있습니다.
아래 예시는 하나의 토픽에 구독하지만 `topicPattern` 같은 파라미터를 사용하면 최소한의 코드로 수많은 consumer를 만들 수 있습니다.
```java
private final String groupId = "test-group-basic-consumer";

@KafkaListener(topics = { "test-topic" }, containerFactory = "basicListenerContainerFactory", groupId = groupId)
public void listen(String message) {
    System.out.println("[" + groupId + "] basic consumer : " + message);
    // handle business
}
```
※ 참고: consumer group에 대해 배워보세요.

<br/>

### 최종 모습
```java
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.springframework.context.annotation.Bean;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory;
import org.springframework.kafka.core.ConsumerFactory;
import org.springframework.kafka.core.DefaultKafkaConsumerFactory;
import org.springframework.stereotype.Component;

import java.util.HashMap;
import java.util.Map;

/**
 * 1강: 이벤트 수신 방법 설명
 */
@Component
public class LessonOneBasicConsumer {

    private final String groupId = "test-group-basic-consumer";

    @KafkaListener(topics = { "test-topic" }, containerFactory = "basicListenerContainerFactory", groupId = groupId)
    public void listen(String message) {
        System.out.println("[" + groupId + "] basic consumer : " + message);
        // handle business
    }

    public ConsumerFactory<String, String> consumerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        return new DefaultKafkaConsumerFactory<>(props);
    }

    @Bean("basicListenerContainerFactory")
    public ConcurrentKafkaListenerContainerFactory<String, String> simpleKafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(this.consumerFactory());
        return factory;
    }
}
```

<br/>

## Test 방법 
### Producer 직접 구현 
Producer Tutorial을 통해 producer를 직접 구현해보고 송신 테스트 하는 방법을 참고하세요.

### Kafka CLI를 활용하여 수신 테스트 방법
$ `kafka-console-producer.sh --broker-list localhost:9092 --topic 토픽명`

<br/>

## 참고 사항
* auto.commit.offset
* consumer group