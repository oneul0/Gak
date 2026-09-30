# 05. 테스트 전략

이 문서는 현재 저장소에 존재하는 테스트와 최소 검증 순서를 설명한다. 계획이나 일회성 실행 결과는 기록하지 않고, 실제 테스트 코드와 실행 명령만 유지한다.

코드를 변경한 개발자는 [변경 유형별 최소 회귀](#4-변경-유형별-최소-회귀)에서 필요한 검증을 먼저 선택한다. 이후 우선순위와 실행 환경을 확인하고, 백엔드·프론트엔드 명령으로 해당 검증을 수행할 수 있다.

## 1. 검증 우선순위

1. 변경한 모듈의 단위 테스트
2. 서비스 간 DTO·Kafka·DB 경계를 다루는 통합 테스트
3. 인증·VOD 흐름을 다루는 브라우저 E2E

외부 서비스가 필요한 테스트는 기본 테스트와 분리한다. 자동 테스트에서는 CHZZK와 Ollama의 실제 네트워크를 호출하지 않는다.

## 2. 백엔드 테스트

| 모듈 | 주요 대상 |
|------|-----------|
| `analyzer` | 채팅 최적화, 감정 분석, LLM 가드레일, VOD 하이라이트 계산 |
| `collector` | 애플리케이션 컨텍스트, Mock CHZZK WebSocket |
| `core-api` | 인증 식별자, RAG 임베딩·검색, VOD·전체 파이프라인 E2E |

전체 테스트:

```bash
cd backend
./gradlew test
```

모듈 단위 실행:

```bash
cd backend
./gradlew :analyzer:test
./gradlew :collector:test
./gradlew :core-api:test
```

`DatabaseConnectionTest`, `CoreApiApplicationTests`, `FullPipelineE2ETest` 등 일부 테스트는 Docker 또는 Testcontainers가 필요하므로 `@Disabled`로 비활성화되어 있다. 활성화할 때는 PostgreSQL·Redis·Kafka 상태와 테스트 데이터 정리 범위를 먼저 확인한다.

## 3. 프론트엔드 검증

정적 검증:

```bash
cd frontend
npm run lint
npx tsc --noEmit
```

브라우저 E2E:

```bash
cd frontend
npx playwright test
```

현재 시나리오는 `frontend/e2e/dashboard.spec.ts`와 `frontend/e2e/vod-board.spec.ts`에 있다. 실행 전 프론트엔드와 필요한 백엔드 서비스를 기동하고, 테스트가 기대하는 채널·VOD 데이터를 준비한다.

## 4. 변경 유형별 최소 회귀

| 변경 | 최소 검증 |
|------|-----------|
| 인증·필터 | `OwnerIdentityResolverTest` + 비로그인 401 + 타인 채널 403 |
| LLM·프롬프트 | `OllamaAnalyzerServiceTest` + malformed/timeout fallback |
| VOD 점수·선별 | `VodHighlightAnalyzerTest` + VOD 보드 E2E |
| RAG·pgvector | `HighlightEmbeddingServiceTest`, `HighlightRetrievalServiceTest`, `PgVectorUtilsTest` |
| Kafka DTO·토픽 | producer/consumer 양쪽 컴파일 + 관련 E2E |
| DB 마이그레이션 | 빈 DB Flyway 기동 + 기존 스키마 migrate |

테스트 실행과 환경 준비는 [`03_run_guide.md`](03_run_guide.md), 장애 발생 시 점검 순서는 [`04_troubleshooting.md`](04_troubleshooting.md)를 참고한다.
