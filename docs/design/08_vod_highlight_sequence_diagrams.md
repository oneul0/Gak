# 08. VOD 하이라이트 추출 단계별 시퀀스 다이어그램

> 구현 기준: 2026-05-17  
> 대상 파일: `VodController`, `VodAnalysisSlotService`, `VodCollectorController`, `VodChatCrawlerService`, `VodAnalysisStatusService`, `VodHighlightAnalyzer`, `OllamaAnalyzerService`, `VodHighlightConsumer`, `VodTimelinePointConsumer`, `VodAnalysisEventConsumer`, `HighlightEmbeddingService`

이 문서는 VOD 분석 요청이 서비스 사이를 이동하며 하이라이트와 타임라인으로 저장되는 과정을 설명한다. 분석 경로를 유지보수하는 개발자는 각 단계의 담당 서비스, 이벤트 전달과 슬롯 반납 시점을 확인할 수 있다.

전체 흐름은 **분석 요청과 슬롯 획득 → 기존 데이터 초기화 → 채팅 크롤링 → 후보 분석·선별 → 결과 저장과 슬롯 반납 → 결과 조회**의 6단계로 나뉜다. 각 다이어그램을 순서대로 읽고, 마지막 Kafka 토픽 요약에서 서비스 간 연결을 확인한다.

---

## 단계 1 — 분석 요청 및 동시성 가드레일

core-api가 Redis 슬롯을 확인해 시스템 전체와 사용자별 동시 분석 수를 제한한다. 슬롯을 확보한 요청만 다음 단계로 진행한다.

```mermaid
sequenceDiagram
    actor 스트리머
    participant FE as Next.js<br/>(3000)
    participant CA as core-api<br/>(8083)
    participant Redis

    스트리머->>FE: POST /api/v2/vod/{videoNo}/analyze
    FE->>CA: POST /api/v1/vod/{videoNo}/analyze<br/>(X-Gak-Owner-Id 헤더)

    CA->>Redis: INCR vod:active:global
    Redis-->>CA: globalCount

    alt globalCount > 3
        CA->>Redis: DECR vod:active:global
        CA-->>FE: 503 SERVICE_UNAVAILABLE
        FE-->>스트리머: "분석 요청이 많습니다"
    else globalCount ≤ 3
        CA->>Redis: INCR vod:active:user:{ownerId}
        Redis-->>CA: userCount

        alt userCount > 1
            CA->>Redis: DECR vod:active:user:{ownerId}
            CA->>Redis: DECR vod:active:global
            CA-->>FE: 429 TOO_MANY_REQUESTS
            FE-->>스트리머: "이미 분석 중인 VOD가 있습니다"
        else userCount ≤ 1
            CA->>Redis: SET vod:owner:{videoNo} {ownerId} TTL=30m
            CA->>Redis: EXPIRE vod:active:user:{ownerId} 30m
            Note over CA,Redis: 슬롯 획득 성공 (ACQUIRED)
        end
    end
```

---

## 단계 2 — 기존 데이터 초기화 및 크롤링 시작

core-api가 해당 VOD의 기존 하이라이트와 타임라인을 삭제한 뒤 collector에 크롤링을 요청한다. 재분석 결과에 이전 데이터가 섞이지 않도록 초기화하는 단계다.

```mermaid
sequenceDiagram
    participant CA as core-api<br/>(8083)
    participant DB as PostgreSQL
    participant CO as collector<br/>(8081)

    Note over CA: SlotResult.ACQUIRED 이후

    CA->>DB: DELETE vod_highlights WHERE video_no = ?
    CA->>DB: DELETE vod_timeline_points WHERE video_no = ?
    CA->>CO: POST /api/v1/vod/{videoNo}/crawl

    CO-->>CA: 200 OK (즉시 반환, 크롤링은 비동기)
    CA->>DB: UPSERT user_vod_library SET status='ANALYZING'
    CA-->>FE: 200 OK "VOD analysis request sent"
```

---

## 단계 3 — VOD 채팅 전체 크롤링

collector가 Chzzk API에서 cursor를 따라 채팅을 수집하고 Kafka로 청크를 전달한다. collector는 수집 상태를 갱신하고 요청 실패 시 재시도한다. 반복 cursor도 처리한다.

```mermaid
sequenceDiagram
    participant CO as collector<br/>(8081)
    participant Chzzk as Chzzk API
    participant SVC as VodAnalysisStatusService<br/>(in-memory)
    participant Kafka

    Note over CO: VodChatCrawlerService.crawlFullVodChat()

    CO->>SVC: markRequested(videoNo)
    CO->>Chzzk: GET /vod/{videoNo}/metadata
    Chzzk-->>CO: title, duration, category

    loop cursor가 있는 동안 (페이지네이션)
        CO->>SVC: markWaiting(videoNo, pages, chats)
        CO->>Chzzk: GET /vod/{videoNo}/chats?playerMessageTime={cursor}
        Note over CO: timeout=12s, MAX_RETRIES=2

        alt 정상 응답
            Chzzk-->>CO: videoChats 배열, nextPlayerMessageTime
            CO->>Kafka: vod-raw-chat-topic (key=videoNo)
            CO->>SVC: markCrawling(videoNo, pages++, chats+=N)
        else 429 / 503
            CO->>CO: delay(retryCount+1초) 후 재시도
        else MAX_RETRIES 초과
            CO->>SVC: markFailed(videoNo, message)
            Note over CO: 크롤링 중단
        end
    end

    Note over CO: 반복 cursor 감지 시 조기 종료 (visitedCursors 집합)

    CO->>SVC: markAnalyzing(videoNo, pages, chats)
    CO->>Kafka: vod-crawl-complete-topic<br/>(videoNo, title, duration, category, pages, chats)
```

---

## 단계 4 — 하이라이트 분석 및 점수 산정

analyzer가 수집된 채팅을 윈도우별로 채점하고, 상위 후보에 대한 Ollama 리뷰와 시간대 분산 선별을 수행한다. 완료 이벤트 이후에도 채팅이 도착할 수 있다. analyzer는 추가 채팅이 없는 기간을 확인한 뒤 분석을 마무리한다.

```mermaid
sequenceDiagram
    participant Kafka
    participant AZ as analyzer<br/>(8082)<br/>VodHighlightAnalyzer
    participant Ollama

    Kafka-->>AZ: vod-raw-chat-topic (청크마다)
    Note over AZ: VideoAggregate.addChat()<br/>30초 window에 채팅 적재<br/>(synchronized)

    Kafka-->>AZ: vod-crawl-complete-topic
    AZ->>AZ: scheduleFinalize(videoNo)<br/>조용한 기간 1200ms 대기

    loop 조용한 기간 확인 (최대 12회 재시도)
        AZ->>AZ: isQuietFor(1200ms)?
        alt 아직 채팅 도착 중
            AZ->>AZ: 600ms 후 재시도
        else 조용해짐
            Note over AZ: finalize 진행
        end
    end

    Note over AZ: rankWindows() — 각 30초 윈도우 점수 산정

    rect rgb(30, 40, 60)
        Note over AZ: intensityScore 구성 요소
        Note over AZ: densityScore + userScore + burstScore<br/>+ zScoreBoost + emotion 신호들<br/>(laugh/surprise/hype/tension/repetition)
        Note over AZ: transitionScore: 직전 대비 채팅 급증 + 지속 여부
        Note over AZ: editabilityScore: 다양성 + 키워드 집중도 + 대표 채팅 유무
        Note over AZ: totalScore = intensity*0.55 + transition*0.20 + editability*0.25<br/>× edgePenalty × negativePenalty
    end

    AZ->>Ollama: analyzeHighlight() — 상위 12개 후보 (concurrency=3)<br/>timeout=4분
    Ollama-->>AZ: isHighlight, intensity, category, sceneLabel, summary, reasoning

    Note over AZ: LLM 거절 → score×0.38 (hardRejected)<br/>LLM 승인 → (score+2.4)×intensityBoost

    Note over AZ: selectDistributedHighlights()<br/>버킷 분할 (4~8개 구간) → 구간별 대표 1개 먼저 선택<br/>남은 쿼터는 전역 상위로 채움<br/>최종 5~24개 하이라이트
```

---

## 단계 5 — 결과 발행 및 저장

analyzer가 결과와 완료·실패 이벤트를 발행한다. core-api는 타임라인·하이라이트·임베딩을 저장하고 슬롯을 반납하며, collector는 분석 상태를 갱신한다.

```mermaid
sequenceDiagram
    participant AZ as analyzer (8082)
    participant Kafka
    participant CA as core-api (8083)
    participant DB as PostgreSQL
    participant Ollama
    participant CO as collector (8081)

    AZ->>Kafka: vod-window-summary-topic<br/>(모든 WindowStats → VodTimelinePoint)
    AZ->>Kafka: vod-analyzed-topic<br/>(선별된 하이라이트 → VodHighlightPoint)
    AZ->>Kafka: vod-analysis-complete-topic<br/>(timelineCount, highlightCount)

    par vod-window-summary-topic 소비
        Kafka-->>CA: VodTimelinePointConsumer
        CA->>DB: INSERT vod_timeline_points
    and vod-analyzed-topic 소비
        Kafka-->>CA: VodHighlightConsumer
        CA->>DB: INSERT vod_highlights
        CA->>Ollama: POST /api/embed (nomic-embed-text)<br/>ratio 기반 embedding_text → 768차원 벡터
        Ollama-->>CA: float[768]
        CA->>DB: UPDATE vod_highlights SET embedding, embedding_text
    and vod-analysis-complete-topic 소비 (core-api)
        Kafka-->>CA: VodAnalysisEventConsumer
        CA->>Redis: DECR vod:active:user:{ownerId}
        CA->>Redis: DECR vod:active:global
        CA->>Redis: DEL vod:owner:{videoNo}
    and vod-analysis-complete-topic 소비 (collector)
        Kafka-->>CO: VodAnalysisCompletionConsumer
        CO->>CO: VodAnalysisStatusService.markCompleted()
    end

    alt 분석 실패 시
        AZ->>Kafka: vod-analysis-failed-topic
        Kafka-->>CA: VodAnalysisEventConsumer.onAnalysisFailed()
        CA->>Redis: 슬롯 반납
        Kafka-->>CO: VodAnalysisFailureConsumer
        CO->>CO: markFailed(videoNo, reason)
    end
```

---

## 단계 6 — 결과 조회 (개인화 포함)

frontend가 collector에서 진행 상태를, core-api에서 타임라인과 하이라이트를 조회한다. core-api는 데이터가 비어 있으면 타임라인 fallback을 처리한다. 하이라이트는 사용자 활동에 따라 개인화하여 정렬한다.

```mermaid
sequenceDiagram
    actor 스트리머
    participant FE as Next.js (3000)
    participant CA as core-api (8083)
    participant CO as collector (8081)
    participant DB as PostgreSQL

    스트리머->>FE: VOD 분석 상태 폴링
    FE->>CO: GET /api/v1/vod/{videoNo}/status
    CO-->>FE: {status, pagesProcessed, chatsCollected, message}

    Note over CO: status=ANALYZING이 30분 초과 시<br/>core-api에 highlights 조회 후 강제 COMPLETED

    스트리머->>FE: 타임라인 조회
    FE->>CA: GET /api/v1/vod/{videoNo}/timeline
    CA->>DB: SELECT vod_timeline_points WHERE video_no = ?

    alt timeline 데이터 있음
        DB-->>CA: VodTimelinePointEntity[]
        CA-->>FE: 정상 타임라인
    else timeline 비어있음 (fallback)
        CA->>DB: SELECT vod_highlights WHERE video_no = ?
        DB-->>CA: VodHighlight[]
        CA->>CA: toFallbackTimelinePoint() 변환
        CA-->>FE: highlights 기반 타임라인
    end

    스트리머->>FE: 하이라이트 조회
    FE->>CA: GET /api/v1/vod/{videoNo}/highlights
    CA->>DB: SELECT vod_highlights, user_vod_activity (ownerId)
    CA->>DB: SELECT user_vod_activity 전체 (선호 프로필 구성)
    DB-->>CA: highlights + activities

    Note over CA: personalizedScore 계산<br/>PIN(+120) / GOOD(+48) / OPEN(+6) / BAD(-72)<br/>+ 카테고리 친화도×2 + 반응 친화도×1.5<br/>+ editability×1.8 + transition×1.3 + intensity×0.9

    CA->>DB: UPSERT user_vod_library SET status, last_viewed_at
    CA-->>FE: 개인화 정렬된 VodHighlight[]
    FE-->>스트리머: 하이라이트 카드 표시
```

---

## Kafka 토픽 흐름 요약

```
collector ──[vod-raw-chat-topic]──────────────► analyzer
collector ──[vod-crawl-complete-topic]────────► analyzer
analyzer  ──[vod-window-summary-topic]────────► core-api (VodTimelinePointConsumer)
analyzer  ──[vod-analyzed-topic]──────────────► core-api (VodHighlightConsumer)
analyzer  ──[vod-analysis-complete-topic]─────► core-api (VodAnalysisEventConsumer)
                                          └───► collector (VodAnalysisCompletionConsumer)
analyzer  ──[vod-analysis-failed-topic]──────► core-api (VodAnalysisEventConsumer)
                                         └───► collector (VodAnalysisFailureConsumer)
```

| 토픽 | 파티션 키 | 목적 |
|------|-----------|------|
| `vod-raw-chat-topic` | videoNo | 크롤링된 채팅 청크 전달 |
| `vod-crawl-complete-topic` | videoNo | 크롤링 완료 신호 |
| `vod-window-summary-topic` | videoNo | 타임라인 포인트 저장 |
| `vod-analyzed-topic` | videoNo | 하이라이트 후보 저장 |
| `vod-analysis-complete-topic` | videoNo | 슬롯 반납 + 상태 갱신 |
| `vod-analysis-failed-topic` | videoNo | 실패 처리 + 슬롯 반납 |
