# Simple Producer

가장 심플한 spring-kafka Producer를 구현해보겠습니다.

## Producer 구현
### Producer 설정
기본적으로 spring-kafka는 `KafkaTemplate`를 활용하여 레코드를 송신합니다.

Kafka 설정은 직접 `Map`에 원하는 Kafka 속성과 값을 넣고 `KafkaTemplate`을 생성할 때 활용할 수 있으며,
application 설정(application.yml 혹은 application.properties)을 통해 라이브러리에서 만들어주는 `KafkaTemplate`에 원하는 속성을 반영하는 방법이 있습니다.

물론 설정을 활용하면 소스는 간편해지는 장점이 있지만,
해당 Tutorial에서는 다양 Kafka 설정의 조합으로 여러 개의 `KafkaTemplate`을 만들고 선택적으로 사용할 예정이기 때문에 `Map`에 설정을 넣고 `KafkaTemplate`을 생성해보겠습니다.

중요한 부분은 application.yml을 사용하여 `spring.kafka.producer` 설정을 정의하는 방법도 라이브러리 내부적으로 아래와 같이 동작하기 때문에 내부를 이해하는 데 도움이 될 거라 생각됩니다.
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

### Producer 송신하기
이제 생성한 `KafkaTemplate`을 주입받는 방법입니다.
simpleProducerKafkaTemplate이라는 이름의 bean을 받아오기 위해 `@Qaulifier`를 사용합니다.
해당 Tutorial에서는 `@Qualifer`가 없어도 실행되지만, 향후 Tutorial부터는 1개 이상의 `KafkaTemplate`이 존재하기 때문에 `@Qualifier`를 사용하여 이름으로 지정하여 주입합니다.
```java
@Autowired
@Qualifier("simpleProducerKafkaTemplate")
private KafkaTemplate<String, String> kafkaTemplate;
```

그리고 `KafkaTemplate`을 활용하여 전송하는 코드입니다.
기본적으로 send() 메소드는 비동기 방식으로 동작합니다.
```java
public void sendMessage(String message) {
    kafkaTemplate.send(topic, message);
}
```
### 최종 모습
```java
import java.util.HashMap;
import java.util.Map;

import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.common.serialization.StringSerializer;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.kafka.core.DefaultKafkaProducerFactory;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Component;

@Component
public class SimpleProducer {

    @Value("${kafka.topic:sample-topic}")
    private String topic;
    
    @Autowired
    @Qualifier("simpleProducerKafkaTemplate")
    private KafkaTemplate<String, String> kafkaTemplate;
    
    public void sendMessage(String message) {
        kafkaTemplate.send(topic, message);
    }
    
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
}
```

application.yml에서 kafka.topic를 정의하여 토픽명을 변경할 수 있습니다.
kafka.topic이라는 항목이 없으면 `@Value`에 선언된 것처럼 sample-topic이라는 topic에 송신합니다.
토픽명 설정 예시,
```yml
kafka:
  topic: test-topic
```

## Test 방법 
### Test용 REST API를 만들어 송신하기 
```java
@RestController
@RequestMapping("producer")
public class ProducerController {

    @Autowired
    private SimpleProducer producer;
    
    @PostMapping("/simple")
    public void sendUsingSimpleProducer(@RequestBody String message) {
        this.producer.sendMessage(message);
    }
}
```
localhost:8080/producer/simple 호출시 넘겨주는 String 값을 레코드로 만들어 송신합니다.

### Kafka CLI를 활용하여 수신 테스트 방법
$ `kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic 토픽명`

NOTE
* Consumer Tutorial을 통해 Consumer를 직접 구현해서 테스트하는 방법도 있습니다.


## 참고 사항
* 기본적으로 Application에서는 1개의 `KafkaTemplate`을 만들어서 사용하는 경우가 많습니다.
* 하나 Spring Boot Application에서 여러 종류의 Producer를 만들고 선택해서 사용하고자 `@Bean("빈이름")`을 사용하여 Bean 이름을 지정하였습니다.
* 여러 개의 동일한 bean이 존재하기 때문에 `@Qualifier("빈이름")`으로 원하는 종류의 producer를 사용하였습니다.
* application.yml을 사용하여 Kafka 속성을 정의하는 경우, `KafkaTempqlate` 별도로 생성할 필요가 없습니다.