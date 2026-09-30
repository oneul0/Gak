# 06. API 명세

> 기준: 2026-05-29
> 서비스: collector (8081) · core-api (8083) · analyzer (8082, REST 없음)

이 문서는 API를 연동하는 개발자가 엔드포인트의 메서드·경로·인증 조건을 찾기 위한 참조 문서입니다. 인증 방식을 먼저 확인한 뒤 서비스와 기능별 명세에서 필요한 요청·응답 정보를 확인하세요.

---

## 인증 방식

| 방식 | 수단 | 적용 범위 |
|------|------|-----------|
| **Owner** | `GAK_OWNER_ASSERTION` 쿠키 (HMAC-SHA256) + Redis 세션 검증 | 채널 소유자 전용 쓰기 작업 |
| **Internal** | `X-Internal-Secret` 헤더 | `/internal/**` MSA 내부 통신 (불일치 시 404 반환) |
| **Public** | 없음 | 조회 전용 엔드포인트 (일부는 로그인 시 개인화) |

---

## Collector (8081)

### 인증 — `/api/v1/chzzk`

| 메서드 | 경로 | 인증 | 설명 |
|--------|------|------|------|
| GET | `/api/v1/chzzk/login` | Public | CHZZK OAuth 인가 URL로 리다이렉트. `GAK_CHZZK_AUTH_STATE` 쿠키 발급 (10분 TTL) |
| GET | `/api/v1/chzzk/callback` | Public | OAuth 코드 수신 → 액세스 토큰 교환 → 세션 생성. `GAK_CHZZK_AUTH_SESSION` + `GAK_OWNER_ASSERTION` 쿠키 발급 후 `/channels/{channelId}?auth=success` 리다이렉트 |
| GET | `/api/v1/chzzk/me` | Public | 현재 세션 조회. 만료 5분 전이면 자동 갱신 |
| DELETE | `/api/v1/chzzk/logout` | Public | 액세스 토큰 revoke + Redis 세션 삭제 + 쿠키 초기화 |

**`GET /me` 응답**
```json
{
  "authenticated": true,
  "channelId": "string",
  "channelName": "string",
  "expiresAt": "ISO8601",
  "refreshed": false
}
```

---

### 채널 구독 — `/api/v1/channels`

| 메서드 | 경로 | 인증 | 설명 |
|--------|------|------|------|
| GET | `/api/v1/channels/{channelId}/status` | Public | CHZZK API에서 라이브 상태 조회 (live/offline, 채팅 채널 ID, 시청자 수) |
| POST | `/api/v1/channels/{channelId}/subscribe` | Owner | WebSocket 연결 시작, 실시간 채팅 수집 개시. 성인 방송이면 422 반환 |
| DELETE | `/api/v1/channels/{channelId}/subscribe` | Owner | WebSocket 연결 종료 |

**`GET /status` 응답**
```json
{
  "channelId": "string",
  "live": true,
  "status": "live | offline | failed",
  "chatChannelId": "string",
  "liveTitle": "string",
  "viewerCount": 0
}
```

---

### VOD 크롤링 — `/api/v1/vod`

| 메서드 | 경로 | 인증 | 설명 |
|--------|------|------|------|
| GET | `/api/v1/vod/{videoNo}/metadata` | Public | CHZZK에서 VOD 제목·길이·썸네일 조회 |
| GET | `/api/v1/vod/{videoNo}/status` | Public | 분석 상태 조회 (`IDLE → REQUESTED → ANALYZING → COMPLETED / FAILED`). ANALYZING 상태가 30분 경과하면 core-api 결과 확인 후 자동 보정 |
| POST | `/api/v1/vod/{videoNo}/crawl` | Public | VOD 채팅 크롤링 시작 (비동기). Kafka로 채팅 청크 발행 후 완료 신호 송출 |

**`GET /status` 응답**
```json
{
  "videoNo": "string",
  "status": "IDLE | REQUESTED | ANALYZING | COMPLETED | FAILED",
  "startedAt": "ISO8601",
  "pagesProcessed": 0,
  "chatsCollected": 0,
  "message": "string"
}
```

---

### 개발용 — `/api/v1/dev`

| 메서드 | 경로 | 인증 | 설명 |
|--------|------|------|------|
| POST | `/api/v1/dev/mock-chat/{roomId}` | Public | 테스트용 채팅 주입. `?count=8` (max 50). Kafka + Redis에 발행 |

---

## Core-API (8083)

### V2 민심 스트림 — `/api/v2`

> V2 전용 엔드포인트는 라이브 채팅 심리 분석 결과와 유사 하이라이트 알림을 SSE로 전달합니다.

| 메서드 | 경로 | 인증 | 설명 |
|--------|------|------|------|
| GET | `/api/v2/stream/{roomId}` | Public | V2 실시간 SSE 스트림. 15초마다 keep-alive ping |
| GET | `/api/v2/state/{roomId}` | Public | Redis에 저장된 최신 V2 프레임 조회 (SSE 연결 전 초기 상태 로드용) |

**SSE 이벤트 타입**

| 이벤트 | 발생 시점 | 설명 |
|--------|-----------|------|
| `ping` | 15초마다 | 연결 유지용 keep-alive |
| `v2_frame` | Kafka `v2-aggregate` 토픽 소비 시 | EMA 감정 집계, 대표 채팅, 키워드, AI 브리핑, 신뢰 등급 분포 |
| `v2_similar_highlight` | 스파이크 감지 + pgvector 유사도 임계값 초과 시 | 과거 VOD 하이라이트 유사 패턴 알림 |

**`v2_frame` 이벤트 데이터**
```json
{
  "roomId": "string",
  "emittedAt": "ISO8601",
  "balance": 0.85,
  "mentalBuffer": {
    "emaPositive": 0.82,
    "emaNegative": 0.08
  },
  "anchors": [
    { "messageId": "string", "sender": "string", "content": "string", "weight": 24 }
  ],
  "keywords": ["string"],
  "topicLabel": "string",
  "briefing": {
    "summary": "string",
    "confidence": 0.88
  },
  "trustSummary": {
    "fanCount": 18,
    "normalCount": 30,
    "trollCount": 2,
    "total": 50
  }
}
```

**`v2_similar_highlight` 이벤트 데이터**
```json
{
  "roomId": "string",
  "highlightId": 0,
  "videoNo": "string",
  "sceneLabel": "string",
  "category": "string",
  "reasonSummary": "string",
  "similarity": 0.85,
  "trigger": "positive_spike | negative_spike",
  "detectedAt": "ISO8601",
  "insight": "시청자들이 ~하고 있어요"
}
```

> **스파이크 감지 기준**: `emaPositive > 0.55` → `positive_spike`, `emaNegative > 0.45` → `negative_spike`  
> **유사도 임계값**: 환경변수 `GAK_V2_SIMILARITY_THRESHOLD` (기본값 `0.72`)  
> **쿨다운**: `GAK_V2_SIMILARITY_ALERT_COOLDOWN_MINUTES` (기본값 `3`)  
> **insight 생성**: Ollama `gemma:2b`로 현재 채팅 컨텍스트 + 과거 장면 정보를 조합해 자연어 해석 문장 생성

---

### SSE 스트림 — `/api/v1/stream`

| 메서드 | 경로 | 인증 | 설명 |
|--------|------|------|------|
| GET | `/api/v1/stream/{roomId}` | Public | 실시간 채팅 감정 분석 SSE 스트림. 15초마다 keep-alive ping |
| GET | `/api/v1/stream/{roomId}/history` | Public | DB에서 최근 분석된 채팅 이력 조회 |

**SSE 이벤트 타입**: `ping`, `analyzed_chat`, `sentiment`

**`/history` 응답 항목**
```json
{
  "id": 0,
  "roomId": "string",
  "senderId": "string",
  "sender": "string",
  "content": "string",
  "sentiment": "string",
  "timestamp": "ISO8601"
}
```

---

### VOD 분석 결과 — `/api/v1/vod`

| 메서드 | 경로 | 인증 | 설명 |
|--------|------|------|------|
| GET | `/api/v1/vod/{videoNo}/highlights` | Public (로그인 시 개인화) | 편집 후보 구간 목록. 로그인된 소유자에게는 선호도 기반 정렬 적용 |
| GET | `/api/v1/vod/{videoNo}/timeline` | Public | 30초 윈도우 단위 채팅 활동 집계. 타임라인 미니맵용 |
| POST | `/api/v1/vod/{videoNo}/analyze` | Owner | VOD 분석 요청. 슬롯 초과 시 429(사용자 한도) 또는 503(전역 한도) |

**`GET /highlights` 응답 항목**
```json
{
  "id": 0,
  "videoNo": "string",
  "startSeconds": 0,
  "endSeconds": 0,
  "highlightScore": 0.0,
  "category": "string",
  "sceneLabel": "string",
  "emotionDominance": "HYPE | LAUGH | WONDER | TENSION",
  "intensityScore": 0.0,
  "transitionScore": 0.0,
  "editabilityScore": 0.0,
  "reasonSummary": "string",
  "keywordSummary": "string",
  "topMessage": "string",
  "densityRatio": 0.0,
  "createdAt": "ISO8601"
}
```

**`GET /timeline` 응답 항목**
```json
{
  "id": 0,
  "videoNo": "string",
  "startSeconds": 0,
  "endSeconds": 0,
  "messageCount": 0,
  "participantCount": 0,
  "activityScore": 0.0,
  "category": "LAUGH | WONDER | HYPE | TENSION | HOT_MOMENT"
}
```

---

### 사용자 VOD 라이브러리 — `/api/v1/me`

| 메서드 | 경로 | 인증 | 설명 |
|--------|------|------|------|
| GET | `/api/v1/me/vod-library` | Owner | 소유자가 조회·분석한 VOD 목록 |
| GET | `/api/v1/me/vod-preferences` | Owner | 선호 카테고리·감정 통계 (개인화 추천용) |
| GET | `/api/v1/me/vod/{videoNo}/activity` | Owner | 해당 VOD에서 수행한 행동 이력 |
| POST | `/api/v1/me/vod/{videoNo}/activity` | Owner | 하이라이트 행동 기록 (클릭·저장·평가 등) |

**`POST /activity` 요청 바디**
```json
{ "highlightId": 0, "actionType": "OPEN | GOOD | BAD | PIN | SAVE | SKIP" }
```

---

### 투표 — `/api/v1/votes`

| 메서드 | 경로 | 인증 | 설명 |
|--------|------|------|------|
| POST | `/api/v1/votes/{roomId}/start` | Public | 투표 수집 시작. 채팅 `!1`, `!2` 등으로 투표 |
| POST | `/api/v1/votes/{roomId}/stop` | Public | 투표 수집 종료 |
| GET | `/api/v1/votes/{roomId}/keywords` | Public | 현재 투표 세션의 키워드 빈도 맵 조회 |

---

### 투표 관리 — `/api/v1/poll`

| 메서드 | 경로 | 인증 | 설명 |
|--------|------|------|------|
| POST | `/api/v1/poll/{roomId}/session` | Owner | 세션 수집 활성화 여부 설정. `?active=true/false` |
| GET | `/api/v1/poll/{roomId}/session` | Public | 세션 수집 활성 여부 조회 |
| POST | `/api/v1/poll/{roomId}/items` | Owner | 투표 항목 설정 (2~20개, 항목당 1~50자) |
| GET | `/api/v1/poll/{roomId}/items` | Public | 현재 투표 항목 조회 |
| GET | `/api/v1/poll/{roomId}/results` | Public | 투표 집계 결과. `{ "항목명": count }` |
| GET | `/api/v1/poll/{roomId}/voters` | Public | 투표자별 선택 항목. `{ "닉네임": "항목명" }` |
| GET | `/api/v1/poll/{roomId}/voters/{userId}/history` | Public | 특정 투표자의 채팅 이력 |
| DELETE | `/api/v1/poll/{roomId}` | Owner | 투표 전체 초기화 (항목·집계·투표자) |

---

### 도네이션 — `/api/v1/donations`

| 메서드 | 경로 | 인증 | 설명 |
|--------|------|------|------|
| GET | `/api/v1/donations/{channelId}` | Public | Redis 도네이션 큐 조회 |
| POST | `/api/v1/donations/{channelId}/spin` | Owner | 큐 맨 앞 도네이션 팝 (FIFO). 비어 있으면 404 |
| DELETE | `/api/v1/donations/{channelId}` | Owner | 도네이션 큐 전체 삭제 |

**도네이션 항목**
```json
{
  "messageId": "string",
  "donorNickname": "string",
  "message": "string",
  "amount": "string",
  "timestamp": "ISO8601"
}
```

---

### 룰렛 — `/api/v1/roulette`

| 메서드 | 경로 | 인증 | 설명 |
|--------|------|------|------|
| GET | `/api/v1/roulette/{channelId}` | Public | 룰렛 항목·도네이션 가중치·확률 조회 |
| PUT | `/api/v1/roulette/{channelId}/config` | Owner | 항목 및 배율(rate) 설정 |
| POST | `/api/v1/roulette/{channelId}/spin` | Owner | 가중치 기반 랜덤 선택 실행 |
| DELETE | `/api/v1/roulette/{channelId}/weights` | Owner | 가중치만 초기화 |
| DELETE | `/api/v1/roulette/{channelId}` | Owner | 항목·가중치·설정 전체 초기화 |

**`PUT /config` 요청 바디**
```json
{ "items": ["항목1", "항목2"], "rate": 1000 }
```

---

### 라이브 목록 — `/api/v1/lives`

| 메서드 | 경로 | 인증 | 설명 |
|--------|------|------|------|
| GET | `/api/v1/lives` | Public | 현재 수집 중인 채널 목록 + 감정 히트맵. `?size=20&next={cursor}` |

**응답**
```json
{
  "channels": [
    {
      "channelId": "string",
      "channelName": "string",
      "liveTitle": "string",
      "sentiment": { "JOY": 0.3, "HYPE": 0.5 },
      "viewerCount": 0
    }
  ],
  "nextCursor": "string"
}
```

---

### 내부 RAG — `/internal/rag`

| 메서드 | 경로 | 인증 | 설명 |
|--------|------|------|------|
| POST | `/internal/rag/few-shot` | Internal | analyzer가 LLM 프롬프트 구성 전 유사 하이라이트 예시 조회. `?k=3` |

**요청 바디**
```json
{
  "videoNo": "string",
  "category": "string",
  "sceneLabel": "string",
  "emotionDominance": "string",
  "densityRatio": 0.0
}
```

**응답**: 지정 형식의 few-shot 예시 문자열을 반환합니다. 빈 문자열이면 RAG 없이 LLM 처리를 진행합니다.

---

### 개발용 데이터 시드 — `/api/dev/seed`

> `gak.dev-seed.enabled=true`일 때만 활성화합니다.

| 메서드 | 경로 | 설명 |
|--------|------|------|
| POST | `/api/dev/seed/{channelId}/donations` | 테스트 도네이션 주입. `?count=10` (max 50) |
| POST | `/api/dev/seed/{channelId}/votes` | 테스트 투표 주입. `?voters=30` (max 200) |
| POST | `/api/dev/seed/{channelId}/roulette-donations` | 룰렛 항목 기반 도네이션 주입. `?count=20` (max 100) |
| GET | `/api/dev/seed/{channelId}` | 도네이션 풀 크기·투표 현황 조회 |
| DELETE | `/api/dev/seed/{channelId}` | 전체 시드 데이터 초기화 |
