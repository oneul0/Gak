# 01. 아키텍처 결정 기록 (ADR)

> 이 문서는 Project Gak(각)의 핵심 아키텍처 결정을 기술적 의사결정 단계에 따라 기록한다.
> 구성: **배경 → 문제 → 고려한 선택지 → 결정 및 근거 → 영향 범위**

프로젝트를 유지보수하는 개발자는 각 결정의 이유와 트레이드오프를 확인하고, 관련 구조를 변경할 때 지켜야 할 조건을 파악할 수 있다. 결정별 기록을 통해 배경과 문제를 먼저 확인한 뒤 선택 근거와 영향 범위를 읽는다.

---

## ADR-001. 실시간 메시지 브로커 선택

**날짜**: 2026-02-27

### 배경

치지직 라이브 채팅은 초당 수십~수백 건 발생할 수 있다. collector가 채팅을 수집한 즉시 analyzer로 넘기되, 같은 방의 채팅은 순서를 보장해야 감정 분석 결과가 의미 있다.

### 문제

- collector → analyzer 간 채팅 전달이 순서 보장 없이 이루어지면 분석 결과가 시간 역전됨
- collector와 analyzer가 직접 연결되면 둘 중 하나가 내려갈 때 데이터가 유실됨

### 고려한 선택지

| 선택지 | 장점 | 단점 |
|--------|------|------|
| RabbitMQ | 설정 간단 | 파티션 기반 순서 보장 없음 |
| Redis Pub/Sub | 빠름 | 구독자 없으면 메시지 소실, 재처리 불가 |
| **Apache Kafka** | 파티션 키로 순서 보장, 리텐션으로 재처리 가능 | 설정 복잡도 높음 |

### 결정 및 근거

**Apache Kafka 채택**. `roomId`를 파티션 키로 사용하면 같은 방의 채팅이 동일 파티션으로 라우팅되어 순서를 보장한다.

### 영향 범위

```
collector/service/ChatProducer.java          ← roomId를 Kafka key로 설정
analyzer/service/ChatAnalysisProcessor.java  ← batchKafkaListenerContainerFactory 소비
core-api/service/ChatStreamService.java      ← analyzed-chat-topic 소비
```

**트레이드오프**: Zookeeper 또는 KRaft 모드 등 별도 인프라 관리 필요.

---

## ADR-002. 비동기/논블로킹 스택 선택

**날짜**: 2026-02-27

### 배경

핵심 병목은 두 가지다. 하나는 Kafka 메시지를 수신해 Ollama LLM에 HTTP 요청을 보낸 뒤 응답을 기다리는 I/O 대기다. 다른 하나는 다수의 프론트엔드 클라이언트에 SSE 이벤트를 전달하기 위한 연결 유지다.

### 문제

스레드 블로킹 방식에서는 LLM 응답 대기 중 스레드가 점유된다. SSE 연결 100개가 스레드 100개를 점유하는 구조라면 부하가 증가할수록 스레드 풀이 소진될 위험이 있다.

### 고려한 선택지

| 선택지 | 특징 |
|--------|------|
| Spring MVC (블로킹) | 친숙하지만 LLM 대기 중 스레드 낭비 |
| **Spring WebFlux (Reactor)** | 비동기 I/O, 소수 스레드로 다수 연결 처리 |
| Virtual Threads (Java 21) | 블로킹 코드 유지 가능, 단 LTS 기준 Java 17 제약 |

### 결정 및 근거

**Spring WebFlux (Reactor) 채택**. LLM 호출 대기·SSE 유지·Kafka 소비 모두 비동기 파이프라인으로 연결해 스레드 효율을 확보했다.

### 영향 범위

```
core-api/   ← WebFlux 기반, R2DBC(비동기 DB 드라이버) 사용
analyzer/   ← Mono<List<EmotionResult>> 반환, 리액티브 체인
collector/  ← WebClient로 외부 API 호출
```

**트레이드오프**: 리액티브 코드는 디버깅이 어렵고 블로킹 라이브러리(JDBC 등) 혼용 시 `Schedulers.boundedElastic()`을 명시해야 한다. 실제로 DB 저장 경로에서 `.subscribeOn(Schedulers.boundedElastic())`을 별도 적용했다.

---

## ADR-003. 외부 LLM 호출 장애 격리

**날짜**: 2026-02-27

### 배경

Ollama는 로컬에서 실행되는 LLM 서버다. 응답 지연·크래시가 발생하면 Kafka consumer가 대기 상태로 쌓이고 SSE 파이프라인 전체가 블로킹될 수 있다.

### 문제

LLM 장애가 채팅 수집·분석 파이프라인 전체를 중단시킨다.

### 결정 및 근거

**Resilience4j Circuit Breaker 채택**. Ollama 연속 실패 시 Circuit Breaker가 `OPEN` 상태로 전환되어 이후 호출은 fallback(`NEUTRAL` 반환)으로 즉시 처리된다. Ollama 복구 후 `HALF_OPEN → CLOSED`로 자동 전환된다.

```java
// OllamaAnalyzerService.java
@CircuitBreaker(name = "geminiApi", fallbackMethod = "fallbackAnalyzeBatch")
public Mono<List<EmotionResult>> analyzeBatch(List<CompressedChat> chats) { ... }

private Mono<List<EmotionResult>> fallbackAnalyzeBatch(List<CompressedChat> chats, Throwable t) {
    return Mono.just(chats.stream()
        .map(c -> EmotionResult.neutral(c.messageId()))
        .toList());
}
```

### 영향 범위

```
analyzer/service/OllamaAnalyzerService.java  ← @CircuitBreaker 적용
analyzer/resources/application.yaml          ← resilience4j.circuitbreaker 설정
```

**트레이드오프**: CB OPEN 구간에는 감정 분석 없이 NEUTRAL만 흐른다. 이 구간에는 VOD 하이라이트 LLM 리뷰도 기능이 저하된 상태로 동작한다.

---

## ADR-004. 실시간 집계와 영속 저장 분리

**날짜**: 2026-02-27

### 배경

분석된 채팅은 두 가지 용도로 쓰인다. 대시보드에 실시간 감정 분포를 표시하기 위해 집계하고, 세션 기록을 남기기 위해 DB에 저장한다.

### 문제

매 채팅 이벤트마다 PostgreSQL에 집계 쿼리(`SUM`, `COUNT`)를 실행하면 쓰기 부하와 읽기 부하가 동일 DB에 집중된다.

### 결정 및 근거

**Redis Hash + PostgreSQL 분리 채택**.

- Redis Hash: `roomId` 기준으로 7가지 감정 점수를 `HINCRBYFLOAT`으로 누적. SSE 이벤트마다 O(1) 조회.
- PostgreSQL: 세션 활성 중 DB에만 저장. 장기 분석·VOD 하이라이트 기록용.

```java
// ChatStreamService.java
streamRedisService.updateMultiEmotionStats(roomId, message.getEmotionScores())  // 실시간 집계
    .then(streamRedisService.getRoomStats(roomId))                                // O(1) 조회
    .subscribe(stats -> sink.tryEmitNext(...));

analyzedChatRepository.save(entity).subscribeOn(Schedulers.boundedElastic());   // 영속 저장
```

### 영향 범위

```
core-api/service/ChatStreamService.java      ← 분기 처리 (Redis 집계 + DB 저장)
core-api/service/StreamRedisService.java     ← Redis Hash 조작
core-api/repository/AnalyzedChatRepository.java ← R2DBC 영속 저장
```

**트레이드오프**: Redis 장애 시 실시간 집계가 중단된다. 인증(fail-secure)과 달리 집계 실패는 서비스 중단이 아니라 통계 누락으로 처리하도록 설계했다.

---

## ADR-005. ChatOptimizer 교체 가능 구조 (Port & Adapter)

**날짜**: 2026-03-04

### 배경

LLM 입력 전처리(중복 제거·압축)를 담당하는 `ChatOptimizer`를 장기적으로 Java에서 Rust/JNI 모듈로 교체할 계획이 있었다.

### 문제

전처리 로직이 비즈니스 코드에 직접 결합되어 있으면 Rust 교체 시 도메인 코드까지 변경해야 한다.

### 고려한 선택지

| 패턴 | 이유 |
|------|------|
| Strategy Pattern | 로직 변형에 적합하나 JNI는 '외부 런타임 경계'를 넘는 행위라 다름 |
| Template Method | 상속 구조가 JNI 경계에 어울리지 않음 |
| **Port & Adapter (Hexagonal)** | JNI를 외부 인프라(Adapter)로 취급 — 아키텍처 의도에 부합 |

### 결정 및 근거

**Port & Adapter 채택**. `ChatOptimizer` 인터페이스(Port)를 정의하고 Java 구현체(`JavaChatOptimizer`)와 Rust JNI 구현체(`RustChatOptimizer`)를 각각 Adapter로 둔다. `application.yaml` 설정만 바꾸면 교체된다.

```java
// ChatOptimizerConfig.java
@Bean
public ChatOptimizer chatOptimizer(@Value("${optimizer.engine:java}") String engine) {
    return "rust".equals(engine) ? new RustChatOptimizer() : new JavaChatOptimizer();
}
```

### 영향 범위

```
analyzer/optimization/ChatOptimizer.java           ← Port (인터페이스)
analyzer/optimization/java/JavaChatOptimizer.java  ← Java Adapter
analyzer/optimization/jni/RustChatOptimizer.java   ← Rust JNI Adapter
analyzer/config/ChatOptimizerConfig.java           ← 조건부 빈 등록
```

**트레이드오프**: JNI 빌드 파이프라인과 네이티브 라이브러리(`.so`/`.dylib`) 관리 비용 발생.

---

## ADR-006. 치지직 API 연동 방식

**날짜**: 2026-03-05

### 배경

치지직 API는 개인 사용자 토큰과 서버 OAuth 클라이언트 토큰을 구분한다. 초기에는 사용자 토큰으로 채팅을 수집하는 방식을 검토했다.

### 문제

사용자마다 토큰을 발급·갱신·저장하면 collector 서비스가 사용자 토큰 저장소 역할까지 담당해야 하고 토큰 만료 처리가 복잡해진다.

### 결정 및 근거

**서버 통합 인증(Client Credentials) 채택**. 단일 서버 OAuth 앱 토큰으로 채팅 데이터를 수집한다. 후원·구독 이벤트는 별도 필터 없이 Kafka passthrough로 전달해 지연을 최소화한다.

### 영향 범위

```
collector/auth/ChzzkAuthService.java      ← OAuth 토큰 관리
collector/auth/ChzzkSessionRegistry.java  ← 세션 저장
collector/config/ChzzkProperties.java     ← CLIENT_ID, CLIENT_SECRET
```

---

## ADR-007. NID WebSocket 직접 연동

**날짜**: 2026-03-14

### 배경

치지직 공식 채팅 API는 일일 호출 제한(10만 건)이 있다. 동시 채널 수가 늘거나 채팅 빈도가 높으면 제한에 도달한다.

### 문제

공식 REST API 폴링은 확장성이 없고 실시간 채팅 흐름을 재현하지 못한다.

### 결정 및 근거

**브라우저 내부 NID WebSocket 프로토콜 직접 연동**. 치지직 클라이언트가 사용하는 WebSocket 엔드포인트를 직접 연결해 실시간 채팅 스트림을 구독한다. API 호출 제한을 우회하며 딜레이 없이 채팅을 수신한다.

### 영향 범위

```
collector/service/NidChatCollector.java   ← WebSocket 연결·메시지 수신
collector/service/ChatProducer.java       ← 2초 배치로 묶어 Kafka 발행
```

**트레이드오프**: 공식 API가 아니므로 치지직 내부 프로토콜 변경 시 연동이 끊길 수 있다. 현재로서는 유일한 실용적 선택지다.

---

## ADR-008. 2초 배치 단위 채팅 수집

**날짜**: 2026-03-14

### 배경

NID WebSocket으로 수신한 채팅을 건당 Kafka에 발행하면 Kafka 메시지 수가 매우 많아지고 LLM이 채팅 1개씩 분석하는 비효율이 생긴다.

### 문제

단건 발행 → LLM 호출이 초당 수십 회 발생 가능 → Ollama 과부하.

### 결정 및 근거

**2초 배치 묶음 발행 채택**. `ChatProducer`가 2초 간격으로 수집된 채팅을 `RawChatBatch`로 묶어 Kafka에 1건으로 발행한다. LLM은 배치 단위로 호출되므로 요청 수를 줄이고 분석 맥락의 품질을 높인다.

```java
// ChatProducer.java — Flux.interval(Duration.ofSeconds(2))로 배치 구성
```

### 영향 범위

```
collector/service/ChatProducer.java          ← 2초 배치 구성 및 Kafka 발행
analyzer/service/ChatAnalysisProcessor.java  ← batchKafkaListenerContainerFactory로 소비
common/dto/RawChatBatch.java                 ← 배치 DTO
```

---

## ADR-009. DTO 공통 모듈화

**날짜**: 2026-03-14

### 배경

`collector`, `analyzer`, `core-api` 세 서비스가 동일한 채팅 DTO를 각자 정의하고 있었다. 필드 하나를 추가하면 세 곳을 모두 수정해야 했다.

### 결정 및 근거

**`backend/common` 모듈 생성**. 모든 서비스 간 공유 DTO(`RawChatBatch`, `RawChatMessage`, `AnalyzedChatMessage` 등)를 common에 통합하고 각 서비스는 `implementation project(':common')`으로 의존한다.

### 영향 범위

```
common/dto/RawChatBatch.java
common/dto/RawChatMessage.java
common/dto/AnalyzedChatMessage.java
```

**트레이드오프**: common을 변경하면 모든 서비스를 다시 빌드해야 한다. 서비스 간 결합을 줄이려면 향후 별도 schema registry(Avro 등)를 고려할 수 있다.

---

## ADR-010. LLM 클라이언트 추상화 계층 도입

**날짜**: 2026-05-18

### 배경

`OllamaAnalyzerService`는 감정 분석 LLM 호출 시 Ollama HTTP API(`/api/chat`)를 WebClient로 직접 호출했다. `ollamaApiUrl`, `ollamaModel`, `buildOllamaRequest()` 등 Ollama 전용 코드가 서비스 로직과 결합되어 있었다.

### 문제

- 프로바이더(Ollama → OpenAI → Claude 등) 교체 시 `OllamaAnalyzerService` 비즈니스 로직 수정 필요
- 입력 가드레일·출력 검증·Semaphore 등 핵심 로직이 HTTP 호출 코드와 섞여 있어 테스트 및 교체 범위가 불명확
- `ChatOptimizer`는 ADR-005에서 Port & Adapter로 추상화했으나, LLM 클라이언트는 동일 패턴을 적용하지 않은 상태

### 고려한 선택지

| 선택지 | 장점 | 단점 |
|--------|------|------|
| 현행 유지 | 변경 없음 | 프로바이더 교체 = 서비스 코드 수정 |
| Spring AI `ChatClient` 도입 | 표준 추상화 | 외부 라이브러리 의존, 현재 WebFlux 구조와 맞지 않음 |
| **`ChatLlmClient` 인터페이스 직접 정의** | 최소 변경, 기존 Reactor 체인 유지 | 직접 유지보수 필요 |

### 결정 및 근거

**`ChatLlmClient` 인터페이스 도입**. `chat(system, user, temperature, numPredict) → Mono<String>` 시그니처로 HTTP 세부 사항을 캡슐화한다. `OllamaChatClient`가 유일한 구현체로 Ollama 전용 코드를 담당하고 `OllamaAnalyzerService`는 인터페이스에만 의존한다.

```java
// ChatLlmClient.java (Port)
public interface ChatLlmClient {
    Mono<String> chat(String systemPrompt, String userPrompt, double temperature, int numPredict);
}

// OllamaChatClient.java (Adapter)
@Component
public class OllamaChatClient implements ChatLlmClient { ... }

// OpenAI 추가 시 — 서비스 코드 무변경
@Component
@ConditionalOnProperty(name = "app.llm.provider", havingValue = "openai")
public class OpenAiChatClient implements ChatLlmClient { ... }
```

### 영향 범위

```
analyzer/llm/ChatLlmClient.java            ← 신규 (Port 인터페이스)
analyzer/llm/OllamaChatClient.java         ← 신규 (Ollama Adapter)
analyzer/service/OllamaAnalyzerService.java ← 수정 (chatClient 주입, HTTP 코드 제거)
```

가드레일(입력 정제·출력 검증·Semaphore·동적 타임아웃)은 `OllamaAnalyzerService`에 그대로 유지된다. 인터페이스 교체 시에도 가드레일은 재사용된다.

**트레이드오프**: 프로바이더별 추가 옵션(JSON mode 강제, tool use 등)이 필요하면 `chat()` 시그니처를 확장하거나 별도 메서드를 추가해야 한다.

---

## ADR-011. Owner 인증을 HMAC 쿠키와 Redis 세션으로 결합

**날짜**: 2026-05-06

### 배경

서명된 쿠키만 사용하는 무상태(stateless) 인증은 로그아웃 후 토큰을 즉시 폐기할 수 없었다. 헤더·쿼리 폴백은 ownerId 위조와 IDOR을 허용했다.

### 결정 및 근거

`GAK_OWNER_ASSERTION`에는 `channelId.sessionId.expiresAt`을 HMAC-SHA256으로 서명하고 요청마다 Redis의 `gak:owner-session:{channelId}`와 sessionId를 비교한다. 로그아웃은 Redis 키를 삭제해 기존 토큰을 즉시 무효화하며 ownerId는 검증된 쿠키와 filter attribute에서만 읽는다.

**트레이드오프**: Redis 장애 시 모든 owner 인증이 401로 실패한다. 데이터 보호가 가용성보다 중요하므로 의도적으로 fail-secure를 선택했다. 동일 채널의 재로그인은 이전 세션을 무효화한다.

---

## ADR-012. 내부 API에 공유 시크릿과 404 응답 적용

**날짜**: 2026-05-06

### 배경

`/internal/**`는 analyzer와 core-api 사이의 RAG 호출용이지만 로컬·개발 환경에서는 백엔드 포트가 노출될 수 있다.

### 결정 및 근거

`InternalAccessFilter`가 `X-Internal-Secret`을 검증하고 불일치 시 403 대신 404를 반환한다. 프로덕션에서는 이 코드 방어와 함께 백엔드 포트를 내부 네트워크에만 둔다.

**트레이드오프**: 공유 시크릿 회전 시 호출·수신 서비스를 함께 재시작해야 한다. 서비스별 인증 체계는 현재 규모에서 운영 복잡도가 더 크므로 도입하지 않았다.

---

## ADR-013. V2 파이프라인을 V1과 분리하고 SSE로 먼저 제공

**날짜**: 2026-05-18

### 배경

실시간 Trust Score·Anchor Chat·EMA Mental Buffer·브리핑 기능을 기존 감정 분석 경로에 섞으면 독립 배포와 장애 격리가 어려워진다.

### 결정 및 근거

V2 코드는 `com.gak.v2`, Kafka 토픽은 `v2-` 접두어로 분리한다. collector는 V1과 V2 raw 이벤트를 함께 발행한다. `V2Aggregator`는 V2 agent의 부분 결과만으로도 프레임을 만든다. 전달 계층은 브라우저 지원과 기존 인프라 재사용을 위해 SSE를 채택했다.

브리핑과 유사 하이라이트 알림은 실패 시 `Mono.empty()`로 끝나는 보조 경로로 두어 기본 `v2_frame` 전송에 영향을 주지 않는다.

**트레이드오프**: 토픽과 DTO가 늘고 V1·V2의 일부 기능이 중복된다. RSocket 전환은 SSE의 측정된 한계가 확인될 때만 검토한다.
