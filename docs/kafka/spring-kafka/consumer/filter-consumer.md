# Consumer Filtering
이번 시간에는 특정 기준에 따라 수신하는 레코드를 처리할지 판단 하는 filtering 기능에 대해서 배워보겠습니다.


## 목적
수신할 레코드의 특정 정보를 기준으로 처리할지 판단 시 사용.

<br/>

## Consumer 설정 및 구현
첫 번째 Consumer 강의에서 언급한 것처럼, Kafka Client에 대한 설정은 consumerFactory에 반영하고,
Consumer들이 실제 수신 처리를 하는 Spring Boot App.에서 customize 하는 부분은 containerFactory에 반영한다고 배웠습니다. 

Filtering 같은 경우는 이번 특정 App.에서 custom하게 해야 하는 일이기 때문에 containerFactory에 `RecordFilterStrategy`를 구현해야 합니다.

간단한 샘플을 위해 수신하는 레코드 payload에 특정 문자열이 존재할 때만 수신을 하겠다라고 한다면 다음과 같이 구현을 할 수 있습니다. 아래 expression이 true인 경우는 @KafkaListener가 지정된 메소드가 수행을 할 것이며, false인 경우는 수행하지 않을 것입니다.

```java
consumerRecord -> consumerRecord.value().contains(filterContent)
```

`ConsumerRecord` 객체를 보시면 payload 뿐만 아니라 기본적인 metadata (topic, partition, offset 등)을 비롯하여 Kafka headers, payload 등 다양한 정보를 제공하고 있습니다.


그럼 containerFactory 전체 소스를 보겠습니다. 아래 부분에서 `setRecordFilterStrategy` 부분을 참고하세요.
```java
@Bean("filteringListenerContainerFactory")
public ConcurrentKafkaListenerContainerFactory<String, String> simpleKafkaListenerContainerFactory() {
    ConcurrentKafkaListenerContainerFactory<String, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
    factory.setConsumerFactory(this.consumerFactory());

    // if the result is true, then the consumer won't process such record
    factory.setRecordFilterStrategy(consumerRecord -> consumerRecord.value().contains(filterContent));

    return factory;
}
```

<br/>

## 최종 모습
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
 * 3강: 이벤트 필터링
 */
@Component
public class LessonThreeFilteringConsumer {

    private final String filterContent = "111";
    private final String groupId = "test-group-filter";

    @KafkaListener(topics = { "test-topic" }, containerFactory = "filteringListenerContainerFactory", groupId = groupId)
    public void listen(String message) {
        System.out.println("[" + groupId + "] filtering consumer : " + message);
        // handle business
    }

    public ConsumerFactory<String, String> consumerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "test-group"); // can be overridden by groupId parameter in @KafkaListener
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        return new DefaultKafkaConsumerFactory<>(props);
    }

    @Bean("filteringListenerContainerFactory")
    public ConcurrentKafkaListenerContainerFactory<String, String> simpleKafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(this.consumerFactory());

        // if the result is true, then the consumer won't process such record
        factory.setRecordFilterStrategy(consumerRecord -> consumerRecord.value().contains(filterContent));

        return factory;
    }
}
```

<br/>

## Test 방법 
기존과 동일하게 kafka-console-producer.sh를 사용하거나 직접 producer를 만들어서 구현하신 조건에 따라 @KafkaListener 메소드가 수행되는지 확인해보세요.

