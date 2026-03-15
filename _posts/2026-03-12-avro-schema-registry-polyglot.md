---
title: "Polyglot 마이크로서비스에 Avro + Schema Registry 도입하기"
description: "Java, NestJS, Python이 혼재하는 마이크로서비스 환경에서 Kafka 이벤트 직렬화를 JSON에서 Avro + Schema Registry로 전환한 과정과 설계 결정 기록"
author: laze
date: 2026-03-12 11:00:00 +0900
categories: [Dev, Kafka]
tags: [Kafka, Avro, SchemaRegistry]
---

## 배경

프로젝트에는 Java/Spring, NestJS, Python/FastAPI 세 가지 런타임이 공존합니다. Kafka로 이벤트를 주고받는데, 기존에는 JSON으로 직렬화하고 있었습니다. JSON의 문제는 스키마 없이 메시지가 오가기 때문에, producer가 필드명을 바꾸거나 타입을 변경하면 consumer가 런타임에 깨진다는 점입니다.

스키마 검증을 CI에서 JSON Schema로 하는 방법(ADR-038)도 검토했지만, 빌드 타임 검증만으로는 런타임 호환성을 보장할 수 없었습니다. 스키마 레지스트리가 필요했습니다.

## Avro vs Protobuf

| 항목 | Avro | Protobuf |
|------|------|----------|
| **Kafka 생태계 통합** | 네이티브 (Confluent Schema Registry 기본 지원) | 별도 설정 필요 |
| **스키마 진화** | 필드 default 값 기반, 동적 해석 가능 | 필드 번호 기반, 엄격한 규칙 |
| **런타임 스키마 관리** | Schema Registry가 버전 관리 + 호환성 검증을 런타임에 수행 | wire compatibility로 하위 호환은 가능하나, 런타임 스키마 검증은 별도 구성 필요 |
| **바이너리 크기** | 작음 (스키마 ID만 페이로드에 포함) | 작음 |
| **디버깅** | Schema Registry + AKHQ로 디코딩 가능 | 별도 도구 필요 |

Kafka 생태계와의 통합도, 스키마 진화의 유연성 모두 Avro가 유리했습니다. Confluent Schema Registry는 Protobuf도 공식 지원하므로 기술적으로 Protobuf를 사용할 수도 있지만, 이 프로젝트에서는 Confluent Stack을 그대로 사용하기 때문에 별도 설정 없이 동작하는 Avro를 선택했습니다.

## 핵심 설계 결정

### 1. 직렬화 방식: Avro Binary

JSON 대신 Avro Binary로 직렬화합니다. 페이로드에는 스키마 ID(5바이트) + 바이너리 데이터만 들어가므로 메시지 크기가 줄어들고, Schema Registry가 스키마 버전 관리와 호환성 검증을 담당합니다.

### 2. 호환성 정책: BACKWARD_TRANSITIVE

```yaml
SCHEMA_REGISTRY_SCHEMA_COMPATIBILITY_LEVEL: BACKWARD_TRANSITIVE
```

새 스키마로 이전 데이터를 읽을 수 있어야 합니다. 필드 추가 시 반드시 default 값을 지정해야 하고, 필드 삭제는 금지됩니다. `TRANSITIVE`이므로 현재 버전뿐 아니라 모든 이전 버전과 호환되어야 합니다.

### 3. 스키마 정의 → 코드 생성

`.avsc` 파일로 스키마를 정의하고, 빌드 시 자동으로 Java 클래스를 생성합니다.

```json
{
  "type": "record",
  "name": "OrderCreatedEvent",
  "namespace": "com.portal.universe.event.shopping",
  "fields": [
    {"name": "orderNumber", "type": "string"},
    {"name": "userId", "type": "string"},
    {"name": "totalAmount", "type": {"type": "bytes", "logicalType": "decimal",
                                      "precision": 18, "scale": 2}},
    {"name": "itemCount", "type": "int"},
    {"name": "items", "type": {"type": "array", "items": "OrderItemInfo"}},
    {"name": "createdAt", "type": {"type": "long", "logicalType": "timestamp-millis"}}
  ]
}
```

`event-contracts` 모듈에서 gradle-avro-plugin으로 코드를 생성합니다.

```groovy
plugins {
    id 'java-library'
    id 'com.github.davidmc24.gradle.plugin.avro' version '1.9.1'
}

dependencies {
    api 'org.apache.avro:avro:1.12.1'
    api 'io.confluent:kafka-avro-serializer:8.1.1'
}

avro {
    stringType = 'String'       // Utf8 대신 String 사용
    fieldVisibility = 'PRIVATE' // getter/setter 생성
}
```

## Java 구현: common-library

모든 Spring 서비스가 공유하는 common-library에 Avro Producer/Consumer 설정을 넣었습니다.

### Producer

```java
@Configuration
@ConditionalOnProperty(name = "spring.kafka.properties.schema.registry.url")
public class AvroProducerConfig {

    @Bean
    public ProducerFactory<String, SpecificRecord> avroProducerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, KafkaAvroSerializer.class);
        props.put(KafkaAvroSerializerConfig.SCHEMA_REGISTRY_URL_CONFIG, schemaRegistryUrl);
        props.put(KafkaAvroSerializerConfig.AUTO_REGISTER_SCHEMAS, true);
        props.put(ProducerConfig.ACKS_CONFIG, "all");
        props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        return new DefaultKafkaProducerFactory<>(props);
    }

    @Bean
    public KafkaTemplate<String, SpecificRecord> avroKafkaTemplate(
            ProducerFactory<String, SpecificRecord> avroProducerFactory) {
        return new KafkaTemplate<>(avroProducerFactory);
    }
}
```

`SpecificRecord` 타입으로 바인딩하면, `.avsc`에서 생성된 클래스를 타입 안전하게 사용할 수 있습니다. `ACKS_CONFIG: all`과 `ENABLE_IDEMPOTENCE`로 메시지 유실과 중복을 방지합니다.

> **주의**: `AUTO_REGISTER_SCHEMAS: true`는 Producer가 새 스키마를 Schema Registry에 자동 등록합니다. 개발 환경에서는 편리하지만, 운영 환경에서는 의도하지 않은 스키마가 등록될 수 있으므로 반드시 `false`로 설정하고 CI/CD 파이프라인에서 스키마를 관리해야 합니다.

### Consumer — DLQ 패턴

Consumer 쪽은 역직렬화 실패를 대비해 `ErrorHandlingDeserializer`로 감싸고, 처리 불가능한 메시지는 DLQ(Dead Letter Queue)로 보냅니다.

```java
@Bean
public ConsumerFactory<String, SpecificRecord> avroConsumerFactory() {
    Map<String, Object> props = new HashMap<>();
    // ErrorHandlingDeserializer로 감싸서 역직렬화 실패 시 DLQ 전달
    props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, ErrorHandlingDeserializer.class);
    props.put(ErrorHandlingDeserializer.VALUE_DESERIALIZER_CLASS, KafkaAvroDeserializer.class);
    props.put(KafkaAvroDeserializerConfig.SPECIFIC_AVRO_READER_CONFIG, true);
    props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
    return new DefaultKafkaConsumerFactory<>(props);
}

@Bean
public CommonErrorHandler avroErrorHandler(ProducerFactory<String, byte[]> dlqProducerFactory) {
    KafkaTemplate<String, byte[]> dlqTemplate = new KafkaTemplate<>(dlqProducerFactory);

    DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(
        dlqTemplate,
        (record, ex) -> new TopicPartition(record.topic() + ".DLT", record.partition())
    );

    FixedBackOff backOff = new FixedBackOff(retryIntervalMs, maxRetryAttempts);
    DefaultErrorHandler errorHandler = new DefaultErrorHandler(recoverer, backOff);

    // 프로그래밍 오류는 재시도해도 의미 없음
    errorHandler.addNotRetryableExceptions(
        IllegalArgumentException.class,
        NullPointerException.class
    );

    return errorHandler;
}
```

`SPECIFIC_AVRO_READER_CONFIG: true`로 설정하면 Consumer가 제네릭 `GenericRecord` 대신 타입이 지정된 `SpecificRecord` 클래스로 역직렬화합니다. `ENABLE_AUTO_COMMIT: false`로 수동 커밋하고, AckMode는 RECORD 단위입니다.

## Polyglot 대응

### NestJS (prism-service)

Java처럼 빌드 타임 코드 생성이 아니라, `@kafkajs/confluent-schema-registry` 패키지로 런타임에 스키마를 해석합니다. Schema Registry 클라이언트가 스키마 ID로 자동 디코딩하므로 consumer를 재배포하지 않아도 되는 것이 장점입니다. KafkaJS의 `eachMessage` 핸들러에서 `registry.decode(message.value)`를 호출하면 바이너리 페이로드가 JavaScript 객체로 변환됩니다. 타입 안전성은 디코딩된 객체에 대해 별도의 TypeScript 인터페이스를 정의해서 확보합니다.

### Python (chatbot-service)

`confluent-kafka`의 `AvroDeserializer`를 사용하여 Schema Registry 연동과 역직렬화를 처리합니다. `fastavro`는 `.avsc` 파일에서 스키마를 직접 로드하는 용도로 함께 사용하며, 디코딩된 메시지는 Python dict로 반환됩니다. Pydantic 모델로 한 번 더 검증해서 타입 안전성을 보장합니다.

## 인프라 구성

Docker Compose에서 Schema Registry와 AKHQ(디버깅 UI)를 함께 띄웁니다.

```yaml
schema-registry:
  image: confluentinc/cp-schema-registry:8.1.1
  ports:
    - "127.0.0.1:18081:8081"
  environment:
    SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: kafka:29092
    SCHEMA_REGISTRY_SCHEMA_COMPATIBILITY_LEVEL: BACKWARD_TRANSITIVE

akhq:
  image: tchiotludo/akhq:0.26.0
  ports:
    - "127.0.0.1:9000:8080"
```

AKHQ는 Schema Registry와 연동하여 Avro 바이너리 메시지를 JSON으로 디코딩하고, 토픽별 메시지 조회, 스키마 버전 이력, Consumer Group 상태를 한 화면에서 확인할 수 있습니다. JSON 시절의 디버깅 편의성을 그대로 유지할 수 있습니다.

## 스키마 규모

프로젝트에서 관리하는 이벤트 스키마의 현황입니다. 도메인별 분포를 통해 어떤 서비스 간 이벤트 통신이 활발한지 파악할 수 있습니다.

현재 42개의 `.avsc` 파일이 8개 도메인에 걸쳐 정의되어 있습니다.

| 도메인 | 스키마 수 | 예시 |
|--------|---------|------|
| seller | 15+ | ProductCreatedEvent, InventoryChangedEvent |
| shopping | 5+ | OrderCreatedEvent, CouponIssuedEvent |
| auth | 1 | UserSignedUpEvent |
| blog | 4 | PostLikedEvent, UserFollowedEvent |
| drive | 3 | FileUploadedEvent, FolderCreatedEvent |
| prism | 2 | PrismTaskCompletedEvent |

## 교훈

**1. 스키마 진화 정책은 처음부터 정하라**

`BACKWARD_TRANSITIVE`를 나중에 설정하면 기존 스키마가 호환성 검증에 실패할 수 있습니다. 프로젝트 시작부터 호환성 레벨을 정하고, 모든 필드에 default 값을 넣는 습관을 들여야 합니다.

**2. Polyglot 환경에서는 런타임 해석 방식이 유연하다**

Java는 코드 생성이 자연스럽지만, TypeScript나 Python에서는 런타임 스키마 해석이 더 실용적입니다. Avro는 두 방식 모두 지원하므로, 언어별로 최적의 접근법을 선택할 수 있습니다.

**3. DLQ는 선택이 아니라 필수다**

Avro 역직렬화 실패, 스키마 불일치 등 처리 불가능한 메시지가 consumer를 블록하지 않도록 DLQ로 격리해야 합니다. `{topic}.DLT` 네이밍 컨벤션으로 원본 토픽과의 관계를 명확히 합니다.
