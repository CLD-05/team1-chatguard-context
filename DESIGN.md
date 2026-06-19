# 설계 명세서 (Design Spec) — chatguard (라이브 채팅 + 실시간 AI 검열 플랫폼)

> 이 문서는 팀의 **단일 진실 공급원(SSoT)**. 모든 구현과 각자의 AI 지침은 이 문서를 기준으로 한다.
> v1 동결 후 변경은 팀 합의로만 한다.
> 표기 — 🟢 확정 · 🟡 팀 결정 필요.
> 상태 — **A·B·C·D 섹션 v1 동결(2026-06-04) · E 섹션 동결(2026-06-10).** 모든 결정 확정. (단 D5 SLO 수치·D10 부하 규모는 1차 부하테스트 후 재확정.) · **scale-in 결정 D11~D18 추가(2026-06-10), 구현 Week 4·6/17 무관.** · **인터페이스 계약 보강(2026-06-10): A-3 검열상태 매핑·REST 메시지 스키마, A-5 런타임 계약, 로그인·WS인증·self-echo 규칙 (결정 기록 D19~D24).** · **인터페이스 계약 2차 보강 + 에러 계약(2026-06-10 합의): moderation_logs FK 제거·WS 방 바인딩·캐치업 윈도우·적재/전파 순서·에러 계약·차단 피드백 (결정 기록 D25~D30).** · **배포 자동화 세션 반영(2026-06-15): 컨테이너 빌드·파이프라인·GitOps 변동값·EKS 접근·ECR 네이밍·frontend 배포·securityContext·nginx 프록시 (결정 기록 D31~D39). dev EKS에 앱 수동 배포로 end-to-end 동작 검증 완료, 파이프라인 자동화는 후속.** 근거·대안은 하단 "결정 기록" 참조.
> 식별 — 팀 `team1` · 서비스 `chatguard` · VPC CIDR `10.1.0.0/16` (dev `10.1.0.0/17` · prod `10.1.128.0/17`).
>
> **인증 강화 반영(2026-06-16): 멘토 보안 피드백으로 D4 번복 → username+password+BCrypt 검증 도입(결정 기록 D40). A-2 users 스키마·인증·로그인 의미론, A-3 REST·에러표 갱신. app PR #42 머지.**
>
> **app 고도화 반영(2026-06-18): Admin & Moderation Control Plane — 어드민 인가(users.role)·금칙어 DB화(인메모리 캐시 + Redis 무효화)·2차 카테고리별 정적 임계값·참여자 표시(presence)·채팅 freeze (결정 기록 D41~D46). A-1~A-5 갱신. 🟢 설계 확정·구현 예정. (회원가입은 보류, 사용자 방 생성은 미구현, 동영상은 FE-only 독립 재생으로 백엔드 계약 무영향.)**
>
> **통신 경로 정정 반영(2026-06-19): scale-out/in 검증 우선순위에 따라 WS·REST 경로를 nginx 리버스 프록시 → ALB 직결로 개정(D39 개정, 결정 기록 D47). nginx는 정적 서빙 유지(D37 불변). 이로써 D18(ALB LOR)·D15(드레인)가 실제 작동. S3+CloudFront 정적 고도화는 보류(D48). A-1·A-5 갱신.**

---

## A. 컴포넌트 책임 · 데이터 모델 · 메시지 스키마

### A-1. 컴포넌트 책임

| 컴포넌트            | 기술                           | 책임                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| ------------------- | ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Frontend            | React                          | 채팅 UI, WebSocket 클라이언트, 메시지 렌더링, `moderation.hide` 수신 시 해당 메시지 블러/삭제, REST로 방·히스토리 조회, `error` 이벤트 처리(차단됨 표시·재시도 안내)와 close code 분기(1001=재연결 / 1008=중단)(D29·D30). **배포 = 클러스터 파드(nginx 정적 서빙, D37). `/api`·`/ws`는 nginx를 거치지 않고 ALB Ingress가 Chat Server로 직접 라우팅(D47, D39 개정) — nginx 리버스 프록시 제거, 프론트는 상대경로·`wss://${location.host}/ws` 무변경.** **[Week4]** WebSocket 자동 재연결 + jittered backoff, 재접속 시 최근 윈도우 캐치업 병합(검열 상태 `message.status` 단일화) **[고도화]** 어드민 페이지(대시보드·금칙어·로그 탭, 진입 가드 `role=ADMIN`), 참여자 수·목록(`presence.update`), freeze 배너(`room.freeze`), 동영상 독립 재생(백엔드 무관) (D41·D44·D45) |
| Chat Server         | Spring Boot (WebSocket + REST) | 연결 수락·유지, 인증, 채팅 수신, **1차 키워드 검열(동기)**, message id 발급, Redis 채널 publish/subscribe, **구독 수신 시** 자기 클라이언트에 push(로컬 직접 push 안 함 — A-4 step3), 의심 메시지 큐 적재, MySQL 영속화, 메트릭 노출. **[Week4]** graceful drain(preStop sleep → SIGTERM 1001 close), 새 연결 분산(ALB LOR) + pod당 연결 cap **[고도화]** 금칙어 DB+인메모리 캐시+`config:banned-words` 무효화 구독, 어드민 인가(role)·`/api/admin/*`(403 FORBIDDEN), presence(Redis Set 갱신+`presence.update`), freeze(상태 캐시+`chat.send` 집행) (D41·D42·D44·D45)                                                                                                                                                                                                   |
| Moderation Worker   | Python (transformers)          | 검열 큐 소비, AI 모델 판정(in-process), `moderation_logs` 기록, 악성 시 `messages.status` 갱신 + `moderation.hide` publish, 메트릭 노출. **KEDA가 큐 깊이로 스케일하는 대상**. **[Week4]** graceful shutdown(SIGTERM 시 현재 job 마무리 후 종료, 한 번에 1개 pop) **[고도화]** 2차 verdict = 카테고리별 정적 임계값 정책(D43)                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| Redis (ElastiCache) | —                              | Pub/Sub 채널(전파) + 검열 큐(Redis List)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| MySQL (RDS)         | —                              | users · rooms · messages · moderation_logs 영속 저장                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |

> **[Week4] 태그** = 설계 확정·구현은 Week 4(중간점검 후). 6/17 의무 아님. 상세는 하단 결정 기록 D11~D18 / 학습 문서 `scale-in 설계노트`.

**설계 결정**

- 🟢 Chat Server는 REST와 WebSocket을 **한 앱**으로. (분리는 스트레치)
- 🟢 **Worker 모델 호스팅 = in-process** (워커 프로세스 내 모델 로드). 작은 CPU 모델이라 단순하고 스케일 단위가 명확. (근거·대안: 결정 기록 D3)
- **Terraform vs GitOps 경계**(가이드라인 17) — Terraform = VPC·EKS·RDS·Redis·IAM·ECR + ArgoCD/KEDA/Prometheus Helm 설치 + Namespace/StorageClass. GitOps(ArgoCD ← config repo) = 앱 Deployment/Service/Ingress, HPA, KEDA ScaledObject, ServiceMonitor, 대시보드.
- 🟢 **외부 라우팅 = ALB Ingress가 경로 분기**(D47): `/api`·`/ws` → Chat Server Service, `/` → Frontend(nginx 정적) Service. nginx 리버스 프록시 제거. 같은 출처(단일 ALB 호스트)라 CORS 무발생·홉 최단. WS 장수 연결 위해 ALB `idle_timeout.timeout_seconds` 연장. (D39 개정 — 근거: 결정 기록 D47)

### A-2. 데이터 모델 (MySQL)

🟢 **message id = 서버 발급 ULID(26자 문자열).** 수신 즉시 발급 → 그 id로 broadcast·큐 적재 → 나중에 `moderation.hide`가 그 id를 타겟. (DB auto-id를 기다리면 전파가 느려지므로 앱에서 발급)

```
users
  id            BIGINT      PK
  username      VARCHAR(50) UNIQUE
  password      VARCHAR(60)            -- BCrypt 해시(60자 고정). 평문 미저장 (D40)
  display_name  VARCHAR(50)
  role          ENUM('USER','ADMIN') NOT NULL DEFAULT 'USER'   -- 어드민 인가 (D41)
  created_at    DATETIME

rooms
  id            BIGINT      PK
  name          VARCHAR(100)
  streamer_name VARCHAR(50)
  created_at    DATETIME

messages
  id          CHAR(26)  PK            -- ULID, 서버 발급
  room_id     BIGINT    FK -> rooms.id
  user_id     BIGINT    FK -> users.id
  content     TEXT
  status      ENUM('VISIBLE','BLURRED','DELETED') DEFAULT 'VISIBLE'
  created_at  DATETIME
  INDEX (room_id, created_at)

moderation_logs
  id            BIGINT   PK
  message_id    CHAR(26)               -- ULID. FK 아님(키워드 차단 건은 messages 미존재 — D25)
  stage         ENUM('KEYWORD','AI')
  verdict       ENUM('PASS','BLOCK')
  score         FLOAT       NULL       -- AI 판정 점수
  model_version VARCHAR(50) NULL
  reason        VARCHAR(200) NULL
  content       TEXT        NULL       -- KEYWORD·BLOCK 행에서만 기록(차단 본문 감사 — D25)
  checked_at    DATETIME
  INDEX (message_id)

banned_words                           -- 1차 키워드 전역 목록 (D42)
  id          BIGINT       PK
  word        VARCHAR(100) UNIQUE       -- 매칭=대소문자 정규화 substring contains
  created_by  BIGINT       FK -> users.id   -- 추가한 어드민
  created_at  DATETIME
```

- `messages.status`가 렌더링을 결정: VISIBLE 노출 / BLURRED 블러 / DELETED 숨김.
- `moderation_logs`는 감사 추적이자 대시보드 재료(단계별 차단율 등). 키워드 차단 건은 messages에 저장되지 않으므로 `content`에 원문을 보존한다(D25).
- 🟢 **시간 규약 = 모든 시각 UTC 저장·전송** (DB DATETIME = UTC, API는 ISO-8601 `Z` 표기).
- 🟢 **rooms 생성 경로**: v1은 방 생성 API 없음 — **마이그레이션 시드로 주입**(데모용 고정 방 N개).
- 🟢 **인증 = username + password 로그인 + 액세스 전용 JWT.** Spring Security 의존성은 도입하되 **BCrypt 비밀번호 해싱·검증 용도로만** 사용하고, 인가는 기존 자체 `JwtAuthInterceptor`를 유지(`SecurityFilterChain`은 `anyRequest().permitAll()`로 무중단 구성 — REST는 ThreadLocal 홀더, WS는 핸드셰이크에서 검증 후 세션 바인딩). 리프레시 토큰·풀 RBAC은 미채택. (근거·대안·D4 번복 경위: 결정 기록 D4·D40)
- 🟢 **JWT 클레임 = `{ sub: user_id, display_name, role }`.** broadcast 시 Chat Server는 검증된 토큰/세션에서 display_name을 읽어 **메시지당 DB 조회를 하지 않는다**(전송 p95 SLO 보호). `role`은 어드민 인가용(D41) — 토큰에 박히므로 어드민 박탈은 토큰 만료까지 지연(액세스 전용 JWT의 알려진 트레이드오프, D4·D29와 일관). (결정 기록 D22·D41)
- 🟢 **WS 접속 = 접속 URL 쿼리파라미터 `?token=...&room_id=...`** → 핸드셰이크에서 토큰 검증 + 방 존재 확인 후 **세션을 방 1개에 바인딩**. 방 이동은 재접속으로 처리(`room.join` 이벤트는 스트레치). (노출 회피가 필요하면 `Sec-WebSocket-Protocol` 서브프로토콜로 확장 — 스트레치. 결정 기록 D21·D26)
- 🟢 **로그인 의미론**: username으로 조회 → 없으면 `error(UNAUTHORIZED)`, 있으면 `passwordEncoder.matches()`로 비밀번호 검증 → 불일치 시 401. **자동 회원가입은 폐지**(D40 — 기존 "처음 보는 username이면 생성"을 대체). 계정은 마이그레이션 시드로 주입(데모용 `user1`~`user6`, BCrypt 해시). 응답 `{ user_id, display_name, role, token }`(role은 FE 어드민 내비 게이팅용 — 실 강제는 서버 인터셉터).
- 🟢 **어드민 인가 = `users.role`(USER/ADMIN) + JWT role 클레임.** `/api/admin/*`는 인터셉터가 `role=ADMIN`을 검증, 미달 시 403 `FORBIDDEN`. **단일 플래그**(권한 매트릭스·런타임 RBAC 아님 — D40 정신 유지). 시드에서 최소 1명(예: `user1`)을 ADMIN으로. admin 여부는 도메인 데이터라 env 허용목록 대신 DB 컬럼으로(config↔app 결합 회피·확장). (결정 기록 D41)
- 🟢 **금칙어(1차 키워드) = `banned_words` DB 진실원 + Chat Server 파드 인메모리 캐시 + `config:banned-words` Redis pub/sub 무효화.** 검사는 캐시만 읽어 **메시지당 DB 조회 없음**(전송 SLO 보호). 운영자가 어드민 페이지로 런타임 CRUD. `.env` 방식 폐지(EKS 미도달 버그 해소). 전역(모든 방·유저 공통). 매칭=대소문자 정규화 substring contains. (결정 기록 D42)

### A-3. 메시지 · 이벤트 스키마

공통 봉투: `{ "type": "...", "payload": { ... } }`

**WebSocket — Client → Server**

```json
{ "type": "chat.send", "payload": { "room_id": 1, "content": "안녕하세요" } }
```

(user_id는 토큰에서 검증된 AuthContext에서 추출 — body로 받지 않음. **`room_id`는 핸드셰이크에서 바인딩된 세션 방과 일치해야 하며, 불일치 시 `error(ROOM_MISMATCH)` 거부 — D26.** 재연결 직전 전송분의 중복 방지(클라 멱등키)는 v1 비범위.)

**WebSocket — Server → Client**

```json
{ "type": "chat.message",
  "payload": { "id": "01J9...", "room_id": 1, "user_id": 7,
               "display_name": "viewer7", "content": "안녕하세요",
               "created_at": "2026-06-04T12:00:00Z" } }

{ "type": "moderation.hide",
  "payload": { "id": "01J9...", "action": "blur" } }   // action: "blur" | "delete"

{ "type": "presence.update",
  "payload": { "room_id": 1, "count": 3,
               "members": [ { "user_id": 7, "display_name": "viewer7" } ] } }

{ "type": "room.freeze",
  "payload": { "room_id": 1, "frozen": true } }   // frozen: true | false
```

- 🟢 **상태 매핑**: `action:"blur" ↔ messages.status=BLURRED`, `action:"delete" ↔ DELETED`. `VISIBLE`은 hide 이벤트 없음. 클라이언트는 `messages.status`를 렌더링 단일 진실원으로 사용(D14). **v1에서 worker는 `blur`만 발행**(A-4 step4) — `delete`/`DELETED`는 스키마상 예약(미사용, 향후/수동 모더레이션용). (결정 기록 D19)
- 🟢 **presence / freeze (D44·D45)**: `presence.update` = 방 참여자 수·목록(접속/해제 시 갱신). `room.freeze` = 방 채팅 일시중지 토글(`frozen: true|false`). **신규 접속자는 핸드셰이크 바인딩 직후 현재 presence 스냅샷 + 현재 freeze 상태를 1회 받고**, 이후엔 변동분 이벤트만 받는다.

**Redis Pub/Sub**

- 🟢 채널 = `room:{room_id}`. Chat Server들이 `chat.message`·`moderation.hide`·`presence.update`·`room.freeze` 봉투를 publish, 모든 Chat Server가 subscribe하여 해당 방 클라이언트에 forward(presence·freeze는 forward와 동시에 로컬 상태도 갱신).
- 🟢 **전역 설정 무효화 채널 = `config:banned-words` (D42).** 어드민이 금칙어 변경 시 신호 publish → 전 Chat Server가 DB에서 캐시 재로드.
- 🟢 **presence/freeze 상태 키 (D44·D45)**: `room:{id}:members`(Hash, `user_id → display_name`; 수=HLEN·목록=HGETALL), `room:{id}:frozen`(flag). DB 아닌 Redis 전용(휘발성 운영 상태).

**검열 큐**

- 🟢 **큐 구현 = Redis List** (`mod:queue`, LPUSH/BRPOP) + KEDA Redis Lists 스케일러. 견고성 필요 시 Streams/SQS로 확장(스트레치). (근거·대안: 결정 기록 D1)
- job 포맷:

```json
{
  "message_id": "01J9...",
  "room_id": 1,
  "content": "안녕하세요",
  "enqueued_at": "2026-06-04T12:00:00Z"
}
```

- 🟢 **AI 큐 적재 = 1차 통과한 모든 메시지.** (명백한 욕설은 1차에서 즉시 차단, 나머지는 AI 비동기 확인.) 큐 부하가 충분해 오토스케일 데모에 유리. (근거·대안: 결정 기록 D2)

**REST API (최소)**

| 메서드 | 경로                                            | 용도                                                                                 |
| ------ | ----------------------------------------------- | ------------------------------------------------------------------------------------ |
| POST   | /api/login                                      | username+password 로그인 → user_id, display_name, **role**, 액세스 JWT (실패 시 401) |
| GET    | /api/rooms                                      | 방 목록                                                                              |
| GET    | /api/rooms/{id}/messages?before={ULID}&limit=50 | 히스토리 (DELETED 제외)                                                              |
| GET    | /actuator/health                                | 헬스체크                                                                             |
| GET    | /actuator/prometheus                            | 메트릭                                                                               |

**REST 응답 — message object** (`GET /api/rooms/{id}/messages` 항목)

```json
{
  "id": "01J9...",
  "room_id": 1,
  "user_id": 7,
  "display_name": "viewer7",
  "content": "안녕하세요",
  "status": "VISIBLE",
  "created_at": "2026-06-04T12:00:00Z"
} // status: "VISIBLE" | "BLURRED"
```

- 🟢 `status` 포함(재접속 캐치업이 의존). DELETED는 목록에서 제외(위 표). **재접속 캐치업(D14)** = 해당 윈도우는 **서버 히스토리가 진실원**: 로컬에 있으나 히스토리에 없는 id는 삭제로 간주해 제거하고, `status`는 히스토리 값으로 덮어쓴다(append-only 아님 — D14 채택안과 일치). **캐치업 윈도우 = `limit=50`(before 미지정) 최신 50건이며, reconcile은 이 범위 내 id에만 적용한다(D27).** (결정 기록 D20·D27)

**어드민 REST API (D46 — 전부 `role=ADMIN` 필요, 미달 시 403 `FORBIDDEN`)**

| 메서드 | 경로                                                                     | 용도                                                                                                                                       |
| ------ | ------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------ |
| GET    | /api/admin/keywords                                                      | 금칙어 목록                                                                                                                                |
| POST   | /api/admin/keywords                                                      | 추가 `{ word }` → DB insert + `config:banned-words` 무효화 publish                                                                         |
| DELETE | /api/admin/keywords/{id}                                                 | 삭제 → DB delete + 무효화 publish                                                                                                          |
| GET    | /api/admin/moderation-logs?stage={KEYWORD\|AI\|all}&before={id}&limit=50 | 모더레이션 로그 뷰. content = `COALESCE(moderation_logs.content, messages.content)`(AI 행은 messages join, 키워드 행은 logs.content — D46) |
| GET    | /api/admin/stats                                                         | 대시보드 집계(총 메시지·1차 차단·2차 블러·차단율, DB COUNT). "최근 검열"은 `moderation-logs?limit=5` 재사용                                |
| POST   | /api/admin/rooms/{id}/freeze                                             | `{ frozen: true\|false }` → `room:{id}:frozen` 기록 + `room.freeze` publish (D45)                                                          |

**에러 계약 (D29·D30)**

REST — 공통 에러 봉투 (모든 4xx/5xx):

```json
{ "error": { "code": "ROOM_NOT_FOUND", "message": "존재하지 않는 방입니다" } }
```

| HTTP | code              | 발생                                                                        |
| ---- | ----------------- | --------------------------------------------------------------------------- |
| 400  | `INVALID_REQUEST` | 필드 누락·형식 오류(login username 공백 등)                                 |
| 401  | `UNAUTHORIZED`    | 토큰 없음·무효·만료, **로그인 실패(미존재 username·비밀번호 불일치 — D40)** |
| 403  | `FORBIDDEN`       | `role≠ADMIN`이 `/api/admin/*` 호출 (D46)                                    |
| 404  | `ROOM_NOT_FOUND`  | 존재하지 않는 room_id                                                       |
| 500  | `INTERNAL`        | 서버 내부 오류                                                              |

WS — 핸드셰이크 단계 (연결 수립 전):

- 토큰 무효 → 업그레이드 거부 **HTTP 401**
- room_id 없음/미존재 → 업그레이드 거부 **HTTP 404**
- 연결 수립 후에는 인증 실패 경로 없음 — 토큰은 핸드셰이크에서만 검증, **연결 중 만료는 무시**(재연결 시 새 토큰 필요. 액세스 전용 JWT의 알려진 트레이드오프 — D4·D29)

WS — 연결 후 in-band `error` 이벤트 (연결은 유지):

```json
{
  "type": "error",
  "payload": {
    "code": "MESSAGE_BLOCKED",
    "message": "금칙어가 포함되어 있습니다"
  }
}
```

| code              | 발생                                        | 클라 처리               |
| ----------------- | ------------------------------------------- | ----------------------- |
| `MESSAGE_BLOCKED` | 1차 키워드 차단(발신자에게만 — D30)         | 입력창 옆 "차단됨" 표시 |
| `CHAT_FROZEN`     | freeze된 방에 chat.send(발신자에게만 — D45) | "채팅 일시중지됨" 표시  |
| `ROOM_MISMATCH`   | chat.send.room_id ≠ 세션 바인딩 방(D26)     | 재접속 유도             |
| `INVALID_PAYLOAD` | content 비어있음·**500자 초과**·스키마 위반 | 입력 검증 안내          |
| `INTERNAL`        | 저장·적재 실패(D28의 LPUSH 실패 포함)       | "전송 실패, 다시 시도"  |

WS — close code (이 둘만 사용):

| code | 의미                             | 클라 처리                       |
| ---- | -------------------------------- | ------------------------------- |
| 1001 | 서버 드레인(scale-in·배포 — D15) | 즉시 자동 재연결(D14)           |
| 1008 | 프로토콜·인증 위반               | 재연결하지 않고 로그인 화면으로 |

- 🟢 **ack 정책**: `chat.send` 성공 시 별도 ack 없음 — 구독 경로의 self-echo(D23)가 도착 확인 역할. 실패 시에만 `error`. (결정 기록 D29)

### A-4. 처리 흐름 (시퀀스 요약)

1. Client `chat.send` → Chat Server
2. Chat Server: **freeze 검사(D45)** — frozen이면 저장·큐·전파 없이 **발신자에게 `error(CHAT_FROZEN)` 회신**(끝). 아니면 → ULID 발급 → 1차 키워드 검열(인메모리 캐시에서 읽음 — DB 조회 없음, D42)
   - **차단**: `moderation_logs(KEYWORD, BLOCK, content=원문)` 기록(D25) → **발신자에게 `error(MESSAGE_BLOCKED)` 회신(D30)**, 타 클라이언트 전파 없음 (끝) — messages 미저장
   - **통과**: MySQL 저장(status=VISIBLE) → **`mod:queue`에 job LPUSH → `room:{id}`에 `chat.message` publish** (순서 고정 — D28)
     - LPUSH 실패: publish 하지 않고 `error(INTERNAL)` 응답 → 검열 안 된 노출 차단, 클라 재시도 가능
     - publish 실패: 메시지는 저장·큐에 존재(히스토리에서만 보임) — 드문 케이스로 허용
3. 모든 Chat Server: 구독 채널에서 `chat.message` 수신 → 해당 방 클라이언트에 push (**즉시 노출**). **발신 서버도 로컬에서 직접 push하지 않고 publish만 하며, 자기 클라 포함 모든 push는 이 구독 경로에서 일어난다**(self-receive 포함 → 이중 푸시 방지). (결정 기록 D23)
4. Worker: 큐 소비 → 모델 판정(in-process) → **카테고리별 정적 임계값 정책 적용(D43)** → `moderation_logs(AI, score, ...)` 기록
   - **BLOCK**(어느 보호 카테고리든 `score[cat] ≥ threshold[cat]` — 혐오 엄격 / 욕설 관대): `messages.status = BLURRED` → `room:{id}`에 `moderation.hide` publish → 클라이언트 사후 블러 (**v1은 `blur`만** — A-3 매핑)
5. KEDA: `mod:queue` 길이 감시 → 워커 수 조절
6. **presence(D44)**: WS 접속(바인딩 후) → `HSET room:{id}:members` + `presence.update` publish + 신규 클라에 스냅샷. 해제 → `HDEL` + `presence.update` publish.
7. **freeze(D45)**: 어드민 `POST .../freeze` → `room:{id}:frozen` 기록 + `room.freeze` publish → 전 파드 로컬 플래그 갱신 + 클라 배너. (step 2의 freeze 검사가 이 플래그를 읽음)
8. **금칙어 동기화(D42)**: Chat Server 부팅 시 DB 로드 → 인메모리 캐시. `config:banned-words` 신호 수신 시 DB 재로드.

### A-5. 런타임 계약 (포트 · env · 네임스페이스 · 헬스)

🟢 app↔config 경계를 못 박는다. config(매니페스트)·infra가 이 값을 그대로 참조. (결정 기록 D24)

| 컴포넌트          | 포트 | 헬스/메트릭                                             | 비고                                                                         |
| ----------------- | ---- | ------------------------------------------------------- | ---------------------------------------------------------------------------- |
| Chat Server       | 8080 | `/actuator/health` · `/actuator/prometheus` (동일 포트) | WS+REST 한 앱                                                                |
| Moderation Worker | 8000 | `/metrics`                                              | prometheus_client · liveness/readiness 프로브 겸용(HTTP 200 = 프로세스 생존) |

- 🟢 **네임스페이스 = `chatguard`** (dev/prod는 별도 클러스터라 환경 구분 불필요). 슬러그 `chatguard`는 AWS 리소스 이름엔 안 들어가지만 k8s 내부 논리 이름엔 사용 — C-2 prefix(`team1-{env}-`)는 공유 계정의 AWS 리소스 충돌 방지용이라 클러스터 내부 네임스페이스엔 강제되지 않음.
- 🟢 **env 변수 계약** (ESO가 동기화하는 K8s Secret 키 이름 = 아래 `(secret)` 표시 env명과 일치):

```
Chat Server
  DB_URL, DB_USER, DB_PASSWORD(secret),
  REDIS_HOST, REDIS_PORT,
  JWT_SECRET(secret),
  MOD_QUEUE_KEY=mod:queue, ROOM_CHANNEL_PREFIX=room:, CONFIG_CHANNEL_PREFIX=config:
Moderation Worker
  DB_URL, DB_USER, DB_PASSWORD(secret),
  REDIS_HOST, REDIS_PORT,
  MOD_QUEUE_KEY=mod:queue, MODEL_VERSION
```

- 큐 키(`mod:queue`)·채널 패턴(`room:{id}`)은 A-3 확정값과 동일. 시크릿 원문은 Secrets Manager에만(코드·state 금지, CLAUDE.md §5).
- **app 고도화 추가(D41~D45)**: 어드민 인가는 `users.role`(DB)·JWT 클레임으로 처리 → **신규 시크릿·허용목록 env 없음**. presence·freeze 키(`room:{id}:members`·`room:{id}:frozen`)는 `room:` prefix에서 파생되어 신규 env 불필요. 신규는 무효화 채널 prefix(`config:`)뿐(비밀 아님·git 안정값).
- **경로 라우팅·WS 유지(D47)**: 외부 진입은 ALB Ingress 단일 호스트 — `/api`·`/ws`→Chat Server(8080), `/`→Frontend(80). nginx는 정적 전용(프록시 제거). WS 장시간 유지 위해 ALB `idle_timeout.timeout_seconds`를 충분히 연장(기존 nginx `proxy_read_timeout 3600s` 대체) + 서버측 ping heartbeat(idle·좀비 연결 대비, 클라 무변경). 신규 시크릿·env 없음(평문 HTTP 유지 — TLS·ACM·도메인 불필요; https 고도화는 D48 보류 시점에 별도).

---

## B. 관측성 · SLO · 메트릭 계획

### B-1. SLI / SLO

> 골든 시그널(지연·트래픽·오류·포화)을 재료로, 사용자 체감 지표 3개를 SLI로 정한다.

| SLI              | 정의                                    | SLO (출발값)                 | 측정        |
| ---------------- | --------------------------------------- | ---------------------------- | ----------- |
| 채팅 전송 성공률 | 성공 broadcast / 전체 chat.send         | ≥ 99.9% (월 에러버짓 ≈ 43분) | Chat Server |
| 채팅 전송 지연   | chat.send 수신 → broadcast 푸시         | p95 < 300ms                  | Chat Server |
| 검열 반영 지연   | 큐 적재 → moderation.hide 발행(차단 건) | p95 < 5s                     | Worker      |

- 🟢 **검열 반영 지연이 핵심 SLI.** 오토스케일이 "지키는" 지표 — 폭주로 큐가 쌓이면 이 지연이 오르고, KEDA가 워커를 늘려 다시 끌어내림. 데모·발표의 중심.
- 에러 버짓: 99.9% = 0.1% = 월 ≈ 43분. Grafana에 잔여 버짓·번레이트 패널.
- dev/prod 차이(가이드라인 18-3): dev = 기본 대시보드·retention 7일·dev 채널 / prod = SLO 알람·retention 30일+·Runbook URL·prod 채널.
- 🟢 위 SLO 수치는 **출발값으로 합의.** 1차 부하테스트 측정 후 재확정. (결정 기록 D5)

### B-2. 서비스별 노출 메트릭

**Chat Server** (Spring Boot · Micrometer/MeterRegistry → `/actuator/prometheus`)

- `http_server_requests_*` — REST RED(요청률·오류·지연), 자동 수집
- `ws_active_connections` (gauge) — **스케일 ① 신호** + 포화
- `chat_messages_total{result=passed|blocked_keyword}` (counter) — 트래픽 + 1차 차단율
- `chat_broadcast_latency_seconds` (histogram) — 전송 지연 SLI
- JVM(heap·GC) — 포화, 자동 수집

**Moderation Worker** (Python · prometheus_client → `/metrics`)

- `moderation_jobs_total{verdict=pass|block}` (counter) — 트래픽 + AI 차단율
- `moderation_inference_seconds` (histogram) — 모델 추론 시간
- `moderation_queue_wait_seconds` (histogram) — 큐 대기(적재→픽업), 부하 시 상승
- `moderation_e2e_seconds` (histogram) — 적재→hide 발행(차단 경로에서만 기록), **검열 반영 지연 SLI**

**Redis 큐 깊이**

- `redis_exporter`로 `mod:queue` 길이(LLEN) 노출 → **스케일 ② 신호** 시각화. (KEDA는 Redis에서 직접 읽음)

**클러스터** (kube-prometheus-stack 기본 제공)

- kube-state-metrics: **워커·Chat Server 레플리카 수** ← 오토스케일을 눈으로 보여주는 핵심
- node-exporter: 노드 CPU/메모리(포화)

### B-3. 대시보드 구성

🟢 코어 2개 + 권장 1개:

1. **오토스케일 스토리(헤드라인)** — 한 타임라인에 `mod:queue 깊이` + `워커 레플리카 수` + `검열 반영 지연`을 겹쳐 표시 → "큐 상승 → 스케일 → 지연 회복" 한눈에. (스케일 ①용으로 접속 수 + Chat Server 레플리카도)
2. **서비스 골든 시그널** — 지연(전송·검열) / 트래픽(메시지율) / 오류(5xx·실패) / 포화(접속 수·큐 깊이·CPU). Grafana 기성 "Golden Signals" 템플릿 활용.
3. (권장) **SLO·에러버짓** — SLI 현재값 vs SLO, 잔여 버짓, 번레이트.

- 🟢 **검열 품질 대시보드(차단율·AI 점수 분포) 데이터 = Prometheus 카운터**(워커가 노출, MeterRegistry/prometheus_client 패턴). 개별 메시지 드릴다운이 필요하면 Grafana→MySQL 직접 조회 추가(스트레치). (결정 기록 D7)

### B-4. 알람 (Alertmanager)

🟢 과민하지 않게 소수만:

- `HighChatErrorRate` — 전송 오류율 > 1%, 5분 (warning) → 가용성 SLO 연동
- `ModerationLatencyHigh` — 검열 반영 p95 > SLO, 5분 → "오토스케일이 못 따라간" 신호
- `PodCrashLooping` / `TargetDown` — kube-prometheus-stack 기본
- (스트레치) `ErrorBudgetFastBurn` — 다중 윈도 번레이트
- 🟢 **알림 채널 = Slack** (dev/prod 채널 분리). Webhook URL은 Secret으로 관리(커밋 금지). (결정 기록 D6)

### B-5. 구현 메모

- kube-prometheus-stack가 Prometheus·Grafana·Alertmanager·node-exporter·kube-state-metrics를 한 번에 제공.
- 각 서비스에 **ServiceMonitor**로 스크레이프 등록. ⚠️ ServiceMonitor의 `release` 라벨이 차트 라벨과 일치해야 동작(가이드라인 12-7).
- 큐 깊이 시각화용 `redis_exporter` 설치(Helm).
- 대시보드·ServiceMonitor·PrometheusRule·알람은 **GitOps(config repo)** 관리 (Terraform 아님 — A-1 경계).
- SLI 알람(PrometheusRule) 작성 시 `release` 라벨 일치 주의 (가이드라인 18-4 예시 참고).
- **[Week4] scale-in 관찰** — 별도 대시보드 불필요. 헤드라인(B-3 ①: 큐 깊이 + 워커 레플리카 + 검열 반영 지연)의 **우측 회복·축소 구간**과 Chat Server 패널(연결 수 + 레플리카 수)에서 그대로 읽힌다. 시연은 사이클을 미리 돌려 Grafana 시간범위로 펼쳐 보기(결정 D17). 상세: 결정 D11~D18 / `scale-in 설계노트`.

---

## C. 네이밍 · 태그 · 리포 · state 규약

### C-1. 리포 구조 (3-repo)

🟢 가이드라인 15 준수.

- `team1-chatguard-app` — 앱 소스(React, Spring Boot, Python worker), Dockerfile, CI
- `team1-chatguard-config` — k8s 매니페스트(overlays), ArgoCD Application/ApplicationSet
- `team1-chatguard-infra` — Terraform(modules + envs)
- 참고: 서비스 슬러그 `chatguard`는 **리포 이름에만** 사용. 리소스 prefix엔 미포함.

**원칙 — 환경은 브랜치가 아니다:**

- 브랜치 = 코드 성숙도(feature → main). 환경 = 디렉터리/오버레이/이미지 태그.
- app: main + feature. 환경 구분은 이미지 태그(git SHA).
- config: main 단일. `overlays/dev`, `overlays/prod`.
- infra: main 단일. `envs/dev`, `envs/prod`.
- dev/prod 브랜치 안 만듦(드리프트·충돌 방지).

### C-2. 네이밍 · 태그

🟢 가이드라인 04 준수.

- 표준 태그(default_tags 자동 부착): `Team=team1` / `Environment` / `Project=chatguard` / `Owner`. 키는 PascalCase.
- prefix 두 부류:
  - 이름이 유일해야 하는 것(EKS Cluster, Node Group, ECR, RDS, ElastiCache, S3) → 이름에 `team1-{env}-` 강제. 예: `team1-dev-api-server`, `team1-prod-mysql`. (프로젝트 슬러그는 리소스 이름엔 안 들어감) **컴포넌트별 ECR 이름은 역할 기반으로 확정: `team1-dev-api-server`(Chat Server)·`team1-dev-ai-worker`(Worker)·`team1-dev-frontend`(Frontend) — D36.**
  - ID로 식별되는 것(VPC, Subnet, NAT, IGW, SG) → `Name` 태그 + `Team=team1` 태그.
- 🟢 team1 / chatguard / VPC CIDR 10.1.0.0/16 확정. (결정 기록 D8)

### C-3. dev / prod tfvars 차이

🟢 같은 모듈, 변수로 차이. 핵심은 비용 vs 안정성.

| 항목                    | dev                     | prod                   |
| ----------------------- | ----------------------- | ---------------------- |
| name_prefix             | team1-dev               | team1-prod             |
| VPC CIDR                | 10.1.0.0/17             | 10.1.128.0/17          |
| AZ 수                   | 2                       | 3                      |
| NAT Gateway             | 1 (비용)                | AZ별 다중(HA)          |
| EKS 노드                | 2 × t3.medium           | 3 × t3.large           |
| EKS endpoint            | public + private        | private 또는 IP 제한   |
| RDS                     | db.t4g.micro, Single-AZ | db.t4g.small, Multi-AZ |
| RDS deletion_protection | false                   | true                   |
| RDS skip_final_snapshot | true                    | false                  |
| Backup retention        | 1일                     | 7일                    |
| Redis                   | cache.t4g.micro         | cache.t4g.small        |
| 관측성 retention        | 7일                     | 30일+                  |
| 알람 채널               | dev                     | prod                   |

- 🟢 prod = **HA 구조(Multi-AZ RDS·다중 NAT·노드 AZ 분산)는 유지, 인스턴스 크기는 최소.** (결정 기록 D9)

### C-4. state 관리

🟢 가이드라인 08 준수.

- 백엔드: S3 + 네이티브 락(use_lockfile, DynamoDB 미사용).
- (환경 × 레이어)마다 별도 state. 경로:
  - `team1/dev/infra/terraform.tfstate`
  - `team1/dev/platform-addons/terraform.tfstate`
  - `team1/prod/infra/...` · `team1/prod/platform-addons/...`
- backend 블록은 변수 불가 → 루트별 static `backend.tf` 또는 `backend-config`.
- `.terraform.lock.hcl` 커밋(재현성).

---

## D. 부하테스트 계획

### D-1. 원칙: 변수 격리

🟢 두 부하 축(접속 vs 검열 처리량)은 병목 특성이 다르므로 **한 번에 하나씩** 측정. 한 시나리오에서 둘 다 극한으로 밀지 않는다(측정이 흐려짐).

### D-2. 시나리오

| 실험                    | 자극                  | 관찰(스케일)            | 핵심 메트릭                  |
| ----------------------- | --------------------- | ----------------------- | ---------------------------- |
| A. 접속 계층            | WebSocket 연결 램프업 | Chat Server 레플리카(①) | 접속 수, 전송 지연           |
| B. 검열 워커 (헤드라인) | 메시지 처리량 폭주    | 워커 레플리카(②, KEDA)  | 큐 깊이, 검열 반영 지연      |
| (서사) 경기일 통합      | 접속 + 채팅 동시 폭주 | 둘 다                   | 발표 데모용. 본 측정은 A·B로 |

- B 실험 시 **접속 계층이 병목 안 되게** 넉넉히 provision 또는 사전 스케일. (워커 포화는 접속 수가 아닌 메시지 처리량에 좌우)

### D-3. k6 프로파일

🟢 단계별:

- smoke — 최소 부하로 동작 확인
- load — 평상 수준 지속
- spike — 급격 폭주(경기일 모사) → 오토스케일 트리거
- soak — 중간 부하 장시간(누수·안정성)
- thresholds 예: `http_req_duration p95 < 300ms`, WS 연결 성공률 > 99%, `checks` 통과율.

### D-4. 사전 스케일 + 반응 스케일

🟢 둘 다 시연:

- 사전(scheduled) — 경기 시작 전 워커/노드 예열 (예측 가능한 폭주 대비)
- 반응(KEDA) — 큐 깊이 기반 자동 확장 (예측 못 한 폭주 대비)

### D-5. 측정 · 판정 · 기록

- SLO 대비: 전송 지연 p95, **검열 반영 지연 p95**(핵심), 오류율.
- 스케일 동작: 워커 2 → N 확장 시간, 큐 적체 해소 시간, 부하 후 축소.
- 기록: 측정 숫자를 섹션 F(스파이크 결과)와 발표 자료에 ("2→12 워커 90초, 검열 지연 X초 회복").
- 🟢 방침: **절대 수치보다 "스케일 동작·회복" 증명 우선.** 목표 부하 규모·k6 위치는 1차 부하테스트로 확정. 부하 생성기는 대상 클러스터와 분리. (결정 기록 D10)
- **[Week4] 부하 후 축소(scale-in)의 graceful 동작 검증** — 워커 graceful shutdown(in-flight job 무손실)·Chat Server graceful drain(연결 단절 시 클라 재접속)도 이 단계에서 함께 확인. pod당 연결 **cap 값**은 여기서 측정해 확정(파드 메모리 ÷ 연결당 발자국 × 여유 — D18·D10 연계). 상세: 결정 D11~D18.

---

## E. 중간점검(6/17) Done 정의

> 6/17 중간점검에서 "무엇이 동작함을 보여줄지"의 합의. 이 목표를 기준으로 역산해 일한다. 가이드라인 21(Week 2 = CI/CD + 중간점검 통과)에 맞춤. 🟢 팀 확인 완료 · 동결(2026-06-10).

### E-1. 목표 한 줄

dev 환경에서 **"채팅이 흐르고, 파이프라인이 돌고, 위험한 가정이 깨졌음"**을 end-to-end로 증명. (완성도가 아니라 토대의 동작 증명이 목표 — 풀 데모·관측성·prod는 3~4주차.)

### E-2. Done 체크리스트

**반드시 (토대):**

1. dev 인프라 `terraform apply` 성공 + `kubectl`로 EKS 접속 확인 (위험 제거 1)
2. 채팅 워킹 스켈레톤 — React 입력 → Chat Server(WebSocket) → Redis Pub/Sub → 다른 클라이언트 표시. **멀티 파드 fan-out 동작**(위험 제거 2). 1차 키워드 검열 즉시 차단 동작.
3. CI/CD 한 바퀴 — app main merge → GitHub Actions(OIDC) → ECR push → config repo 이미지 태그 갱신 → ArgoCD가 dev에 배포 (가이드라인 16)
4. 관측성 최소 — Prometheus/Grafana 기동, 최소 한 서비스 스크레이프 + 기본 패널(접속 수 또는 큐 깊이)

**동작 증명 (스파이크 수준, 폭은 작아도 — 위험 제거):** 5. AI 검열 워커가 큐 job을 꺼내 판정 → `moderation.hide` 전파 → 사후 블러 1회 (모델 CPU 서빙 스파이크 통과 — 위험 제거 3) 6. KEDA가 큐 깊이로 워커 1→N 1회 확장 확인 (오토스케일 스파이크 통과 — 위험 제거 4)

### E-3. 시연 시나리오 (그날)

1. `kubectl get nodes/pods`로 dev 클러스터 접속 보여주기
2. 두 브라우저로 실시간 채팅 + 금칙어 즉시 차단 1회
3. 악성 채팅 1개 → 잠시 후 사후 블러 (검열 흐름)
4. 코드 한 줄 변경 push → 파이프라인 → ArgoCD dev 반영 (라이브 또는 녹화)
5. Grafana 기본 대시보드 한 화면
6. k6로 큐 부하 → 워커 1→N 늘어남 1회 (스파이크)

### E-4. 6/17엔 기대하지 않음 (3~4주차로)

prod 환경, 풀 부하·오토스케일 데모 완성, SLO/에러버짓 대시보드, OpenTelemetry, 우회형 탐지 개선, hardening 다수, 사후 블러 UX 폴리시.

- **scale-in(graceful 축소·재접속)** — Week 4 산출물. **워커 scale-in(T1)·Chat Server 인프라 설정(LOR·scaleDown behavior·PDB·preStop, T2)은 커밋**, **클라 재접속·캐치업·1001 close(T3)는 스트레치.** 6/17은 **scale-out 1회**(E-2 #6)로 충분하며 scale-in은 중간점검 대상이 아님. (결정 D11~D18 / `scale-in 설계노트`)

### E-5. 역산 일정 (대략)

- ~Week1 끝(≈6/12): dev 인프라 apply + 접속, 채팅 스켈레톤 동작
- Week2(6/15~16): CI/CD 연결, ArgoCD dev 배포, 관측성 기동, 검열·KEDA 스파이크 통과
- 6/16: 리허설·시연 점검
- 6/17: 중간점검

---

## 결정 기록 (Decision Log)

> ADR(아키텍처 결정 기록) 경량판.
> 각 결정의 채택안·검토한 대안·근거를 남긴다. (A·B·C·D: 2026-06-04 동결)

| ID  | 결정                      | 채택                                                                                                              | 검토한 대안                                                             | 근거                                                                                                                                                        |
| --- | ------------------------- | ----------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| D1  | 검열 큐 구현              | Redis List                                                                                                        | Redis Streams(ack·at-least-once), SQS(완전관리형·DLQ·Pod Identity 활용) | fan-out용 Redis가 이미 필수 → 새 인프라 0. 검열 일감은 1차 통과분이라 드문 유실이 치명적이지 않음. 단순함·빠른 착수 우선. 견고성 필요 시 Streams/SQS로 확장 |
| D2  | AI 큐 적재 범위           | 1차 통과한 모든 메시지                                                                                            | 샘플링, 위험 기반 선별                                                  | 큐 부하가 충분해 오토스케일 데모에 유리. 미탐(키워드가 놓친 우회형) 커버 최대화. 비용 최적화는 본 프로젝트 목적 아님                                        |
| D3  | Worker 모델 호스팅        | in-process(워커 내 로드)                                                                                          | 별도 모델 서빙 파드(FastAPI/KServe)                                     | 작은 CPU 모델이라 단순. 스케일 단위가 워커 하나로 명확해 KEDA 데모가 깔끔. 대형/GPU 모델이면 분리가 정석                                                    |
| D4  | 인증 깊이                 | ~~닉네임 로그인 + 액세스 전용 JWT (Spring Security 미사용)~~ **→ D40으로 번복: username+password + BCrypt 검증**  | 무인증, id/pw + JWT 풀 Spring Security(access+refresh)                  | (당초) 인증은 프로젝트 주제가 아님, 신원은 가볍게. **(번복) 멘토 보안 피드백 반영 — password 검증은 비용이 작고 보안 강화가 큼. 상세 D40**                  |
| D5  | SLO 목표치                | 가용성 99.9% / 전송 p95 300ms / 검열 반영 p95 5s **(출발값)**                                                     | 더 느슨·엄격한 값                                                       | 데모에서 달성·관찰 가능한 현실적 기준. **1차 부하테스트 후 수치 재확정**                                                                                    |
| D6  | 알림 채널                 | Slack (dev/prod 채널 분리)                                                                                        | 웹훅·이메일·UI 발화만                                                   | 가이드라인 권장 패턴, 데모 가시성 좋음. Webhook URL은 Secret 관리                                                                                           |
| D7  | 검열 품질 대시보드 데이터 | Prometheus 카운터                                                                                                 | Grafana→MySQL 직접 조회(스트레치)                                       | 단순·추가 데이터소스 불필요. 집계가 관측성 핵심. 개별 메시지 드릴다운 원하면 MySQL 추가                                                                     |
| D8  | 네이밍·CIDR               | prefix `team1-{env}-` · 서비스 `chatguard`(리포명) · VPC `10.1.0.0/16` → dev `10.1.0.0/17` · prod `10.1.128.0/17` | (CIDR 운영진 배정 확인 완료)                                            | 공유 계정 충돌 방지(가이드라인 04·06). /17 반분할은 비겹침·정렬 정확, EKS Pod IP 수요(/20+)에도 여유                                                        |
| D9  | prod 인프라 사양          | HA 구조 + 최소 인스턴스(노드 t3.large×3, db.t4g.small Multi-AZ)                                                   | 가이드라인 예시(m6i.large 등) 그대로                                    | 데모 목적이라 과한 사양 불필요. 평가는 설계(HA·변수화)지 용량이 아님. 작은 인스턴스가 오토스케일 데모에도 유리                                              |
| D10 | 부하테스트 규모           | 스케일 동작·회복 증명 우선, 수치는 부하테스트로 확정                                                              | 대규모(수만 연결) 고정                                                  | 부트캠프 예산 한계. 올바른 신호 기반 스케일·회복이 포트폴리오 핵심. 부하 생성기는 대상과 분리                                                               |

### 결정 기록 — scale-in (2026-06-10 추가)

> v1 동결(2026-06-04, A·B·C·D) 이후 추가된 애드엔다. 모두 🟢 **설계 확정 · 구현 Week 4(중간점검 후).** 6/17 중간점검 의무 아님(E-2는 scale-out 1회로 충분). 각 행의 `[T1/T2/T3]`은 난이도 티어 — **T1(워커)·T2(Chat Server 인프라 설정)는 커밋, T3(클라 앱코드)는 스트레치.** 상세 개념·다이어그램은 학습 문서 `scale-in 설계노트` 참조.

| ID  | 결정                             | 채택                                                                                                                                 | 검토한 대안                                          | 근거                                                                                                                                                                                              |
| --- | -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| D11 | 워커 in-flight 보호 `[T1]`       | SIGTERM graceful shutdown(새 pop 중단 → 현재 job 마무리 → exit), 한 번에 1개 pop, grace ≥ 최대 추론 시간                             | 유실 허용 / 큐를 Streams·SQS로(ack·visibility)       | BRPOP은 ack가 없어 SIGKILL 시 일감 증발 = 검열 구멍(악성 메시지 미차단 잔존). D1(Redis List)을 안 바꾸고 막는 최소 비용                                                                           |
| D12 | 워커 스케일 방향 `[T1]`          | 비대칭 — 확장 빠름(stab 0), 축소 느림(stab 120~300s)                                                                                 | 대칭 스케일                                          | in-process 콜드스타트(모델 로딩) 반복 방지 + 검열 반영 지연 SLI 안정                                                                                                                              |
| D13 | 워커 바닥 `[T1]`                 | `minReplicaCount` 1+(dev 1 · prod 2), SLI 경로 scale-to-zero 미사용                                                                  | scale-to-zero(0)                                     | 0→1 콜드스타트가 첫 메시지 SLI(5s)를 깸. 유휴 워커 1개 비용 사소. prod는 HA(D9 정신)                                                                                                              |
| D14 | Chat Server 클라 재접속 `[T3]`   | 자동 재연결(a) + jittered backoff(b) + 재접속 시 최근 윈도우 id 병합 캐치업(c), 검열 상태 단일화(`message.status`, overlay map 제거) | 무재연결 / append-only 캐치업 / overlay map 유지     | scale-in 안전성은 클라 재접속 견고성이 결정. 워커가 DB status를 hide 발행보다 먼저 기록(A-4)하므로 status가 안전한 단일 진실원. (a)는 서비스 기본기라 사실상 baseline                             |
| D15 | Chat Server 서버 drain `[T2/T3]` | preStop `sleep` → SIGTERM 1001 close → 넉넉한 grace, ALB deregistration delay 짧게                                                   | SIGKILL 방치(TCP timeout) / readiness fail 방식      | 명부 전파(트랙①)와 앱 종료(트랙②)의 race를 preStop sleep으로 해소. 능동 close라 dereg delay 길 필요 없음. (preStop·grace·dereg=설정 T2, 1001 close 핸들러=앱코드 T3)                              |
| D16 | Chat Server 축소 속도 `[T2]`     | 보수적 — scaleDown stab 600s, 120s당 1개 + PDB `maxUnavailable:1` 별도                                                               | 워커와 동일 수준 / PDB로 scale-in 제어               | 파드당 재접속 폭주·load-conserving 메트릭 특성상 워커보다 보수적이어야. PDB는 HPA 축소엔 무효(eviction만 적용) → 노드 정비 축만 담당                                                              |
| D17 | scale-in 시연 `[T1·T2]`          | Grafana 타임라인(미리 1회 돌려 시간범위로 펼쳐 보기) + 시연 영상 타임랩스                                                            | 발표장 라이브 대기 / 데모용 짧은 stab 프로파일(보조) | 보수적(느린) 축소를 라이브로 기다릴 수 없음. Prometheus retention 내 데이터를 시간범위 조회로 표시                                                                                                |
| D18 | Chat Server 연결 분산 `[T2]`     | ALB LOR(`least_outstanding_requests`) + pod당 연결 cap(안전 천장) + 기존 연결 강제 재분배 안 함                                      | 라운드로빈 / cap을 분산 수단으로 / 강제 1001 재분배  | 기존 연결은 파드 고정 → scale-out이 새 유입 흡수에 효과 내려면 새 연결을 빈 파드로(LOR). cap은 OOM 방지 천장이지 분산용 아님(스케일 목표를 cap 아래로). 강제 재분배는 멀쩡한 사용자 단절이라 과함 |

### 결정 기록 — 인터페이스 계약 (2026-06-10 추가)

> 병렬 작업(레포·담당자별 독립 AI)에서 통합 경계가 갈라지지 않도록 응용/운영 계약을 못 박은 결정들. A·B 본문에 반영됨.

| ID  | 결정                      | 채택                                                                                      | 검토한 대안                                                | 근거                                                                                                                                                    |
| --- | ------------------------- | ----------------------------------------------------------------------------------------- | ---------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| D19 | 검열 상태 표현            | `messages.status` 단일 진실원 + `action↔status` 매핑, **v1은 `blur`만**                   | `moderation.hide.action`을 진실원으로 / `delete` 즉시 구현 | 워커가 DB status를 hide 발행보다 먼저 기록(A-4)하므로 status가 재접속에 안전한 단일원(D14). delete 트리거(수동 모더레이션 도구)는 범위 밖 → 스키마 예약 |
| D20 | REST 메시지 스키마·캐치업 | 응답에 `status` 포함 + 윈도우=서버 히스토리 진실원 reconcile(부재 id=삭제, status 덮어씀) | `status` 미포함 / append-only 캐치업                       | D14 재접속 견고성의 전제. 클라가 이전 블러/삭제를 일관되게 복원                                                                                         |
| D21 | WS 토큰 전달              | 접속 URL 쿼리파라미터 `?token=`                                                           | 서브프로토콜 / Authorization 헤더(브라우저 WS 불가)        | 데모 단순·구현 명확. 로그·URL 노출 회피 필요 시 서브프로토콜로 확장(스트레치)                                                                           |
| D22 | JWT 클레임·display_name   | 토큰에 `display_name` 포함 → 메시지당 DB 조회 회피                                        | 매 broadcast마다 DB 조회                                   | 전송 p95 300ms SLO 보호. 닉네임 변경은 v1 비범위라 토큰 캐시로 충분                                                                                     |
| D23 | self-echo                 | 구독 경로에서만 push(발신자 포함)                                                         | 로컬 즉시 push + self-receive skip                         | 단일 push 경로로 이중 노출 방지·구현 단순. Redis pub/sub는 self에도 전달                                                                                |
| D24 | 런타임 계약(포트·ns·env)  | A-5에 명문화(8080/8000 · ns `chatguard` · env 키 고정)                                    | 담당자별 임의 결정                                         | app↔config 병렬 작업의 결정성 확보. 매니페스트·Secret 키가 앱과 일치해야 부팅                                                                           |

### 결정 기록 — 인터페이스 계약 2차 · 에러 계약 (2026-06-10 합의)

> 구현 착수 직전 점검에서 발견된 계약 구멍 4건(D25~D28) + 에러 처리 계약(D29~D30). 모두 🟢 합의 완료, A 본문에 반영됨.

| ID  | 결정                      | 채택                                                                                                                                                                                                                                                                                                        | 검토한 대안                                                                               | 근거                                                                                                                                                                                           |
| --- | ------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| D25 | moderation_logs 정합성    | **FK 제거**(message_id는 ULID로 항상 기록, 인덱스 유지) + **`content TEXT NULL` 컬럼**(KEYWORD·BLOCK 행에서만 기록)                                                                                                                                                                                         | 차단 메시지도 messages에 저장(status에 REJECTED 추가) / 키워드 차단은 Prometheus 카운터만 | 키워드 차단 경로는 messages 미저장(A-4 step2)이라 기존 FK 스키마는 INSERT 실패. 차단 본문은 키워드 튜닝·감사·데모에 유용. messages 오염 없이 최소 변경. append-only 로그라 FK 강제 실익 적음   |
| D26 | WS 방 바인딩              | 접속 URL = **`?token=...&room_id=...`** → 핸드셰이크에서 방 존재 검증 후 **세션=방 1개 바인딩**. 방 이동=재접속. `chat.send.room_id` 불일치 시 `error(ROOM_MISMATCH)` 거부                                                                                                                                  | `room.join`/`room.leave` 이벤트(멀티룸)                                                   | 세션→방 매핑 규약이 없으면 fan-out 대상("해당 방 클라이언트")이 미정의 → 구현 갈라짐. 라이브 채팅 UX상 1세션 1방이 자연스럽고 D21과 일관. join/leave는 스트레치                                |
| D27 | 캐치업 윈도우 정의        | 윈도우 = **`GET /api/rooms/{id}/messages?limit=50`(before 미지정, 최신 50건)**. D20 reconcile은 이 범위 내 id에만 적용                                                                                                                                                                                      | `after={ULID}` 파라미터(증분 조회)                                                        | D14·D20이 reconcile 규칙은 정했으나 윈도우 미정의. 기존 엔드포인트로 충분, 50건 초과 단절은 전체 리로드와 동일해 자연 복구. `after`는 스트레치                                                 |
| D28 | 전파·적재 순서            | **저장 → `mod:queue` LPUSH → publish.** LPUSH 실패 시 publish 중단 + `error(INTERNAL)`(검열 구멍 차단). publish 실패는 히스토리만 존재 허용(드묾)                                                                                                                                                           | 현행(저장→publish→LPUSH) + 유실 허용 / 트랜잭셔널 아웃박스                                | 현행 순서는 LPUSH 실패 시 "노출됐지만 영원히 미검열" = 검열 서비스에 최악 실패모드. 순서 교체는 비용 0, 실패모드를 미전파(무해·재시도 가능)로 전환. 아웃박스는 v1에 과함                       |
| D29 | 에러 계약(REST·WS)        | REST 공통 에러 봉투 `{error:{code,message}}` + 코드표(A-3). WS 핸드셰이크 실패=업그레이드 거부(HTTP 401/404), 연결 후=in-band `error` 이벤트(연결 유지), close code는 **1001·1008만**. `chat.send` 성공 ack 없음(self-echo가 ack — D23), content **최대 500자**. 연결 중 토큰 만료는 무시(신규 연결만 검증) | 루트 평탄화 / 성공 ack 추가 / 만료 시 강제 close                                          | 에러 포맷·close code 미정의면 담당자별 구현이 갈라짐. ack는 self-echo와 중복. 연결 중 재인증은 v1 비범위(D4와 일관)                                                                            |
| D30 | 키워드 차단 발신자 피드백 | 차단 시 **발신자에게만 `error(MESSAGE_BLOCKED)` 회신** → 프론트 "차단됨" 표시. 타 클라이언트 전파 없음                                                                                                                                                                                                      | 무응답(shadow-block) / 별도 `chat.blocked` 타입                                           | 무응답이면 발신자 화면에서 메시지 증발 → E-3 시연 #2("금칙어 즉시 차단") 불가시. shadow-block은 우회 억제엔 유리하나 데모 가시성 우선(스트레치 토글 가능). `error` 봉투 재사용으로 타입 최소화 |

### 결정 기록 — 배포 자동화 (2026-06-15 추가)

> GitOps 기반 배포 자동화 세션에서 내린 결정들. app·config·infra 각 레포 PR로 반영됨.
> 구현 상태 표기 — ✅ 완료(머지·검증) · ⏳ 설계 확정·구현 후속. 상세 여정은 학습 문서 `배포 자동화 학습노트` 참조.
> 이 세션에서 dev EKS에 앱을 **수동 배포**해 채팅+AI검열 end-to-end 동작을 검증했고, 파이프라인 자동화(D33)·시크릿 자동화(D34 ESO)·자동 배포(ArgoCD)는 후속 작업으로 남았다.

| ID  | 결정                            | 채택                                                                                                                                                                                                                     | 검토한 대안                                            | 근거                                                                                                                                                                                                                                                    | 상태                                                        |
| --- | ------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------- |
| D31 | 컨테이너 이미지 빌드 표준       | 멀티스테이지(빌드/런타임 분리) + 비루트 USER + exec-form ENTRYPOINT, 이미지 빌드 시 테스트 스킵(`-DskipTests`)                                                                                                           | 단일 스테이지 / 빌드 내 테스트 실행                    | 런타임 이미지에서 빌드도구·소스 제거(크기·공격면). exec form은 PID 1이 SIGTERM 직수신 — D11·D15 graceful shutdown 전제. 테스트는 PR 게이트 `ci.yml` 담당이고 `@SpringBootTest`는 docker build 컨텍스트(DB·Redis 없음)에서 실행 불가                     | ✅                                                          |
| D32 | worker 이미지 전략 (torch·모델) | ① torch를 PyTorch CPU 인덱스로 설치(`2.5.1+cpu`, requirements.txt 무수정) ② AI 모델(`kor_unsmile`)을 빌드 시 이미지에 베이크(`HF_HOME`)                                                                                  | ① PyPI 기본(CUDA 포함) 설치 ② 기동 시 HF 다운로드 유지 | ① CUDA 휠은 6~7GB → CPU 모델(D3)엔 불필요, CPU 휠로 실측 2.08GB ② 런타임 다운로드면 KEDA 스케일아웃마다 파드별 ~400MB HF 다운로드 → 콜드스타트 증가(D12 역행)·NAT 비용. 베이크 시 노드 캐시로 추가 파드 즉시 기동, `MODEL_VERSION` 추적성 일치          | ✅                                                          |
| D33 | 배포 파이프라인 구조            | app main push → 이미지 3개 빌드(태그=git SHA, matrix 병렬) → OIDC 인증 → ECR push → config `overlays/dev` 이미지 태그 갱신 커밋. config 갱신은 `needs`로 직렬(반쪽 배포 방지), `concurrency`로 연속 머지 대기 직렬화     | Access Key 저장 / 단일 job / 즉시취소                  | "환경=이미지 태그(SHA)"로 불변·추적. OIDC는 장기 자격증명 제거(서명 토큰→STS AssumeRole). 세 이미지 전부 push 후에만 태그 갱신해야 "config는 새 SHA인데 이미지 없음" 방지. config 쓰기는 fine-grained PAT(config 레포 한정)                             | ⏳ (deploy.yml 머지됨, OIDC role 미생성으로 미가동)         |
| D34 | GitOps 변동값·시크릿 처리       | 변동값(DB*URL·REDIS_HOST)·시크릿(DB*\*·JWT_SECRET)을 config(git)에 두지 않고 클러스터에 직접 주입(K8s Secret, `envFrom` 참조). git엔 안정값(포트·큐키·레플리카)만. 최종형은 ESO가 Secrets Manager에서 자동 동기화        | ConfigMap에 주소 기재 / overlay patch로 매일 커밋      | 부트캠프 매일 destroy/apply로 엔드포인트 변동 → git 커밋 시 드리프트·잡음. desired state라도 "어디 적느냐"는 안정성에 따라 다름. Deployment는 Secret 이름만 참조하므로 공급자(수동→ESO) 교체에 무변경. A-5 env 계약 키와 일치(D24)                      | ⏳ (수동 주입 동작 검증, ESO 자동화 후속)                   |
| D35 | EKS 접근 관리                   | `access_config.authentication_mode = API_AND_CONFIG_MAP` 전환 + access entry로 팀원 IAM 유저를 코드에 명시 등록(`AmazonEKSClusterAdminPolicy`). `bootstrap_cluster_creator_admin_permissions = true` 명시(in-place 보장) | aws-auth ConfigMap 수동 편집 유지 / 생성자만 접근      | CONFIG_MAP은 클러스터 생성자만 자동 접근 → 타 팀원 kubectl 인증 거부. access entry로 접근자를 Terraform 코드에 선언(추적·재현, 콘솔 수동편집 회피). bootstrap 속성 생략 시 provider가 클러스터 강제 교체 트리거(provider 이슈 #38967) → 명시로 in-place | ✅                                                          |
| D36 | 컴포넌트·ECR 네이밍             | 역할 기반 이름: `team1-dev-api-server`(Chat Server)·`team1-dev-ai-worker`(Worker)·`team1-dev-frontend`(Frontend). app(deploy.yml)·config(images)·infra(ECR) 3곳 글자 단위 일치                                           | `team1-dev-{chat\|worker\|front}`(초기 본문 예시)      | 본문 C-2 예시(`team1-dev-chat`)와 infra 실제 생성 이름이 갈려 통합 깨짐(D24 인터페이스 계약 위반). 역할 기반이 서술적. infra 실제 이름으로 통일하고 본문 예시도 동기화                                                                                  | ✅                                                          |
| D37 | Frontend 배포 방식              | 클러스터 파드(nginx 정적 서빙)                                                                                                                                                                                           | S3 + CloudFront(정적 호스팅)                           | 빠른 EKS 동작 검증 우선 — 3컴포넌트를 동일 파이프라인으로. S3+CloudFront(CDN·클러스터 부하 분리·비용)가 정적 프론트 실무 표준에 더 가까우나, 전환은 향후 팀 논의(현 결정은 "검증 우선"의 잠정)                                                          | ✅ (잠정)                                                   |
| D38 | 비루트 UID 표현                 | securityContext `runAsNonRoot: true` + `runAsUser/runAsGroup: 999` 명시, Dockerfile `USER`도 숫자(`999`)로 지정                                                                                                          | Dockerfile `USER app`(이름)만                          | `runAsNonRoot` 검증은 숫자 UID를 요구 — 이름(`app`)이면 컨테이너 시작 전 root 여부 확인 불가로 `CreateContainerConfigError`. Dockerfile UID 숫자화가 근본, 매니페스트 명시는 이중 안전                                                                  | ✅                                                          |
| D39 | Frontend↔Backend 연결           | nginx 리버스 프록시 — `/api/`(슬래시 포함)·`/ws`(슬래시 없이, 백엔드 핸들러 경로와 일치)를 Chat Server Service로 프록시. WS는 Upgrade 헤더 + `proxy_read_timeout 3600s`                                                  | 프론트가 백엔드 절대주소 직접 호출(CORS 필요)          | 프론트는 상대경로(`/api`,`/ws`)로 호출 → 같은 출처라 CORS 불필요. nginx `location` 경로가 프론트 호출·백엔드 핸들러와 글자 단위 일치해야 함(`/ws/`≠`/ws`). WS 장시간 유지 위해 기본 60s timeout 연장                                                    | ✅ (D47로 개정 — nginx 리버스 프록시 제거, ALB 직결로 대체) |

### 결정 기록 — 인증 강화 (2026-06-16 추가, D4 번복)

> 멘토링 세션의 보안 피드백을 반영해 D4(닉네임 단순 로그인)를 번복. app PR #42로 반영·머지됨.
> D4와 함께 읽을 것 — D4는 "왜 처음엔 단순하게 갔나", D40은 "왜·어떻게 강화했나".

| ID  | 결정                         | 채택                                                                                                                                                                                                                                                                                                                                                                                                 | 검토한 대안                                                               | 근거                                                                                                                                                                                                                                                                                                                                                                                        | 상태 |
| --- | ---------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---- |
| D40 | 비밀번호 인증 도입 (D4 번복) | username + **password** 로그인, BCrypt 단방향 해시 검증(`passwordEncoder.matches()`). Spring Security는 **BCrypt·암호화 용도로만** 도입하고 인가는 기존 `JwtAuthInterceptor` 유지(`SecurityFilterChain`=`anyRequest().permitAll()`, `/api/login`만 인터셉터 제외). 자동 회원가입 폐지 → 계정은 시드(`user1`~`user6`, BCrypt 해시 `data.sql`). 로그인 경로는 `/api/login` 단일 유지(병렬 경로 미도입) | (D4) 닉네임만으로 자동가입·무검증 / 풀 Spring Security RBAC·리프레시 토큰 | D4는 "인증은 주제가 아니니 단순하게"였으나, 멘토 피드백대로 **로그인된 user가 백엔드 API를 호출 가능**한 구조에서 password 검증은 **비용은 작고 보안 강화는 큼**(외부 노출 시 특히). 인가 로직(JWT 인터셉터·WS 핸드셰이크·프론트 AuthContext)은 건드리지 않아 병렬 작업과 무충돌. 시드 해시는 평문 미커밋(소스에 평문 노출 0). 리프레시·RBAC은 여전히 범위 밖(D4의 "가볍게" 정신 일부 유지) | ✅   |

### 결정 기록 — app 고도화 / Admin & Moderation Control Plane (2026-06-18 추가)

> 어드민 페이지를 중심으로 한 app 추가기능 회의(2026-06-18) 결정. A-1~A-5 본문에 반영됨.
> 🟢 설계 확정 · 구현 예정. 손대는 컴포넌트가 Worker인 건 D43 하나뿐(나머지는 Chat Server + FE) → scale-out 데모와 시점만 분리.
> 통합 락 포인트: JWT 클레임에 `role` 추가(D22 갱신)는 로그인·인터셉터·FE가 모두 닿는 계약 → 가장 먼저 합의·고정.
> 범위 밖(이번 변경): 회원가입(보류·최후순위) · 사용자 방 생성(미구현, 시드 방 유지) · 동영상(FE 독립 재생, 백엔드 계약 무영향).

| ID  | 결정                  | 채택                                                                                                                                            | 검토한 대안                                                    | 근거                                                                                                                                                                                                                                                      | 상태 |
| --- | --------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---- |
| D41 | 어드민 인가           | users `role ENUM('USER','ADMIN')` + JWT role 클레임 + 기존 인터셉터 검증. **단일 플래그**(권한 매트릭스 아님)                                   | `ADMIN_USER_IDS` env 허용목록 / 풀 RBAC·다중 롤                | admin 여부는 *도메인 데이터*라 DB에 유지(config↔app 결합 회피)·확장(enum 값 추가) 용이. 최소 유지로 D40 "RBAC 범위 밖"과 일관. 토큰에 role 박힘 → 박탈은 만료까지 지연(D4·D29 트레이드오프 일관). 보안 노출은 env안과 동일(시드가 git) — role은 비밀 아님 | 🟢   |
| D42 | 금칙어 관리           | DB(진실원) + 파드 인메모리 캐시 + `config:banned-words` pub/sub 무효화. 매칭=정규화 substring contains. `.env` 폐지                             | env/ConfigMap(런타임 편집 불가) / 주기 폴링 / 메시지당 DB 조회 | `.env`가 EKS 미도달(1차 검열 배포 버그) 해소 + 운영자 런타임 CRUD + 메시지당 DB/Redis 회피로 전송 300ms SLO 보호 + fan-out과 같은 "설정 실시간 전파" 패턴 재사용                                                                                          | 🟢   |
| D43 | 2차 검열 정책         | 워커 verdict = **카테고리별 정적 임계값**(코드 상수). 편집 UI·방별 차등 컷                                                                      | 단일 임계값 / 편집 가능 카테고리 슬라이더 / 방별 정책          | unsmile 멀티라벨을 활용해 "독성을 동일 취급 안 함" 입장을 모델·인프라 변경 0으로 표현. 편집 UI·방별은 품 대비 효용 낮고 워커(스케일 주인공)에 표면 추가. "신뢰도"는 측정·한계·개선안 리포트(별도 발표자료)가 짊어짐                                       | 🟢   |
| D44 | 참여자 표시(presence) | Redis Hash `room:{id}:members` + `presence.update`(room 채널). 수 + 목록                                                                        | 수만(SCARD) / DB / TTL·refcount 견고화                         | 핵심 제품(라이브 채팅) 완성도 + "stateless 파드 간 분산 상태" 쇼케이스(Redis·pub/sub 재사용). 멀티탭 refcount·크래시 TTL은 스트레치(데모 staleness 허용)                                                                                                  | 🟢   |
| D45 | 채팅 freeze           | 운영자(ADMIN) 전용. `room:{id}:frozen` 플래그 + 캐시 + `room.freeze`(room 채널). `chat.send` ingestion에서 **거부**(`CHAT_FROZEN`), 버퍼링 없음 | 방장 권한(방 소유권 필요) / 버퍼-후-재생 의미론                | 주제(검열) 직결 운영 도구 + 강한 라이브 시연. 거부 의미론이 단순(버퍼링 불필요)하고 폭주 진정 목적에 부합(쌓았다 재생하면 폭주 재현). 방장 freeze는 방 소유권(미구현) 의존 → v1 어드민 전용                                                               | 🟢   |
| D46 | 어드민 API·로그 뷰    | `/api/admin/*`(role 인터셉터, 미달 403 `FORBIDDEN`). 로그 뷰 content=`COALESCE(moderation_logs.content, messages.content)`                      | 루트 분산 / 키워드 차단 본문 미표시                            | 어드민 표면을 한 prefix로 묶어 인가 일원화. AI 행은 본문 미저장(D25)이라 messages join, 키워드 행은 logs.content → COALESCE로 둘 다 표시                                                                                                                  | 🟢   |

### 결정 기록 — 통신 경로 정정 (2026-06-19 추가, D39 개정)

> scale-out/scale-in 검증을 현 단계 최우선으로 두며, 그 설계(D18 ALB LOR·D15 드레인)가 실제 작동하도록 WS·REST 경로를 ALB 직결로 개정. nginx는 정적 서빙 유지(D37 불변). 상세 개념·근거·다이어그램은 학습 문서 `WS 라우팅·스케일 학습노트`.

| ID  | 결정                                           | 채택                                                                                                                                                                                                                                             | 검토한 대안                                                                     | 근거                                                                                                                                                                                                                                                                                                                              | 상태                   |
| --- | ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------- |
| D47 | WS·REST 경로 ALB 직결 (D39 개정)               | nginx는 정적 서빙만. ALB Ingress가 경로 분기 — `/api`·`/ws`→Chat Server Service, `/`→Frontend Service. nginx 리버스 프록시 제거. 프론트는 상대경로·`wss://${location.host}/ws` 무변경. WS 유지 = ALB `idle_timeout` 연장 + 서버측 ping heartbeat | 현행 nginx 프록시 유지(D39) / WS를 CloudFront 경유(옵션4)                       | 현행 nginx 프록시는 ALB가 chat-server를 직접 못 봐 D18(LOR)·D15(드레인) **미작동**(분산=kube-proxy 랜덤). ALB 직결이라야 scale-out/in 설계가 실제 작동·시연 가능 — 현 단계 최우선의 전제. 같은 출처라 CORS 무발생·프론트 무변경·홉 최단. D37(정적 서빙)은 유지                                                                    | 🟢 설계 확정·구현 예정 |
| D48 | 정적 고도화(S3+CloudFront) 보류 + WS 직결 원칙 | S3+CloudFront는 "정적 서빙" 한정 선택적 폴리시로 보류(핵심 데모 후). 채택 시에도 **WS는 항상 ALB 직결 유지**(CloudFront 경유 금지)                                                                                                               | 지금 S3+CloudFront 전환(WS도 CloudFront 경유=옵션4) / WS 직결+CloudFront(옵션3) | 정적 서빙 위치는 scale·thesis와 무관. CloudFront는 생성·삭제 느려 매일 destroy/apply와 충돌. WS를 CloudFront로 통과시키면 idle 타임아웃(heartbeat 필요)·close-code 전파 불확실성으로 prized 경로(WS scale-in)에 엣지 레이어 추가. https 보안 컨텍스트가 되면 직결 WS는 wss 강제→ALB ACM·도메인·TLS 필요(평문 직결 옵션2는 불필요) | 🟡 보류(향후 팀 논의)  |

---

## 부록: 결정 기록 상세 안내 (참고)

> D5 ~ D10의 배경·선택지·고려점은 이전 버전 부록에 상세히 정리돼 있었으며, 결정 완료에 따라 핵심만 위 "결정 기록" 표로 통합했다. 재논의가 필요하면 표의 "검토한 대안"과 "근거"를 출발점으로 삼는다.
> D11 ~ D18(scale-in)의 상세 개념·다이어그램·트레이드오프는 학습 문서 `scale-in 설계노트`에 별도 정리돼 있다.
> D25 ~ D30은 구현 착수 직전의 계약 점검(2026-06-10)에서 식별되어 2026-06-10 팀 합의로 반영됐다.
> D31 ~ D39는 배포 자동화 세션(2026-06-15)에서 내려졌고, app·config·infra 각 레포 PR(팀 승인)로 반영됐다. 상세 여정·개념은 학습 문서 `배포 자동화 학습노트`에 별도 정리돼 있다.
> D41 ~ D46은 app 고도화 회의(2026-06-18)에서 결정됐고, 본 명세서 A-1~A-5에 반영됐다. 🟢 설계 확정·구현 예정(후속 app PR). 통합 락 포인트 = JWT 클레임 `role` 추가.
> D47 ~ D48은 통신 경로 정정 세션(2026-06-19)에서 결정됐고, A-1·A-5에 반영됐다. D39(nginx 프록시)를 개정·D37(정적 서빙)은 불변. scale-out/in 설계(D18·D15)를 실제 작동시키는 전제. 상세 개념·다이어그램은 학습 문서 `WS 라우팅·스케일 학습노트` 참조.
