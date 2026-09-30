# 12. 프로덕션 배포 체크리스트

> 이 문서는 배포 전 반드시 확인해야 할 보안·환경 설정 항목을 정리합니다.
> 로컬 개발 실행은 `03_run_guide.md`를 참고하세요.

배포 담당자는 시크릿 생성부터 환경 설정, DB 계정 준비, 접근 범위와 배포 후 동작 확인까지 순서대로 점검할 수 있습니다. 신규 서버와 기존 PostgreSQL 볼륨이 있는 환경의 절차를 구분하고, 배포 후에는 시크릿 재발급 조건과 후속 확인 절차를 참고하세요.

---

## Step 1. 시크릿 생성

아래 명령어를 **각각 한 번씩** 실행해 서로 다른 값 4개를 생성합니다.

```bash
openssl rand -hex 32   # GAK_OWNER_TOKEN_SECRET
openssl rand -hex 32   # GAK_INTERNAL_API_SECRET
openssl rand -hex 32   # GAK_POSTGRES_ADMIN_PASSWORD
openssl rand -hex 32   # GAK_POSTGRES_APP_PASSWORD
```

출력 예시:
```
a3f8c2d1e4b5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1
```

생성한 각 값을 `.env`의 해당 항목에 입력합니다.

---

## Step 2. backend/.env 작성

`backend/.env.example`을 복사해 `backend/.env`를 만들고 아래 항목을 채웁니다.

```env
# ── Chzzk Open API ─────────────────────────────────────────────────────────
# Chzzk 개발자 콘솔(https://developers.chzzk.naver.com)에서 발급
CHZZK_CLIENT_ID=여기에_발급받은_ID
CHZZK_CLIENT_SECRET=여기에_발급받은_SECRET
CHZZK_CLIENT_URI=https://실제도메인.com/api/v1/chzzk/callback

# ── 보안 시크릿 ──────────────────────────────────────────────────────────────
# openssl rand -hex 32 로 생성한 값. 각 항목 서로 다른 값 사용.
# 절대 기본값(dev-***-secret)을 그대로 쓰지 말 것.
GAK_OWNER_TOKEN_SECRET=Step1에서_생성한_값_1
GAK_INTERNAL_API_SECRET=Step1에서_생성한_값_2

# ── CORS ─────────────────────────────────────────────────────────────────────
# 프론트엔드가 배포된 실제 도메인. 여러 개면 콤마로 구분.
GAK_CORS_ALLOWED_ORIGINS=https://실제도메인.com

# ── PostgreSQL 관리자 계정 (Flyway DDL 전용) ──────────────────────────────────
# docker-compose의 POSTGRES_USER에 해당. 스키마 마이그레이션에만 사용됨.
GAK_POSTGRES_ADMIN_USER=gak_admin
GAK_POSTGRES_ADMIN_PASSWORD=Step1에서_생성한_값_3
GAK_POSTGRES_DB=gak_db

# ── PostgreSQL 앱 런타임 계정 (SELECT/INSERT/UPDATE/DELETE만 허용) ─────────────
# 애플리케이션이 실제로 DB에 접속할 때 사용하는 계정. DDL 권한 없음.
# docker compose up 시 init-app-user.sh가 자동으로 이 계정을 생성함.
GAK_POSTGRES_APP_USER=gak_app
GAK_POSTGRES_APP_PASSWORD=Step1에서_생성한_값_4

# ── 개발 전용 시드 ────────────────────────────────────────────────────────────
# 프로덕션에서는 반드시 false
GAK_DEV_SEED_ENABLED=false
```

> **주의:** `.env` 파일은 `.gitignore`에 등록되어 있습니다. git에 커밋하지 마세요.

---

## Step 3. PostgreSQL 컨테이너 최초 기동

**기존 postgres 볼륨이 없는 경우** (신규 서버):
```bash
docker compose up -d postgres
```
컨테이너 로그에 다음 메시지가 있는지 확인합니다.
```
[init-app-user] Runtime user 'gak_app' ready.
```

**기존 볼륨이 있는 경우** (볼륨이 존재하면 init 스크립트가 재실행되지 않음):
```bash
# postgres 컨테이너 접속 후 수동 실행
docker exec -it gak-postgres psql -U $GAK_POSTGRES_ADMIN_USER -d $GAK_POSTGRES_DB \
  -c "CREATE USER gak_app WITH PASSWORD '실제앱패스워드';"
docker exec -it gak-postgres psql -U $GAK_POSTGRES_ADMIN_USER -d $GAK_POSTGRES_DB \
  -c "GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO gak_app;"
docker exec -it gak-postgres psql -U $GAK_POSTGRES_ADMIN_USER -d $GAK_POSTGRES_DB \
  -c "GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO gak_app;"
docker exec -it gak-postgres psql -U $GAK_POSTGRES_ADMIN_USER -d $GAK_POSTGRES_DB \
  -c "ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO gak_app;"
docker exec -it gak-postgres psql -U $GAK_POSTGRES_ADMIN_USER -d $GAK_POSTGRES_DB \
  -c "ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT USAGE, SELECT ON SEQUENCES TO gak_app;"
```

---

## Step 4. 시크릿이 적용됐는지 확인

서비스를 시작한 뒤 로그에 아래 WARN 메시지가 없는지 확인합니다.

```
[Security] gak.owner-token-secret is using the insecure default value.
[Security] gak.internal-api-secret is using the insecure default value.
```

WARN 메시지가 있으면 `.env` 값을 반영하지 못한 상태입니다. 서비스를 재시작하고 다시 확인합니다.

---

## Step 5. Chzzk callback URI 등록

Chzzk 개발자 콘솔에서 OAuth redirect URI를 프로덕션 도메인으로 업데이트합니다.

```
https://실제도메인.com/api/v1/chzzk/callback
```

로컬 URI(`http://localhost:8081/...`)가 등록된 채로 배포하면 로그인이 동작하지 않습니다.

---

## Step 6. 네트워크 포트 노출 범위 확인

| 서비스 | 포트 | 외부 노출 여부 |
|---|---|---|
| frontend (Next.js) | 3000 | 노출 (또는 Nginx 뒤에 위치) |
| collector | 8081 | 내부망만 |
| analyzer | 8082 | 내부망만 |
| core-api | 8083 | 내부망만 (frontend Next.js가 proxy) |
| PostgreSQL | 5432 | 내부망만 |
| Redis | 6379 | 내부망만 |
| Kafka | 9092 | 내부망만 |

Docker Compose 사용 시 `docker-compose.yml`에서 `ports:` 항목이 외부에 노출되지 않도록 합니다.
core-api와 collector는 `expose:`만 사용하고 `ports:`는 제거합니다.

> collector, analyzer, core-api를 외부에 직접 노출하면 `InternalAccessFilter`나
> `OwnerAccessFilter`를 우회하는 시도가 가능해집니다.

---

## Step 7. 배포 후 동작 확인

```bash
# 1. 인증 없이 보호된 경로 접근 시 401 반환 확인
curl -s -o /dev/null -w "%{http_code}" https://실제도메인.com/api/v1/poll/test123/session
# → 401

# 2. 내부 경로 외부 접근 시 404 반환 확인
curl -s -o /dev/null -w "%{http_code}" https://실제도메인.com/internal/rag/few-shot
# → 404

# 3. 잘못된 origin에서 CORS 차단 확인
curl -s -H "Origin: https://evil.com" https://실제도메인.com/api/v1/lives
# → Access-Control-Allow-Origin 헤더 없음
```

---

## 시크릿 재발급이 필요한 경우

- 시크릿이 유출됐거나 유출이 의심될 때
- 팀원이 퇴사하거나 접근 권한을 회수할 때

**재발급 절차:**
1. `openssl rand -hex 32`로 새 값 생성
2. `.env`의 해당 항목 교체
3. 영향받는 서비스 재시작
   - `GAK_OWNER_TOKEN_SECRET` 변경 시: core-api, collector 재시작 (기존 로그인 세션 전체 만료)
   - `GAK_INTERNAL_API_SECRET` 변경 시: core-api, analyzer 재시작
   - `GAK_POSTGRES_APP_PASSWORD` 변경 시: postgres 컨테이너에서 `ALTER USER gak_app WITH PASSWORD '새값';` 실행 후 core-api 재시작
4. 로그에서 WARN 없음 확인

재시작 후에는 Step 7의 배포 후 동작 확인을 다시 수행합니다. `GAK_OWNER_TOKEN_SECRET`을 변경했다면 기존 로그인 세션이 만료되므로 다시 로그인합니다.

---

## 관련 문서

- [03_run_guide.md](03_run_guide.md) — 로컬 개발 실행 순서
- [11_system_reliability.md](11_system_reliability.md) — 현재 보안 경계와 장애 전략
