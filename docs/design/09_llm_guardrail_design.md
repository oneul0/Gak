# 09. LLM 가드레일 및 VOD 동시성 설계

작성일: 2026-05-02

이 문서는 LLM 입출력 검증과 VOD 동시성 제한을 도입한 이유를 설명한다. 분석 경로를 유지보수하는 개발자는 문제와 대안을 비교하고, 입력 제한·타임아웃·슬롯 관리·실패 정책이 어디에 적용되는지 확인할 수 있다. 구현 내용에 이어 메트릭, 현재 한계와 검증 포인트를 살펴본다.

## 1. 배경 및 문제 상황

### 1-1. LLM 입력 무제한

`OllamaAnalyzerService.analyzeBatch()`는 `ChatOptimizer`가 압축한 배치를 그대로 LLM에 전달했다.
배치 크기나 총 입력 문자 수에 상한이 없어 다음 문제가 발생할 수 있었다.

- 배치가 클수록 고정 60초 타임아웃이 부족해 `TimeoutException` → CircuitBreaker 오픈
- 빈 콘텐츠나 극단적으로 긴 채팅이 LLM 입력에 포함되어 응답 품질 저하

### 1-2. LLM 출력 무검증

LLM이 반환한 JSON을 파싱하는 데 성공하면 감정 점수를 검증 없이 사용했다.

- 감정 키 7개 중 일부가 누락돼도 `getOrDefault(emotion, 0.0)` 없이 통과
- 점수가 0~1 범위를 벗어나도 DB에 저장
- 모든 점수가 0인 의미 없는 결과가 NEUTRAL 처리 없이 저장

### 1-3. 데이터 손실이 측정 불가

배치 처리 중 새 배치가 들어오고 `AtomicBoolean.isProcessing`이 `true`이면
`Mono.just(List.of())`를 반환해 채팅을 처리하지 않고 버렸다.
스킵 발생 여부를 추적할 메트릭이 없었다.

### 1-4. VOD 동시 분석 무제한

`VodController.triggerAnalysis()`는 호출 즉시 collector에 크롤 요청을 보냈다.
사용자별 또는 시스템 전체 동시 분석 수 제한이 없어 다음이 우려됐다.

- 여러 사용자가 동시에 분석을 요청하면 Ollama LLM 큐에 요청이 쌓임
- 한 사용자가 먼저 점유한 LLM으로 인해 다른 사용자의 분석 시간이 예측 불가
- collector 메모리와 Kafka lag이 선형적으로 증가

VOD 동시성의 상태 머신과 알려진 한계도 이 문서에서 함께 관리한다.

---

## 2. 고려한 대안

### 입력 가드레일

| 옵션 | 설명 | 선택 여부 |
|---|---|---|
| ChatOptimizer 상한 추가 | 압축 단계에서 크기 제한 | 기각 — ChatOptimizer는 Port & Adapter 구조로 Java/Rust 교체 대상이므로 책임 분리 |
| analyzeBatch 내 가드레일 | LLM 호출 직전에 강제 | **채택** — LLM 경계에서 강제하는 것이 의미 명확 |

### 동시성 제어

| 옵션 | 설명 | 선택 여부 |
|---|---|---|
| AtomicBoolean 유지 + 메트릭만 추가 | 최소 변경 | 기각 — 다중 슬롯 확장이 불가, 스킵 이유 구분 불가 |
| `Semaphore(1)` | 슬롯 1개, 다중 슬롯 확장 가능 | **채택** |
| BlockingQueue | 보류 큐 구현 가능 | 기각 — 현재는 스킵이 맞음. 큐 쌓임 자체가 분석 지연을 의미 |

### VOD 동시성

| 옵션 | 설명 | 선택 여부 |
|---|---|---|
| in-memory ConcurrentHashMap | 단일 인스턴스에서만 동작 | 기각 — 향후 수평 확장 불가 |
| Redis 카운터 + TTL | 분산 환경 지원, stuck 자동 만료 | **채택** |
| DB 기반 상태 조회 | 정확하지만 느림 | 기각 — 요청 진입 시점에서 즉시 판단 필요 |

---

## 3. 최종 결정 및 구현 내용

### 3-1. LLM 입력 가드레일 (`OllamaAnalyzerService`)

**변경 위치:** `applyInputGuardrails()` 신규 메서드, `analyzeBatch()` 호출부

```
빈 채팅 제거 → 배치 크기 상한(MAX_BATCH_SIZE=30) → 총 문자 수 상한(MAX_INPUT_CHARS=3000)
```

- 상한 초과 시 `gak.llm.batch.capped` 카운터 기록
- 입력이 모두 걸러지면 LLM 호출 없이 빈 리스트 반환

### 3-2. 동적 타임아웃 (`computeTimeout()`)

```
timeout = min(90, 20 + batchSize × 1.5) 초
```

기존의 고정 60초 타임아웃을 대체한다. 배치 크기 10이면 35초, 30이면 65초를 적용한다. 상한은 90초다.

### 3-3. LLM 출력 가드레일 (`validateScores()`)

**변경 위치:** `parseOllamaResponse()` 내 scores 적용 시점

- `VALID_EMOTIONS` Set으로 7개 감정 키 완결성 보장
- 각 점수를 `Math.max(0.0, Math.min(1.0, score))`로 클램핑
- 합계 < 0.001이면 NEUTRAL로 교정 + `gak.llm.output.zeroed` 카운터
- 예상 외 키 포함 시 WARN 로그

### 3-4. Semaphore 교체 (`AtomicBoolean` → `Semaphore(1)`)

**변경 위치:** `OllamaAnalyzerService` 필드, `analyzeBatch()` 흐름

```java
// 변경 전
private final AtomicBoolean isProcessing = new AtomicBoolean(false);
if (isProcessing.get()) return Mono.just(List.of()); // 조용한 손실

// 변경 후
private final Semaphore llmSlot = new Semaphore(1);
return Mono.defer(() -> {
    if (!llmSlot.tryAcquire()) {
        recordCount("gak.llm.batch.skipped"); // 관측 가능한 이벤트로 전환
        return Mono.just(List.of());
    }
    return Mono.defer(() -> doAnalyzeBatch(capped))
        .doFinally(ignored -> llmSlot.release());
});
```

- `doFinally`로 성공·실패·취소 모든 경우에서 슬롯 반납 보장
- `Semaphore(N)`으로 확장 시 다중 슬롯 지원 가능

**구독 생명주기 기준 슬롯 관리 보강 (2026-06-15)**

초기 `Semaphore(1)` 구현은 `analyzeBatch()` 메서드가 호출되는 즉시 `tryAcquire()`를 수행했다.
하지만 Reactor의 `Mono`는 구독되어야 실행되므로, 메서드 호출과 실제 작업 시작 사이에 다음 불일치가 있었다.

- 반환된 `Mono`가 구독되지 않으면 슬롯은 획득됐지만 `doFinally`가 실행되지 않아 permit이 반납되지 않는다.
- 동일한 `Mono`를 여러 번 구독하면 최초 획득은 1회인데도 `doFinally`가 여러 번 실행되어 permit 수가 증가할 수 있다.
- Circuit Breaker가 OPEN 상태에서 원본 publisher 구독을 차단하면, 메서드 호출 시 선점한 permit을 반납할 기회가 없다.
- `doAnalyzeBatch()`가 publisher 조립 중 동기 예외를 던지면 `doFinally` 연결 전에 permit이 누수될 수 있다.

따라서 입력 정제, `tryAcquire()`, LLM publisher 생성을 `Mono.defer()` 안으로 이동했다.
이제 슬롯 획득과 반납이 구독마다 1:1로 대응한다.

| 상황 | 변경 전 | 변경 후 |
|---|---|---|
| 구독하지 않은 Mono | permit 선점 및 누수 가능 | permit을 획득하지 않음 |
| 동일 Mono 재구독 | permit 과다 반납 가능 | 구독마다 획득·반납 1회 |
| Circuit Breaker OPEN | 원본 미구독 시 permit 누수 가능 | 원본 미구독이므로 permit 미획득 |
| 취소·오류·정상 완료 | `doFinally` 실행 시 반납 | 동일하게 반납 |
| publisher 조립 중 동기 예외 | `doFinally` 연결 전 누수 가능 | 안쪽 `Mono.defer`가 `onError`로 변환 후 반납 |

검증 테스트:

- `analyzeBatch_DefersSlotAcquisitionUntilSubscription`: 구독 전 LLM 호출 및 permit 선점이 없는지 확인
- `analyzeBatch_BalancesSlotAcrossMultipleSubscriptions`: 동일 Mono를 두 번 구독해도 permit이 1로 유지되는지 확인

### 3-5. VOD 분석 동시성 제한

**신규 파일:** `VodAnalysisSlotService.java`, `VodAnalysisEventConsumer.java`
**변경 파일:** `VodController.java`, `RedisConfig.java`

VOD 동시성 제한은 아래의 Redis 키, 슬롯 생애주기, 장애 정책, 응답 코드로 구성한다.

**Redis 키 구조**

```
vod:active:global          → INCR/DECR, 시스템 전체 카운터 (상한 3)
vod:active:user:{ownerId}  → INCR/DECR, 사용자별 카운터 (상한 1)
vod:owner:{videoNo}        → ownerId 문자열, 슬롯 반납 시 역매핑용
```

모든 키에 TTL 30분을 설정해 종료되지 않은 상태가 자동으로 만료되게 한다.

**슬롯 생애주기**

```
triggerAnalysis() 요청
    └─ slotService.tryAcquire(ownerId, videoNo)
           ├─ REJECTED_USER   → HTTP 429 반환
           ├─ REJECTED_GLOBAL → HTTP 503 반환
           └─ ACQUIRED
                  └─ 분석 파이프라인 시작 (collector → analyzer → core-api)
                         └─ Kafka: vod-analysis-complete-topic
                                └─ VodAnalysisEventConsumer
                                       └─ slotService.releaseByVideoNo(videoNo)
```

**Redis 장애 시 동작 (fail-open 전략)**

`tryAcquire()`는 Redis 오류가 발생하면 `ACQUIRED`를 반환한다. 분석은 허용하지만
슬롯 카운터가 갱신되지 않으므로 일시적으로 제한이 풀릴 수 있다.
Redis 장애로 분석을 차단하는 대신 분석을 계속 허용하도록 결정했다.

**HTTP 응답 코드 선택 근거**

| 상태 | HTTP | 이유 |
|---|---|---|
| REJECTED_USER | 429 Too Many Requests | 사용자가 직접 발생시킨 제한 |
| REJECTED_GLOBAL | 503 Service Unavailable | 시스템 자원 소진, 사용자 귀책 아님 |

---

## 4. 신규 메트릭

**안정성 메트릭** — 시스템이 정상 작동하는지

| 메트릭 이름 | 발생 조건 |
|---|---|
| `gak.llm.batch.skipped` | Semaphore 슬롯이 사용 중이어서 배치를 스킵 |
| `gak.llm.batch.capped` | 입력 가드레일이 배치 크기 또는 문자 수를 줄임 |
| `gak.llm.output.zeroed` | 감정 점수 합계가 0이어서 NEUTRAL로 교정 |

**서비스 적합성 메트릭** — LLM이 요구한 형식으로 답했는지

| 메트릭 이름 | 발생 조건 | 해당 항목 |
|---|---|---|
| `gak.llm.output.empty` | 감정 분석 응답이 빈 문자열 | 질문에 직접 답했는지 |
| `gak.llm.output.parse_failed` | 감정 분석 JSON 파싱 실패 | 오류가 있었는지 |
| `gak.llm.output.message_missing` | 요청한 messageId 결과 누락 | 핵심 내용 빠뜨리지 않았는지 |
| `gak.llm.highlight.parse_failed` | 하이라이트 JSON 파싱 실패 | 오류가 있었는지 |
| `gak.llm.highlight.field_missing` | `is_highlight` · `reasoning` 키 누락 | 핵심 내용 빠뜨리지 않았는지 |

조회 엔드포인트: `GET http://localhost:8082/actuator/metrics/{메트릭명}`

---

## 5. 변경 파일 목록

| 파일 | 변경 유형 | 주요 내용 |
|---|---|---|
| `analyzer/.../OllamaAnalyzerService.java` | 수정 | 입력/출력 가드레일, Semaphore, 동적 타임아웃 |
| `core-api/.../VodAnalysisSlotService.java` | 신규 | Redis 기반 VOD 동시성 슬롯 관리 |
| `core-api/.../VodAnalysisEventConsumer.java` | 신규 | 분석 완료/실패 시 슬롯 반납 Kafka 컨슈머 |
| `core-api/.../VodController.java` | 수정 | triggerAnalysis에 슬롯 가드레일 연결, 429/503 반환 |
| `core-api/.../RedisConfig.java` | 수정 | `ReactiveStringRedisTemplate` 빈 추가 |

---

## 6. 현재 상태

- [x] 사용자별 동시 분석 제한 추가 (`VodAnalysisSlotService.MAX_PER_USER = 1`)
- [x] 시스템 전체 동시 분석 제한 추가 (`VodAnalysisSlotService.MAX_GLOBAL = 3`)
- 실제 인프라 기준 동시 분석 가능량은 아직 측정이 필요하다.
- 제한 초과 요청은 대기열 없이 즉시 거절한다.
- 슬롯 키는 30분 TTL로 비정상 종료 시 자동 만료된다.

---

## 7. 향후 고려사항

- **MAX_GLOBAL 조정**: 현재 보수적으로 3으로 설정했다. Ollama 서버 사양과 실측 데이터를 바탕으로 조정해야 한다.
- **대기열과 QUEUED 상태**: 현재는 제한 초과 요청을 즉시 거절한다. 실제 트래픽에서 대기 요구가 확인될 때만 `REJECTED_*` 대신 대기열에 등록하고 순차 처리하는 방식을 검토한다. 도입 시 프론트엔드 상태 표시도 함께 고려한다.
- **Semaphore 슬롯 수**: 실시간 분석과 VOD 분석이 같은 Ollama 인스턴스를 공유하므로, LLM 부하를 측정한 뒤 조정한다.
- **상태 영속화**: collector의 VOD 상태는 인메모리다. 재기동 빈도가 높아져 조회 보정으로 부족해질 때 Redis 이전을 검토한다.

---

## 8. LLM 클라이언트 추상화

> 관련 ADR: `01_ADR.md` ADR-010

### 배경

가드레일·파싱 로직이 완성된 이후에도 `OllamaAnalyzerService`는 Ollama HTTP API를 직접 호출하고 있었다. `ChatOptimizer`가 ADR-005에서 Port & Adapter로 추상화된 것과 달리, LLM 클라이언트 계층은 동일 패턴이 적용되지 않았다.

### 변경 내용

**신규 파일**

| 파일 | 역할 |
|---|---|
| `analyzer/llm/ChatLlmClient.java` | Port 인터페이스 — `chat(system, user, temp, tokens) → Mono<String>` |
| `analyzer/llm/OllamaChatClient.java` | Ollama Adapter — HTTP 호출, `OllamaRequest` 구성, raw content 반환 |

**`OllamaAnalyzerService` 변경**

| 제거 | 추가 |
|---|---|
| `ollamaApiUrl`, `ollamaModel` (`@Value`) | `ChatLlmClient chatClient` 생성자 주입 |
| `buildOllamaRequest()` | — |
| WebClient LLM 호출 2곳 | `chatClient.chat(...)` 위임 |
| `parseOllamaResponse(OllamaResponse, ...)` | `parseResponse(String content, ...)` |
| `parseHighlightDecision(OllamaResponse)` | `parseHighlightContent(String content)` |

가드레일(입력 정제·출력 검증·Semaphore·동적 타임아웃)은 서비스에 그대로 유지된다.

### 프로바이더 교체 방법

```java
// 새 구현체 추가
@Component
@ConditionalOnProperty(name = "app.llm.provider", havingValue = "openai")
public class OpenAiChatClient implements ChatLlmClient {
    @Override
    public Mono<String> chat(String system, String user, double temp, int maxTokens) {
        // POST /v1/chat/completions
    }
}
```

`application.yaml`에 `app.llm.provider: openai` 설정을 추가한 뒤 재시작한다. `OllamaAnalyzerService`는 변경하지 않는다.

---

## 9. 검증 포인트

### OllamaAnalyzerServiceTest — 9개 전체 통과

| 테스트 | 검증 항목 |
|---|---|
| `analyzeBatch_Success` | LLM 응답 정상 파싱, 감정 점수 매핑 |
| `analyzeBatch_PartialMissingResponse` | 누락 메시지 NEUTRAL 폴백 |
| `analyzeBatch_WithMarkdownCodeBlock` | Markdown 코드 블록 포함 응답 파싱 |
| `analyzeBatch_WithExtraText` | 여분 텍스트 포함 응답에서 JSON 추출 |
| `analyzeBatch_MalformedJson_Fallback` | 파싱 불가 응답 → 전체 NEUTRAL 폴백 |
| `analyzeHighlight_Success` | RAG few-shot 주입 + 하이라이트 판정 정상 파싱, 신호 비율 포맷 포함 |
| `analyzeHighlight_DefaultsAndClamp` | 빈 필드 기본값 채움, intensity 1~10 범위 클램핑 |
| `analyzeHighlight_BlankOrMalformed_Fallback` | 빈 응답·손상 응답 → fallback 결정 반환 |
| `analyzeHighlight_PromptLoadFailure_Fallback` | 프롬프트 로딩 실패 → fallback 반환, Ollama LLM 미호출 확인 |

### VodHighlightAnalyzerTest — 4개 전체 통과

| 테스트 | 검증 항목 |
|---|---|
| `consumeCompletion_NormalizesUnknownEditorialCategory` | LLM이 알 수 없는 카테고리 반환 시 "소통"으로 정규화 |
| `consumeCompletion_ExcludesRejectedHighlights` | LLM 거절(`isHighlight=false`) 후보 제외 → `highlightsCount=0` |
| `consumeCompletion_LlmReviewTimeoutFallsBackToHeuristics` | LLM 타임아웃 → 휴리스틱 스코어로 최소 5개 하이라이트 발행 |
| `consumeCompletion_ComposesSceneLabelForGachaFlex` | 가챠 키워드 + 놀람 신호 조합 → `sceneLabel="비틱"` 자동 결정 |
