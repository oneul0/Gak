# 02. 온보딩 가이드

> 프로젝트를 처음 보는 개발자가 서비스 구성, 주요 데이터 흐름, 코드 위치를 빠르게 파악하기 위한 문서입니다.  
> 세부 설계와 구현 근거는 각 항목의 링크 문서를 참고하세요.

---

## 1. 프로젝트 한 줄 요약

**각(Gak)**은 치지직 스트리머를 위한 Owner 전용 대시보드입니다.

실시간 채팅을 수집하고 투표·룰렛을 운영하며, VOD 채팅을 분석해 편집 후보 구간을 자동으로 추출합니다.

---

## 2. 서비스 구성

| 서비스 | 포트 | 역할 |
|---|---:|---|
| `frontend` | 3000 | Next.js UI 및 API Proxy |
| `collector` | 8081 | CHZZK OAuth, 라이브 채팅 수집, VOD 크롤링 |
| `analyzer` | 8082 | 채팅 감정 분석, VOD 하이라이트 계산 |
| `core-api` | 8083 | 분석 결과 저장·조회, SSE, 접근 제어 |

브라우저는 백엔드 서비스를 직접 호출하지 않습니다.

모든 요청은 Next.js API Route를 통해 전달합니다.

```text
/api/chzzk/* → collector
/api/v1/*    → core-api
/api/v2/*    → core-api
```

---

## 3. 필수 인프라

Docker Compose로 다음 인프라를 실행합니다.

| 구성 요소 | 포트 | 용도 |
|---|---:|---|
| PostgreSQL 15 + pgvector | 5432 | 서비스 데이터 및 임베딩 저장 |
| Redis 7 | 6379 | 세션, 실시간 상태, VOD 분석 슬롯 |
| Apache Kafka + Zookeeper | 9092 | 서비스 간 이벤트 전달 |
| Ollama | 11434 | 감정 분석 LLM 및 임베딩 생성 |

로컬 실행 방법은 [`03_run_guide.md`](03_run_guide.md)를 참고하세요.

---

## 4. 전체 요청 흐름

```text
Browser
 └─ Next.js (3000)
      ├─ /api/chzzk/* → collector (8081)
      │                   OAuth · 채팅 수집 · VOD 크롤링
      │
      ├─ /api/v1/*    → core-api (8083)
      │                   분석 결과 · SSE · 투표 · VOD
      │
      └─ /api/v2/*    → core-api (8083)
                          V2 실시간 분석 · SSE

collector ──Kafka──▶ analyzer
analyzer  ──Kafka──▶ core-api

core-api ─────────▶ PostgreSQL
core-api ─────────▶ Redis
```

서비스 간 데이터 전달은 Kafka를 중심으로 구성합니다.

---

## 5. 인증 구조

### 로그인 흐름

```text
1. 스트리머가 "치지직으로 로그인" 클릭
2. frontend → GET /api/chzzk/login → collector
3. collector가 CHZZK OAuth URL 생성
4. Redis에 OAuth state 저장 (TTL 10분)
5. 브라우저를 CHZZK 로그인 페이지로 리다이렉트
6. CHZZK → collector /callback?code=&state=
7. collector가 code를 token으로 교환
8. token으로 channelId 조회
9. HMAC-SHA256으로 서명한 GAK_OWNER_ASSERTION 쿠키 발급
10. Redis에 gak:owner-session:{channelId} 저장
11. /channels/{channelId}로 리다이렉트
```

### core-api 요청 검증

Owner 요청은 다음 순서로 검증합니다.

```text
GAK_OWNER_ASSERTION 쿠키
        ↓
HMAC 서명 검증
        ↓ 실패
       401

        ↓ 성공
Redis gak:owner-session:{ownerId} 조회
        ↓ 없음 또는 불일치
       401

        ↓ 성공
URL channelId == ownerId 확인
        ↓ 불일치
       403
```

### 보안 처리

| 대상 | 처리 |
|---|---|
| JavaScript를 통한 쿠키 접근 | `HttpOnly=true` |
| HTTPS 환경의 쿠키 전송 | 프로덕션에서 `Secure=true` |
| 로그아웃 후 기존 assertion 재사용 | Redis 세션 삭제 후 요청 거부 |
| 내부 API 직접 접근 | `InternalAccessFilter`가 검증 실패 시 404 반환 |
| 다른 channelId 접근 | 쿠키·Redis 세션·URL `channelId` 일치 여부 검증 |

상세 설계:

- [`01_ADR.md`](01_ADR.md)
- [`11_system_reliability.md`](11_system_reliability.md)

---

## 6. 실시간 채팅 처리

투표, 룰렛, 실시간 채팅 분석은 동일한 채팅 수집 경로에서 시작합니다.

메시지 종류에 따라 analyzer에서 처리 경로가 달라집니다.

```text
CHZZK NID WebSocket
        ↓
collector
채팅 수집 · 2초 배치
        ↓
Kafka
raw-chat-batch-topic
key = roomId
        ↓
analyzer
ChatAnalysisProcessor
        │
        ├─ DONATION
        │    └─ 그대로 전달 → 룰렛 트리거
        │
        ├─ SUBSCRIPTION
        │    └─ 그대로 전달 → 알림
        │
        ├─ VOTE (!투표 N)
        │    └─ Redis 투표 집계
        │
        └─ CHAT
             ↓
        HeuristicSentimentAnalyzer
             │
             ├─ 명확한 채팅
             │    └─ Fast-Path
             │         → analyzed-chat-topic
             │
             └─ 모호한 채팅
                  └─ OllamaAnalyzerService
                       ├─ 입력 제한
                       ├─ 7개 감정 분석
                       └─ 출력 검증
                            ↓
                     analyzed-chat-topic

        ↓
core-api
ChatStreamService
        ↓
DB 저장 + SSE
        ↓
frontend
```

Kafka 메시지는 `roomId`를 key로 사용합니다. 같은 방의 메시지가 같은 파티션으로 전달되므로 해당 파티션 안에서 순서를 유지합니다.

### 감정 분석의 현재 사용 범위

실시간 `CHAT` 메시지도 Fast-Path 또는 Slow-Path를 통해 감정 점수를 생성합니다.

다만 `chat_analyzed`, `stats_update` 결과는 현재 프론트엔드에서 소비하지 않습니다.

`OllamaAnalyzerService`가 실제 사용자 기능에 직접 영향을 주는 주요 경로는 **VOD 하이라이트 후보 리뷰**입니다.

### SSE replay

`ChatStreamService`는 `Sinks.replay(100)`을 사용합니다.

신규 구독자가 연결되면 최근 최대 100개 이벤트를 다시 전달합니다.

---

## 7. VOD 하이라이트 분석

전체 시퀀스는 [`08_vod_highlight_sequence_diagrams.md`](08_vod_highlight_sequence_diagrams.md)를 참고하세요.

### 7-1. VOD 채팅 수집

`collector`가 cursor 기반 페이지네이션으로 전체 VOD 채팅을 수집합니다.

- `visitedCursors`로 동일 cursor 반복 방지
- 요청 timeout: 12초
- 최대 재시도: 2회

### 7-2. 30초 단위 후보 점수 계산

수집한 채팅을 30초 윈도우로 나눕니다.

각 윈도우에는 세 종류의 점수를 계산합니다.

| 점수 | 주요 기준 |
|---|---|
| `intensityScore` | 채팅 밀도, 고유 발화자, burst, Z-score, 감정 토큰 |
| `transitionScore` | 이전 구간 대비 반응 증가 및 이후 지속 여부 |
| `editabilityScore` | 메시지 다양성, 키워드 집중도, 대표 채팅 |

```text
totalScore =
    intensityScore × 0.55
  + transitionScore × 0.20
  + editabilityScore × 0.25
```

### 7-3. LLM 후보 리뷰

휴리스틱 점수 상위 12개 구간을 Ollama가 다시 검토합니다.

- 최대 동시 리뷰: 3건
- 리뷰 제한 시간: 4분
- LLM 거절 후보: `score × 0.38`

LLM은 전체 VOD를 분석하지 않고 **휴리스틱으로 선별한 후보의 맥락을 검토하는 단계**에서 사용합니다.

### 7-4. 시간대 분산

높은 점수만 순서대로 선택하면 특정 시간대에 후보가 집중될 수 있습니다.

이를 줄이기 위해 VOD를 4~8개 시간 버킷으로 나눕니다.

1. 각 버킷에서 대표 후보 1개를 우선 선택
2. 남은 개수는 전체 후보 중 높은 점수부터 선택
3. 최종 5~24개 후보 생성

### 7-5. VOD 분석 동시성

Redis에서 분석 슬롯을 관리합니다.

- 사용자별 최대 1건
- 전체 최대 3건
- 슬롯 TTL 30분

Redis에서 슬롯 상태를 확인하지 못한 경우 VOD 분석은 `fail-open`으로 처리합니다.

반면 인증 세션 확인에 실패하면 요청을 거부하는 `fail-secure` 정책을 사용합니다.

---

## 8. 데이터베이스

ERD와 테이블 설계는 [`07_erd.md`](07_erd.md)를 참고하세요.

| 테이블 | 용도 |
|---|---|
| `analyzed_chats` | 실시간 채팅 감정 분석 결과 |
| `highlight_records` | 라이브 방송 하이라이트 순간 |
| `vod_highlights` | VOD 편집 후보 구간 |
| `vod_timeline_points` | VOD 시간대별 활동 집계 |
| `user_vod_library` | 스트리머 VOD 목록 |
| `user_vod_activity` | VOD 하이라이트 상호작용 기록 |

스키마 변경은 Flyway로 관리합니다.

```text
V1__*.sql
...
V9__*.sql
```

DB 계정은 역할에 따라 분리합니다.

| 계정 | 권한 |
|---|---|
| `gak_admin` | DDL 및 Flyway migration |
| `gak_app` | 애플리케이션 DML |

---

## 9. LLM 처리 경계

`OllamaAnalyzerService`는 다음 두 경로에서 호출됩니다.

1. 실시간 채팅 Slow-Path
2. VOD 하이라이트 LLM 리뷰

실시간 감정 분석 결과는 현재 UI에 직접 노출되지 않으며, VOD 하이라이트 리뷰가 주요 사용자 기능 경로입니다.

LLM 호출 전후에는 다음 검증을 적용합니다.

```text
입력
 ├─ 빈 채팅 제거
 ├─ 최대 30개
 └─ 최대 3,000자

실행
 └─ Semaphore(1)

출력
 ├─ 감정 키 7개 존재 확인
 ├─ 점수 [0.0, 1.0] 범위 보정
 └─ 전체 합 < 0.001 → NEUTRAL
```

LLM 응답의 형식, 처리 시간, 성공 여부는 애플리케이션에서 직접 통제할 수 없습니다.

따라서 LLM 호출을 하나의 외부 경계로 두고 입력 크기, 동시 실행, 출력 형식, 실패 결과를 애플리케이션에서 별도로 통제합니다.

상세 내용은 [`09_llm_guardrail_design.md`](09_llm_guardrail_design.md)를 참고하세요.

---

## 10. 트러블슈팅 빠른 참조

| 증상 | 확인할 원인 | 처리 |
|---|---|---|
| SSE 신규 구독 시 이전 이벤트를 받지 못함 | `Sinks.multicast()` 사용 여부 | `Sinks.replay(100)` 사용 |
| core-api DB 인증 실패 | 애플리케이션과 DB 계정 불일치 | `application-dev.yaml` 확인 |
| Docker DB 계정이 변경되지 않음 | 기존 volume에 이전 계정 정보가 남아 있음 | volume 재생성이 필요한지 확인 |
| VOD 상태가 `ANALYZING`에 머무름 | Kafka consumer 또는 완료 이벤트 처리 중단 | consumer 상태 확인 후 COMPLETED 보정 경로 확인 |
| 브라우저에서 preflight 요청 실패 | `OwnerAccessFilter`가 직접 요청 차단 | Next.js API Proxy 경유 여부 확인 |
| 하이라이트가 VOD 앞부분에 집중됨 | 초반 채팅 밀도가 높은 점수 차지 | 버킷 분산 선별 동작 확인 |

> `docker compose down -v`는 기존 Docker volume 데이터를 삭제합니다. 로컬 데이터 삭제가 가능한 경우에만 사용합니다.

상세 내용: [`04_troubleshooting.md`](04_troubleshooting.md)

---

## 11. 코드 탐색 가이드

처음 코드를 볼 때는 아래 순서대로 탐색하는 것이 가장 빠릅니다.

### 인증

```text
collector/
 ├─ controller/ChzzkAuthController.java
 │    로그인 시작 · OAuth callback
 │
 └─ auth/ChzzkAuthService.java
      인증 세션 발급 · 검증

core-api/
 ├─ config/OwnerAccessFilter.java
 │    Owner assertion · Redis session 검증
 │
 ├─ config/InternalAccessFilter.java
 │    내부 API 접근 제한
 │
 └─ domain/chat/service/OwnerIdentityResolver.java
      ServerWebExchange에서 ownerId 추출
```

### 실시간 채팅

```text
collector/
 ├─ service/NidChatCollector.java
 │    CHZZK NID WebSocket 연결 · 채팅 수집
 │
 └─ service/ChatProducer.java
      2초 단위 Kafka 배치 발행

analyzer/
 ├─ service/ChatAnalysisProcessor.java
 │    Kafka 소비 · Fast/Slow Path 분기
 │
 ├─ service/HeuristicSentimentAnalyzer.java
 │    키워드 기반 Fast-Path
 │
 └─ service/OllamaAnalyzerService.java
      LLM 기반 Slow-Path · 입출력 가드레일

core-api/
 ├─ domain/chat/service/ChatStreamService.java
 │    Sinks.replay(100) · SSE
 │
 └─ domain/chat/controller/StreamController.java
      /api/v1/sse/{channelId}
```

### VOD 하이라이트

```text
collector/
 ├─ controller/VodCollectorController.java
 │    POST /crawl · GET /status
 │
 ├─ service/VodChatCrawlerService.java
 │    cursor 기반 VOD 채팅 수집
 │
 └─ service/VodAnalysisStatusService.java
      in-memory 분석 상태 관리

analyzer/
 └─ service/VodHighlightAnalyzer.java
      30초 윈도우
      → 점수 계산
      → LLM 리뷰
      → 최종 후보 선별

core-api/
 ├─ domain/chat/controller/VodController.java
 │    분석 시작 · 결과 조회
 │
 ├─ domain/chat/service/VodAnalysisSlotService.java
 │    Redis 동시 분석 제한
 │
 ├─ domain/chat/service/VodHighlightConsumer.java
 │    vod-analyzed-topic 소비
 │
 ├─ domain/chat/service/VodTimelinePointConsumer.java
 │    vod-window-summary-topic 소비
 │
 ├─ domain/chat/service/VodAnalysisEventConsumer.java
 │    분석 완료 후 슬롯 반납
 │
 └─ rag/HighlightEmbeddingService.java
      임베딩 생성 · pgvector 저장
```

---

## 12. 주요 환경 변수

| 변수 | 사용 서비스 | 용도 |
|---|---|---|
| `GAK_POSTGRES_APP_USER` | core-api | DML 계정, 기본값 `gak_app` |
| `GAK_POSTGRES_ADMIN_USER` | core-api | DDL·Flyway 계정, 기본값 `gak_admin` |
| `GAK_OWNER_TOKEN_SECRET` | collector, core-api | Owner assertion HMAC-SHA256 서명 키 |
| `GAK_CHZZK_CLIENT_ID` | collector | CHZZK OAuth Client ID |
| `GAK_CHZZK_CLIENT_SECRET` | collector | CHZZK OAuth Client Secret |
| `GAK_COOKIE_SECURE` | collector | 프로덕션 `true`, 로컬 `false` |

---

## 13. 다음에 볼 문서

처음 프로젝트를 살펴본다면 다음 순서를 권장합니다.

1. [`03_run_guide.md`](03_run_guide.md) — 로컬에서 프로젝트 실행
2. [`08_vod_highlight_sequence_diagrams.md`](08_vod_highlight_sequence_diagrams.md) — 핵심 VOD 데이터 흐름 파악
3. [`01_ADR.md`](01_ADR.md) — 주요 기술 선택과 설계 이유 확인
4. [`06_api_spec.md`](06_api_spec.md) — API와 인증 규칙 확인
5. [`07_erd.md`](07_erd.md) — 데이터 구조 확인
6. [`04_troubleshooting.md`](04_troubleshooting.md) — 장애 발생 시 확인 지점 파악

### 현재 구현에서 알아둘 점

- VOD 분석 상태는 완료 이벤트를 기준으로 변경하며, 조회 시 상태 보정 로직도 사용합니다.
- 정확한 VOD 타임라인을 조회하려면 `vod_timeline_points` 저장 경로가 정상적으로 동작해야 합니다.
- Redis 기반 VOD 분석 슬롯은 **사용자별 1건, 전체 3건**으로 제한합니다.
