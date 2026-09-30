# 07. ERD

> 구현 기준: 2026-05-29
> 스키마 소스: `backend/core-api/src/main/resources/db/migration/V1~V9__*.sql`

이 문서는 데이터 구조를 확인하는 개발자를 위한 스키마 참조 문서다. 관계도와 외래 키(FK) 설계에서 연결 방식을 확인한다. 테이블 설명·인덱스·정규화 정책에서 저장 목적과 제약을 찾을 수 있다.

---

## 관계도

```mermaid
erDiagram
    analyzed_chats {
        bigserial    id           PK
        varchar      message_id   UK "NOT NULL"
        varchar      room_id      "NOT NULL"
        text         content
        varchar      sender
        varchar      sender_id
        varchar      emotion_type
        double       emotion_score
        timestamp    analyzed_at  "NOT NULL"
    }

    highlight_records {
        serial       id           PK
        varchar      room_id      "NOT NULL"
        varchar      emotion_type "NOT NULL"
        double       peak_score   "NOT NULL"
        text         top_message
        text         live_image_url
        timestamp    timestamp    "NOT NULL"
    }

    vod_highlights {
        bigserial    id                PK
        varchar      video_no          "NOT NULL"
        int          start_seconds
        int          end_seconds
        double       highlight_score
        varchar      category
        varchar      scene_label
        varchar      reaction_label
        text         description
        text         reason_summary
        text         top_message
        text         keyword_summary
        double       intensity_score
        double       transition_score
        double       editability_score
        double       laugh_ratio
        double       hype_ratio
        double       surprise_ratio
        double       tension_ratio
        double       density_ratio
        double       unique_user_ratio
        varchar      emotion_dominance
        text         embedding_text
        vector       embedding         "768차원"
        timestamp    created_at        "DEFAULT now()"
    }

    vod_timeline_points {
        bigserial    id                PK
        varchar      video_no          "NOT NULL"
        int          start_seconds
        int          end_seconds
        int          message_count
        int          participant_count
        double       activity_score
        varchar      category
        text         top_message
        timestamp    created_at        "DEFAULT now()"
    }

    user_vod_library {
        bigserial    id               PK
        varchar      owner_id         "NOT NULL"
        varchar      video_no         "NOT NULL"
        varchar      status
        timestamp    last_viewed_at
        timestamp    last_analyzed_at
        timestamp    created_at
        timestamp    updated_at
    }

    user_vod_activity {
        bigserial    id           PK
        varchar      owner_id     "NOT NULL"
        varchar      video_no     "NOT NULL"
        bigint       highlight_id FK
        varchar      action_type  "NOT NULL"
        timestamp    created_at   "DEFAULT now()"
    }

    user_vod_activity }o--o| vod_highlights : "highlight_id → id  ON DELETE SET NULL"
    user_vod_activity }o--|| user_vod_library : "owner_id+video_no  ON DELETE CASCADE"
```

---

## FK 설계

| 제약 | 전략 | 의미 |
|------|------|------|
| `user_vod_activity.highlight_id → vod_highlights.id` | ON DELETE SET NULL | 하이라이트 삭제 시 activity 로그는 보존, 참조만 끊김 |
| `user_vod_activity(owner_id, video_no) → user_vod_library(owner_id, video_no)` | ON DELETE CASCADE | 라이브러리 항목 삭제 시 activity 로그 함께 삭제 |

`video_no`, `room_id`, `owner_id`는 치지직 외부 ID이므로 DB FK를 설정할 수 없다. 애플리케이션 레이어에서 관리한다.

---

## 테이블 설명

### `analyzed_chats`

실시간 라이브 채팅 감정 분석 결과를 저장한다. Kafka `analyzed-chat-topic`에서 소비한 데이터가 여기에 쌓인다.

| 컬럼 | 설명 |
|------|------|
| `message_id` | 치지직 채팅 메시지 고유 ID. UNIQUE 키로 중복 저장을 방지한다. |
| `room_id` | 방송 채널 ID |
| `emotion_type` | `JOY` · `HOPE` · `WONDER` · `HYPE` · `SADNESS` · `ANGER` · `DISGUST` · `NEUTRAL` 중 하나 |
| `emotion_score` | 해당 감정의 신뢰도 점수 [0.0, 1.0] |

---

### `highlight_records`

라이브 방송 중 실시간으로 감지된 감정 피크 순간을 기록한다. `room_id` 기준으로 구간별 최고 반응을 저장한다.

| 컬럼 | 설명 |
|------|------|
| `room_id` | 방송 채널 ID |
| `emotion_type` | 피크 감정 유형 |
| `peak_score` | 해당 시점 최고 감정 점수 |
| `live_image_url` | 캡처 이미지 URL (선택) |

---

### `vod_highlights`

VOD 분석으로 선별한 편집 후보 구간을 저장한다. V3~V9 마이그레이션을 거쳐 점수 체계·RAG 임베딩·FK 제약이 순차적으로 추가됐다.

| 컬럼 그룹 | 컬럼 |
|-----------|------|
| 위치 | `video_no`, `start_seconds`, `end_seconds` |
| 점수 | `highlight_score`, `intensity_score`, `transition_score`, `editability_score` |
| 레이블 | `category`, `scene_label`, `reaction_label` |
| 설명 | `description`, `reason_summary`, `top_message`, `keyword_summary` |
| 신호 비율 | `laugh/hype/surprise/tension_ratio`, `density_ratio`, `unique_user_ratio`, `emotion_dominance` |
| RAG | `embedding_text` (비율 기반 텍스트), `embedding` (vector 768) — VOD 분석 few-shot 검색 및 라이브 유사 하이라이트 감지 양쪽에서 사용 |

**점수 산식**

```
editability_score = 메시지 다양성×2.2 + 발화자 균형×1.8 + 대표 채팅×max4
                  + 전환점수×0.65 + 키워드 집중도×1.2 + 키워드 변화×1.4

highlight_score   = (intensity×0.55 + transition×0.20 + editability×0.25)
                  × edgePenalty × negativePenalty
```

---

### `vod_timeline_points`

30초 윈도우 단위 채팅 활동 집계를 저장한다. 타임라인 시각화용으로 사용하며, 모든 윈도우를 저장하므로 `vod_highlights`보다 레코드 수가 훨씬 많다.

| 컬럼 | 설명 |
|------|------|
| `message_count` | 해당 30초 구간 채팅 수 |
| `participant_count` | 고유 발화자 수 |
| `activity_score` | `messages×1.2 + users×2.1 + burstSignal + variety×8 + coverage×6` |
| `category` | `LAUGH` / `WONDER` / `HYPE` / `TENSION` / `HOT_MOMENT` |

---

### `user_vod_library`

스트리머가 조회하거나 분석을 요청한 VOD 목록을 저장한다. `UNIQUE(owner_id, video_no)`로 1사용자 1VOD를 제한한다.

| `status` | 의미 |
|----------|------|
| `ANALYZING` | 분석 요청 중 |
| `READY` | 분석 완료, 하이라이트 조회 가능 |
| `VIEWED` | 조회만 한 상태 |

---

### `user_vod_activity`

하이라이트에 대한 사용자 행동 로그를 저장한다. 개인화 추천 모델의 학습 데이터로 사용한다.

| `action_type` | 가중치 | 설명 |
|---------------|--------|------|
| `PIN` | +4.0 | 즐겨찾기 |
| `GOOD` / `SAVE` | +3.0 | 긍정 평가 |
| `OPEN` | +1.0 | 클릭해서 봄 |
| `BAD` / `SKIP` | −3.0 | 부정 평가 |

---

## 인덱스

| 테이블 | 인덱스 | 목적 |
|--------|--------|------|
| `analyzed_chats` | UNIQUE(`message_id`) | 중복 채팅 방지 |
| `analyzed_chats` | `(room_id)` | 채널별 채팅 조회 |
| `vod_highlights` | `(video_no)` | VOD별 하이라이트 조회 |
| `vod_highlights` | `ivfflat(embedding vector_cosine_ops, lists=100)` | 벡터 근사 검색 |
| `vod_timeline_points` | `(video_no)` | VOD별 타임라인 조회 |
| `user_vod_library` | UNIQUE(`owner_id`, `video_no`) | 1사용자 1VOD 제한 |
| `user_vod_library` | `(owner_id, updated_at DESC)` | 라이브러리 목록 최신순 조회 |
| `user_vod_activity` | `(owner_id, created_at DESC)` | 행동 로그 최신순 조회 |

---

## 정규화와 의도적 비정규화

기본 테이블은 3NF를 유지하되, 읽기 성능과 재현성을 위해 다음 파생값은 저장한다.

| 컬럼 | 결정 이유 |
|------|-----------|
| `vod_highlights.emotion_dominance` | RAG 입력과 조회에서 네 개 감정 비율을 반복 계산하지 않음 |
| `vod_highlights.embedding_text` | 임베딩 모델 변경 시 원본 입력을 감사·재생성할 수 있음 |
| `vod_timeline_points.activity_score` | 쓰기 1회·읽기 다수인 타임라인에서 집계 재계산을 피함 |

`user_vod_activity.action_type`은 값 종류가 작고 애플리케이션에서 정규화하므로 별도 코드 테이블을 두지 않는다. 내부 참조는 V8·V9의 FK로 보호하고, `video_no`·`room_id`·`owner_id`처럼 CHZZK가 원본을 관리하는 외부 식별자는 애플리케이션에서 관리한다.

---

## 외부 의존

| 데이터 | 저장 위치 | 설명 |
|--------|-----------|------|
| 세션 토큰 | Redis `gak:owner-session:{ownerId}` | HMAC 검증된 세션 |
| OAuth state | Redis `gak:auth:state:{state}` TTL=10m | CSRF 방지 |
| VOD 분석 상태 | collector 인메모리 (`ConcurrentHashMap`) | 재기동 시 초기화됨 |
| 동시 슬롯 카운터 | Redis `vod:active:user:{id}`, `vod:active:global` | TTL=30m, fail-open |
| VOD 메타데이터 | Chzzk API (외부) | title, duration, category |
| V2 최신 프레임 | Redis `gak:v2:state:{roomId}` | 민심 탭 초기 로드용 최신 집계 프레임 |
| V2 스파이크 쿨다운 | 인메모리 `ConcurrentHashMap` | roomId별 마지막 알림 시각 (재기동 시 초기화) |
