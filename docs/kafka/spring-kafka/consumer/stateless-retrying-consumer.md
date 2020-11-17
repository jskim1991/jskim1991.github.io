# Error Handling & Retry [1]
이번 시간에는 Error Handling & Retry 첫 번째 글입니다.
Consumer가 레코드를 처리하다 비즈니스나 시스템에 의해 오류가 발생할 수 있습니다.
그럼 처리가 되지 않은 레코드는 어떤 방법으로 핸들링해야 할까요?

## 목적
`spring-retry`에서 제공하는 `RetryTemplate`을 활용하여 stateless retry를 구현.

<br/>

## Consumer 설정 및 구현
spring-kafka 같은 경우는 spring-retry와 호환이 잘 되어 있어 RetryTemplate만 생성해서 사용하면 나머지는 Framework에서 모두 마법처럼 동작합니다.
그럼 바로 코드로 먼저 보겠습니다.

아래 예시 같은 경우는 `FixedBackOffPolicy`와 `SimpleRetryPolicy`를 만들어서 RetryTemplate에 사용한 것을 볼 수 있습니다.
BackOff라는 것은 오류가 발생하고 잠시 뒤로 물러났다가 다시 도전한다는 개념입니다. BackOffPeriod가 1000ms 이면 오류가 발생하고 1초 후에 retry를 할 것을 의미합니다.
RetryPolicy 같은 경우는 최대 몇 번을 retry 할지 설정할 수 있습니다. 아래 예시는 maxAttempts가 3으로 지정되어 있어 최초 1번 실행 + retry 2번을 시도합니다.

```java
@Bean("retryingListenerContainerFactory")
public ConcurrentKafkaListenerContainerFactory<String, String> simpleKafkaListenerContainerFactory() {
    ConcurrentKafkaListenerContainerFactory<String, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
    factory.setConsumerFactory(this.consumerFactory());

    /* Retries using RetryTemplate */
    RetryTemplate retryTemplate = new RetryTemplate();
    // set how many milliseconds next try should be started
    FixedBackOffPolicy backOffPolicy = new FixedBackOffPolicy(); // or ExponentialBackOffPolicy can be used
    backOffPolicy.setBackOffPeriod(1000l);
    retryTemplate.setBackOffPolicy(backOffPolicy);

    // set maximum attempts
    SimpleRetryPolicy retryPolicy = new SimpleRetryPolicy();
    retryPolicy.setMaxAttempts(3);
    retryTemplate.setRetryPolicy(retryPolicy);

    factory.setRetryTemplate(retryTemplate);

    return factory;
}
```

그럼 KafkaListener에서 강제로 exception을 발생시켜 위 설정값들이 어떤 영향이 있는지 한번 확인해보세요.
```java
@KafkaListener(topics = { "test-topic-retry" }, containerFactory = "retryingListenerContainerFactory", groupId = groupId)
public void listen(String message) {
    System.out.println("[" + groupId + "] simple retrying consumer : " + message);
    // handle business
    throw new RuntimeException("something bad happened");
}
```

<br/>

여기서 중요한 부분은 stateless retry라는 부분입니다.
단순히 RetryTemplate을 봤을 때 maxAttempts 값으로 몇 번 시도하고 있는지 알고 있는데 왜 stateless라고 할까요?
Stateless 라는 부분은 Kafka broker와 consumer 관계에서 의미하는 부분으로 consumer가 일시적인 오류로 인해 retry하고 있다는 것을 Kafka broker는 모른다는 것 입니다.

Kafka broker가 모르면 어떻게 되나요? Broker가 왜 알아야 하나요?
그럼 무조건 stateful한게 좋을까요?
이 질문들은 다음 시간에 알아보도록 하겠습니다.

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
import org.springframework.retry.backoff.FixedBackOffPolicy;
import org.springframework.retry.policy.SimpleRetryPolicy;
import org.springframework.retry.support.RetryTemplate;
import org.springframework.stereotype.Component;

import java.util.HashMap;
import java.util.Map;

/**
 * 4강: 에러 핸들링 & RetryTemplate을 사용한 Consumer retry
 */
@Component
public class LessonFourRetryingConsumer {

    private final String groupId = "test-group-retry";

    @KafkaListener(topics = { "test-topic-retry" }, containerFactory = "retryingListenerContainerFactory", groupId = groupId)
    public void listen(String message) {
        System.out.println("[" + groupId + "] simple retrying consumer : " + message);
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

    @Bean("retryingListenerContainerFactory")
    public ConcurrentKafkaListenerContainerFactory<String, String> simpleKafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(this.consumerFactory());

        /* Retries using RetryTemplate */
        RetryTemplate retryTemplate = new RetryTemplate();
        // set how many milliseconds next try should be started
        FixedBackOffPolicy backOffPolicy = new FixedBackOffPolicy(); // or ExponentialBackOffPolicy can be used
        backOffPolicy.setBackOffPeriod(1000l);
        retryTemplate.setBackOffPolicy(backOffPolicy);

        // set maximum attempts
        SimpleRetryPolicy retryPolicy = new SimpleRetryPolicy();
        retryPolicy.setMaxAttempts(3);
        retryTemplate.setRetryPolicy(retryPolicy);

        factory.setRetryTemplate(retryTemplate);

        return factory;
    }
}

```

<br/>

## 참고 사항
* rebalancing
* spring-retry