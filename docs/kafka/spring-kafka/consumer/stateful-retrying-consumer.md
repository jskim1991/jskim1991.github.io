# Error Handling & Retry [2]
이번에는 Error Handling & Retry 두 번째 글을 만나보겠습니다.
지난 시간에 배운 stateless retry와 다른 stateful retry에 대해 배워보겠습니다.

## 목적
spring-kafka를 활용하여 stateful retry를 구현.

<br/>

## Consumer 설정 및 구현
지난 시간에 Kafka broker는 consumer의 stateless retry에 대해 모른다고 설명해 드렸습니다.
Stateful retry는 consumer가 poll을 수행한다는 점에서 stateless retry와 차이점이 있습니다.

Consumer가 retry를 하면서 `poll`을 수행하지 않으면
Kafka broker는 해당 consumer가 inactive하다고 판단하여
현재 consumer 목록에서 제외하고 다른 active consumer에게 partition을 다시 할당합니다.
이것을 `rebalancing`이라고 하며 이 부분은 consumer의 가장 핵심적인 요소 중 하나입니다.

spring-kafka는 `SeekToCurrentErrorHandler`를 통해 stateful retry를 제공합니다.

※ 참고: Kafka Transaction 경우는 stateful error handler로 `DefaultAfterRollbackProcessor`을 사용합니다.

그럼 코드를 한번 보겠습니다. RetryTemplate과 비슷하게 BackOff 구현체인 `FixedBackOff`나 `ExponentialBackOff`를 생성하여 SeekToCurrentErrorHandler에 넘겨주면 됩니다.
```java
FixedBackOff fixedBackOff = new FixedBackOff();
fixedBackOff.setInterval(1000l);
fixedBackOff.setMaxAttempts(3);

SeekToCurrentErrorHandler errorHandler = new SeekToCurrentErrorHandler(fixedBackOff);
```

만약 수신 오류가 발생한 레코드와 원인에 따라 custom한 코드를 수행하고 싶다면 SeekToCurrentErrorHandler 생성자의 첫 번째 파라미터에 FunctionalInterface를 구현할 수 있습니다.
예시로 처리 실패한 레코드와 원인을 console에 찍어보겠습니다.
```java
@Bean("simpleStatefulRetryingListenerContainerFactory")
public ConcurrentKafkaListenerContainerFactory<String, String> simpleKafkaListenerContainerFactory() {
    ConcurrentKafkaListenerContainerFactory<String, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
    factory.setConsumerFactory(this.consumerFactory());

    /* Retries using SeekToCurrentErrorHandler */
    FixedBackOff fixedBackOff = new FixedBackOff();
    fixedBackOff.setInterval(1000l);
    fixedBackOff.setMaxAttempts(3);

    SeekToCurrentErrorHandler errorHandler = new SeekToCurrentErrorHandler((consumerRecord, e) -> {
        System.out.println("failed record: " + consumerRecord.toString() + " , reason: " + e.getMessage());
        // customize what error handler should do here
    }, fixedBackOff);
    factory.setErrorHandler(errorHandler);
    factory.setStatefulRetry(true);

    return factory;
}
```

이번에도 동일하게 강제로 exception을 주는 consumer로 테스트해보겠습니다. spring-retry와 동일하게 `maxAttempts`를 3으로 지정했지만 총 4번 수신합니다 (최초 1번 실행 + 3번 retry )
```java
@KafkaListener(topics = { "test-topic-stateful-retry" }, containerFactory = "simpleStatefulRetryingListenerContainerFactory", groupId = groupId)
public void listen(String message) {
    System.out.println("[" + groupId + "] simple stateful retrying consumer : " + message);
    // handle business
    throw new RuntimeException("something bad happened");
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
import org.springframework.kafka.listener.SeekToCurrentErrorHandler;
import org.springframework.stereotype.Component;
import org.springframework.util.backoff.FixedBackOff;

import java.util.HashMap;
import java.util.Map;

/**
 * 5강: Consumer stateful retry
 */
@Component
public class LessonFiveStatefulRetryingConsumer {

    private final String groupId = "test-group-stateful-retry";

    @KafkaListener(topics = { "test-topic-stateful-retry" }, containerFactory = "simpleStatefulRetryingListenerContainerFactory", groupId = groupId)
    public void listen(String message) {
        System.out.println("[" + groupId + "] simple stateful retrying consumer : " + message);
        // handle business
        throw new RuntimeException("something bad happened");
    }

    public ConsumerFactory<String, String> consumerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        return new DefaultKafkaConsumerFactory<>(props);
    }

    @Bean("simpleStatefulRetryingListenerContainerFactory")
    public ConcurrentKafkaListenerContainerFactory<String, String> simpleKafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(this.consumerFactory());

        /* Retries using SeekToCurrentErrorHandler */
        FixedBackOff fixedBackOff = new FixedBackOff();
        fixedBackOff.setInterval(1000l);
        fixedBackOff.setMaxAttempts(3);

        SeekToCurrentErrorHandler errorHandler = new SeekToCurrentErrorHandler((consumerRecord, e) -> {
            System.out.println("failed record: " + consumerRecord.toString() + " , reason: " + e.getMessage());
            // customize what error handler should do here
        }, fixedBackOff);
        factory.setErrorHandler(errorHandler);
        factory.setStatefulRetry(true);

        return factory;
    }
}
```

<br/>

## 참고 사항
* rebalancing
* session.timeout.ms
* max.poll.interval.ms
* heartbeat.interval.ms