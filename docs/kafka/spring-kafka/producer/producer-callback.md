# Producer Callback

지난 글에서 가장 심플한 spring-kafka Producer를 구현하였습니.
이번에는 동일한 Producer에 callback을 붙여서 레코드 송신시 결과를 로그로 남기는 방법을 알아보겠습니다.

## Producer 구현
### Producer 설정
지난 글에서 사용한 Producer와 설정차이가 없기 때문에 동일한 설정으로 진행합니다.
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

기본적으로 레코드 송신은 비동기 방식으로 동작합니다.
그래서 `ListenableFutureCallback`을 구현해서 callback으로 추가합니다.

레코드 송신 실패시 `onFailure()` 메소드가 실행되며 exception을 발생시킨 원인을 알 수 있습니다.
아래 예제에서는 exception의 원인을 로그로 찍어보겠습니다.
송신이 성공적이면 `onSuccess()` 메소드가 실행되며 
송신 성공한 레코드에 대한 기본정보(metadata)를 출력하도록 구현하였습니다.
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
### 최종 모습
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

## Test 방법 
### Test용 REST API를 만들어 송신하기 
REST API를 만들기 위해 `@RestController`를 달아줍니다.
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
localhost:8080/producer/simple-callback 호출시 넘겨주는 String 값을 레코드로 만들어 송신합니다.

## 테스트 결과
3건의 레코드를 송신하면 아래와 같은 로그가 찍히는 것을 확인할 수 있습니다.
동일한 토픽으로 레코드를 송신하기 때문에 offset (가장 우측 숫자)가 증가하는 것을 알 수 있습니다.
```
Sent to : test-topic-0-0
Sent to : test-topic-0-1
Sent to : test-topic-0-2
```