# Producer Callback

지난 글에서 가장 심플한 spring-kafka Producer를 구현하였습니다.
이번에는 동일한 Producer에 callback을 붙여서 레코드 송신시 결과를 로그로 남기는 방법을 알아보겠습니다.

## 목적
1. 송신 결과(성공/실패)에 따라 특정 custom 코드를 실행

<br/>

## Producer 설정 및 구현
지난 글에서 사용한 Producer와 설정 차이가 없어 동일한 설정으로 진행합니다.
```java
@Bean(name = "simpleProducerKafkaTemplate")
public KafkaTemplate<String, String> kafkaTemplate() {
    // producer configuration
    Map<String, Object> props = new HashMap<>();
    props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "127.0.0.1:9092");
    props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
    props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);

    // producer factory
    DefaultKafkaProducerFactory<String, String> producerFactory = new DefaultKafkaProducerFactory<>(props);
    
    // kafka template
    return new KafkaTemplate<>(producerFactory);
}
```

같은 이유로 동일한 `KafkaTemplate`을 사용합니다.
```java
@Autowired
@Qualifier("simpleProducerKafkaTemplate")
private KafkaTemplate<String, String> kafkaTemplate;
```
※ 참고: 2번째 방법인 `ProducerListener`를 사용하는 경우에는 `KafkaTemplate`에 set을 해주는 부분이 있습니다.

### 코드 설명 
#### 1. Callback
기본적으로 레코드 송신은 비동기 방식으로 동작합니다.
그래서 `ListenableFutureCallback`을 구현해서 callback으로 추가합니다.
Callback을 사용하면 송신할 때 사용하는 send() 메소드를 block 하지 않고 송신 결과를 받을 수 있는 장점이 있습니다.

레코드 송신 실패 시 `onFailure()` 메소드가 실행되며 송신 실패 원인을 알 수 있습니다.
아래 예제에서는 exception의 원인을 로그로 찍어보겠습니다.

송신이 성공적이면 `onSuccess()` 메소드가 실행되며 
송신 성공한 레코드에 대한 기본 정보(metadata)를 출력하도록 구현하였습니다.

`SendResult<K,V>` 클래스를 보면 전송할 레코드인 `ProducerRecord`와 토픽에 담긴 레코드의 기본 정보인 `RecordMetadata`에 대한 접근이 가능한 것을 알 수 있습니다. 
```java
public void sendMessage(String message) {

    // Message<?> also supported
    Message<String> record = MessageBuilder.withPayload(message)
        .setHeader(KafkaHeaders.TOPIC, topic)
        .build();
    
    ListenableFuture<SendResult<String, String>> future = kafkaTemplate.send(record);
    future.addCallback(new ListenableFutureCallback<SendResult<String, String>>() {
        @Override
        public void onFailure(Throwable throwable) {
            System.out.println("send error : " + throwable.getMessage()); // kill broker and let it fail
        }
        
        @Override
        public void onSuccess(SendResult<String, String> result) {
            RecordMetadata recordMetadata = result.getRecordMetadata();
            System.out.println("Sent to : " + recordMetadata.topic() + "-" + recordMetadata.partition() + "-" + recordMetadata.offset());
        }
    });
}
```

#### 2. ProducerListener
송신 결과에 따라 특정 코드를 수행하는 다른 방법은 `ProducerListener`를 제공하는 것 입니다.
Callback 방식과 다른 부분은 `ProducerListener`는 `KafkaTemplate`에 set 해야 합니다.
```java
    @Bean
    public KafkaTemplate<String, String> kafkaTemplate() {
        // producer configuration
        Map<String, Object> props = new HashMap<>();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "127.0.0.1:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);

        // producer factory
        DefaultKafkaProducerFactory<String, String> producerFactory = new DefaultKafkaProducerFactory<>(props);

        // kafka template
        KafkaTemplate<String, String> kafkaTemplate = new KafkaTemplate<>(producerFactory);
        kafkaTemplate.setProducerListener(new ProducerListener<String, String>() {
            @Override
            public void onSuccess(ProducerRecord<String, String> producerRecord, RecordMetadata recordMetadata) {
                System.out.println("Producer Listener - Sent to : " + recordMetadata.topic() + "-" + recordMetadata.partition() + "-" + recordMetadata.offset());
            }

            @Override
            public void onError(ProducerRecord<String, String> producerRecord, Exception exception) {
                System.out.println("Producer Listener - failed to send record due to " + exception.getMessage());
            }
        });
        return kafkaTemplate;
    }
```
위와 같이 설정이 되면 `kafkaTemplate.send(Message<?>)`를 수행하면 송신 성공에 onSuccess() 메소드를 수행하고 
실패에 onError() 메소드를 수행하는 것을 알 수 있습니다.

<br/>

## 최종 모습
### 1. Callback
```java
import org.apache.kafka.clients.producer.RecordMetadata;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.support.KafkaHeaders;
import org.springframework.kafka.support.SendResult;
import org.springframework.messaging.Message;
import org.springframework.messaging.support.MessageBuilder;
import org.springframework.stereotype.Component;
import org.springframework.util.concurrent.ListenableFuture;
import org.springframework.util.concurrent.ListenableFutureCallback;

@Component
public class SimpleProducerCallback {

    @Value("${kafka.topic:sample-topic}")
    private String topic;
    
    @Autowired
    @Qualifier("simpleProducerKafkaTemplate")
    private KafkaTemplate<String, String> kafkaTemplate;

    public void sendMessage(String message) {
    
        // Message<?> also supported
        Message<String> record = MessageBuilder.withPayload(message)
            .setHeader(KafkaHeaders.TOPIC, topic)
            .build();
    
        ListenableFuture<SendResult<String, String>> future = kafkaTemplate.send(record);
        future.addCallback(new ListenableFutureCallback<SendResult<String, String>>() {
            @Override
            public void onFailure(Throwable throwable) {
                System.out.println("send error : " + throwable.getMessage()); // kill broker and let it fail
            }
            
            @Override
            public void onSuccess(SendResult<String, String> result) {
                RecordMetadata recordMetadata = result.getRecordMetadata();
                System.out.println("Sent to : " + recordMetadata.topic() + "-" + recordMetadata.partition() + "-" + recordMetadata.offset());
            }
        });
    }
}
```

### 2. Producer Listener
```java
import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.clients.producer.RecordMetadata;
import org.apache.kafka.common.serialization.StringSerializer;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.kafka.core.DefaultKafkaProducerFactory;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.support.KafkaHeaders;
import org.springframework.kafka.support.ProducerListener;
import org.springframework.messaging.Message;
import org.springframework.messaging.support.MessageBuilder;
import org.springframework.stereotype.Component;

import java.util.HashMap;
import java.util.Map;

@Component
public class SimpleProducerWithCustomProducerListener {

    @Value("${kafka.topic-mp:sample-topic-mp}")
    private String topic;

    @Autowired
    @Qualifier("simpleCustomProducerListenerTemplate")
    private KafkaTemplate<String, String> kafkaTemplate;

    public void sendMessage(String message) {
        Message<String> record = MessageBuilder.withPayload(message)
                .setHeader(KafkaHeaders.TOPIC, topic)
                .build();
        this.kafkaTemplate.send(record);
    }

    @Bean(name = "simpleCustomProducerListenerTemplate")
    public KafkaTemplate<String, String> kafkaTemplate() {
        // producer configuration
        Map<String, Object> props = new HashMap<>();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "127.0.0.1:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);

        // producer factory
        DefaultKafkaProducerFactory<String, String> producerFactory = new DefaultKafkaProducerFactory<>(props);

        // kafka template
        KafkaTemplate<String, String> kafkaTemplate = new KafkaTemplate<>(producerFactory);
        kafkaTemplate.setProducerListener(new ProducerListener<String, String>() {
            @Override
            public void onSuccess(ProducerRecord<String, String> producerRecord, RecordMetadata recordMetadata) {
                System.out.println("Producer Listener - Sent to : " + recordMetadata.topic() + "-" + recordMetadata.partition() + "-" + recordMetadata.offset());
            }

            @Override
            public void onError(ProducerRecord<String, String> producerRecord, Exception exception) {
                System.out.println("Producer Listener - failed to send record due to " + exception.getMessage());
            }
        });
        return kafkaTemplate;
    }
}
```

<br/>

## Test 방법 
### Test용 REST API를 만들어 송신하기 
```java
@RestController
@RequestMapping("producer")
public class ProducerController {

    @Autowired
    private SimpleProducerCallback producerCallback;
    
    @PostMapping("/simple-callback")
    public void sendUsingSimpleProducerCallback(@RequestBody String message) {
        this.producerCallback.sendMessage(message);
    }
}
```
localhost:8080/producer/simple-callback 호출 시 넘겨주는 String 값을 레코드로 만들어 송신합니다.

ProducerListener도 동일한 방식으로 테스트 할 수 있습니다.

<br/>

## 테스트 결과
3건의 레코드를 송신하면 아래와 같은 로그가 찍히는 것을 확인할 수 있습니다.
동일한 토픽으로 레코드를 송신하기 때문에 offset (가장 우측 숫자)가 증가하는 것을 알 수 있습니다.
```
Sent to : test-topic-0-0
Sent to : test-topic-0-1
Sent to : test-topic-0-2
```