# 설계 명세서 (Design Spec) — chatguard (라이브 채팅 + 실시간 AI 검열 플랫폼)

> 이 문서는 팀의 **단일 진실 공급원(SSoT)**. 모든 구현과 각자의 AI 지침은 이 문서를 기준으로 한다.
> v1 동결 후 변경은 팀 합의로만 한다.
> 표기 — 🟢 확정 · 🟡 팀 결정 필요.
> 상태 — **A·B·C·D 섹션 v1 동결(2026-06-04) · E 섹션 동결(2026-06-10).** 모든 결정 확정. (단 D5 SLO 수치·D10 부하 규모는 1차 부하테스트 후 재확정.) · **scale-in 결정 D11~D18 추가(2026-06-10), 구현 Week 4·6/17 무관.** · **인터페이스 계약 보강(2026-06-10): A-3 검열상태 매핑·REST 메시지 스키마, A-5 런타임 계약, 로그인·WS인증·self-echo 규칙 (결정 기록 D19~D24).** · **인터페이스 계약 2차 보강 + 에러 계약(2026-06-10 합의): moderation_logs FK 제거·WS 방 바인딩·캐치업 윈도우·적재/전파 순서·에러 계약·차단 피드백 (결정 기록 D25~D30).** 근거·대안은 하단 "결정 기록" 참조.
> 식별 — 팀 `team1` · 서비스 `chatguard` · VPC CIDR `10.1.0.0/16` (dev `10.1.0.0/17` · prod `10.1.128.0/17`).

---

## A. 컴포넌트 책임 · 데이터 모델 · 메시지 스키마

### A-1. 컴포넌트 책임

| 컴포넌트            | 기술                           | 책임                                                                                                                                                                                                                                                                                                                                                 |
| ------------------- | ------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Frontend            | React                          | 채팅 UI, WebSocket 클라이언트, 메시지 렌더링, `moderation.hide` 수신 시 해당 메시지 블러/삭제, REST로 방·히스토리 조회, `error` 이벤트 처리(차단됨 표시·재시도 안내)와 close code 분기(1001=재연결 / 1008=중단)(D29·D30). **[Week4]** WebSocket 자동 재연결 + jittered backoff, 재접속 시 최근 윈도우 캐치업 병합(검열 상태 `message.status` 단일화) |
| Chat Server         | Spring Boot (WebSocket + REST) | 연결 수락·유지, 인증, 채팅 수신, **1차 키워드 검열(동기)**, message id 발급, Redis 채널 publish/subscribe, **구독 수신 시** 자기 클라이언트에 push(로컬 직접 push 안 함 — A-4 step3), 의심 메시지 큐 적재, MySQL 영속화, 메트릭 노출. **[Week4]** graceful drain(preStop sleep → SIGTERM 1001 close), 새 연결 분산(ALB LOR) + pod당 연결 cap         |
| Moderation Worker   | Python (transformers)          | 검열 큐 소비, AI 모델 판정(in-process), `moderation_logs` 기록, 악성 시 `messages.status` 갱신 + `moderation.hide` publish, 메트릭 노출. **KEDA가 큐 깊이로 스케일하는 대상**. **[Week4]** graceful shutdown(SIGTERM 시 현재 job 마무리 후 종료, 한 번에 1개 pop)                                                                                    |
| Redis (ElastiCache) | —                              | Pub/Sub 채널(전파) + 검열 큐(Redis List)                                                                                                                                                                                                                                                                                                             |
| MySQL (RDS)         | —                              | users · rooms · messages · moderation_logs 영속 저장                                                                                                                                                                                                                                                                                                 |

> **[Week4] 태그** = 설계 확정·구현은 Week 4(중간점검 후). 6/17 의무 아님. 상세는 하단 결정 기록 D11~D18 / 학습 문서 `scale-in 설계노트`.

**설계 결정**

- 🟢 Chat Server는 REST와 WebSocket을 **한 앱**으로. (분리는 스트레치)
- 🟢 **Worker 모델 호스팅 = in-process** (워커 프로세스 내 모델 로드). 작은 CPU 모델이라 단순하고 스케일 단위가 명확. (근거·대안: 결정 기록 D3)
- **Terraform vs GitOps 경계**(가이드라인 17) — Terraform = VPC·EKS·RDS·Redis·IAM·ECR + ArgoCD/KEDA/Prometheus Helm 설치 + Namespace/StorageClass. GitOps(ArgoCD ← config repo) = 앱 Deployment/Service/Ingress, HPA, KEDA ScaledObject, ServiceMonitor, 대시보드.

### A-2. 데이터 모델 (MySQL)

🟢 **message id = 서버 발급 ULID(26자 문자열).** 수신 즉시 발급 → 그 id로 broadcast·큐 적재 → 나중에 `moderation.hide`가 그 id를 타겟. (DB auto-id를 기다리면 전파가 느려지므로 앱에서 발급)

```
users
  id            BIGINT      PK
  username      VARCHAR(50) UNIQUE
  display_name  VARCHAR(50)
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
```

- `messages.status`가 렌더링을 결정: VISIBLE 노출 / BLURRED 블러 / DELETED 숨김.
- `moderation_logs`는 감사 추적이자 대시보드 재료(단계별 차단율 등). 키워드 차단 건은 messages에 저장되지 않으므로 `content`에 원문을 보존한다(D25).
- 🟢 **시간 규약 = 모든 시각 UTC 저장·전송** (DB DATETIME = UTC, API는 ISO-8601 `Z` 표기).
- 🟢 **rooms 생성 경로**: v1은 방 생성 API 없음 — **마이그레이션 시드로 주입**(데모용 고정 방 N개).
- 🟢 **인증 = 닉네임 단순 로그인 + 액세스 전용 JWT** (Spring Security 미사용, 자체 AuthContext — REST는 ThreadLocal 홀더, WS는 핸드셰이크에서 검증 후 세션 바인딩). 비밀번호·리프레시·풀 구현은 미채택. (근거·대안: 결정 기록 D4)
- 🟢 **JWT 클레임 = `{ sub: user_id, display_name }`.** broadcast 시 Chat Server는 검증된 토큰/세션에서 display_name을 읽어 **메시지당 DB 조회를 하지 않는다**(전송 p95 SLO 보호). (결정 기록 D22)
- 🟢 **WS 접속 = 접속 URL 쿼리파라미터 `?token=...&room_id=...`** → 핸드셰이크에서 토큰 검증 + 방 존재 확인 후 **세션을 방 1개에 바인딩**. 방 이동은 재접속으로 처리(`room.join` 이벤트는 스트레치). (노출 회피가 필요하면 `Sec-WebSocket-Protocol` 서브프로토콜로 확장 — 스트레치. 결정 기록 D21·D26)
- 🟢 **로그인 의미론**: 처음 보는 username이면 user 생성(`display_name`=username 기본), 기존이면 조회. 응답 `{ user_id, display_name, token }`.

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
```

- 🟢 **상태 매핑**: `action:"blur" ↔ messages.status=BLURRED`, `action:"delete" ↔ DELETED`. `VISIBLE`은 hide 이벤트 없음. 클라이언트는 `messages.status`를 렌더링 단일 진실원으로 사용(D14). **v1에서 worker는 `blur`만 발행**(A-4 step4) — `delete`/`DELETED`는 스키마상 예약(미사용, 향후/수동 모더레이션용). (결정 기록 D19)

**Redis Pub/Sub**

- 🟢 채널 = `room:{room_id}`. Chat Server들이 `chat.message`·`moderation.hide` 봉투를 publish, 모든 Chat Server가 subscribe하여 해당 방 클라이언트에 forward.

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

| 메서드 | 경로                                            | 용도                                              |
| ------ | ----------------------------------------------- | ------------------------------------------------- |
| POST   | /api/login                                      | 닉네임 로그인 → user_id, display_name, 액세스 JWT |
| GET    | /api/rooms                                      | 방 목록                                           |
| GET    | /api/rooms/{id}/messages?before={ULID}&limit=50 | 히스토리 (DELETED 제외)                           |
| GET    | /actuator/health                                | 헬스체크                                          |
| GET    | /actuator/prometheus                            | 메트릭                                            |

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

**에러 계약 (D29·D30)**

REST — 공통 에러 봉투 (모든 4xx/5xx):

```json
{ "error": { "code": "ROOM_NOT_FOUND", "message": "존재하지 않는 방입니다" } }
```

| HTTP | code              | 발생                                        |
| ---- | ----------------- | ------------------------------------------- |
| 400  | `INVALID_REQUEST` | 필드 누락·형식 오류(login username 공백 등) |
| 401  | `UNAUTHORIZED`    | 토큰 없음·무효·만료                         |
| 404  | `ROOM_NOT_FOUND`  | 존재하지 않는 room_id                       |
| 500  | `INTERNAL`        | 서버 내부 오류                              |

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
2. Chat Server: ULID 발급 → 1차 키워드 검열
   - **차단**: `moderation_logs(KEYWORD, BLOCK, content=원문)` 기록(D25) → **발신자에게 `error(MESSAGE_BLOCKED)` 회신(D30)**, 타 클라이언트 전파 없음 (끝) — messages 미저장
   - **통과**: MySQL 저장(status=VISIBLE) → **`mod:queue`에 job LPUSH → `room:{id}`에 `chat.message` publish** (순서 고정 — D28)
     - LPUSH 실패: publish 하지 않고 `error(INTERNAL)` 응답 → 검열 안 된 노출 차단, 클라 재시도 가능
     - publish 실패: 메시지는 저장·큐에 존재(히스토리에서만 보임) — 드문 케이스로 허용
3. 모든 Chat Server: 구독 채널에서 `chat.message` 수신 → 해당 방 클라이언트에 push (**즉시 노출**). **발신 서버도 로컬에서 직접 push하지 않고 publish만 하며, 자기 클라 포함 모든 push는 이 구독 경로에서 일어난다**(self-receive 포함 → 이중 푸시 방지). (결정 기록 D23)
4. Worker: 큐 소비 → 모델 판정(in-process) → `moderation_logs(AI, ...)` 기록
   - **BLOCK**: `messages.status = BLURRED` → `room:{id}`에 `moderation.hide` publish → 클라이언트 사후 블러 (**v1은 `blur`만** — A-3 매핑)
5. KEDA: `mod:queue` 길이 감시 → 워커 수 조절

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
  MOD_QUEUE_KEY=mod:queue, ROOM_CHANNEL_PREFIX=room:
Moderation Worker
  DB_URL, DB_USER, DB_PASSWORD(secret),
  REDIS_HOST, REDIS_PORT,
  MOD_QUEUE_KEY=mod:queue, MODEL_VERSION
```

- 큐 키(`mod:queue`)·채널 패턴(`room:{id}`)은 A-3 확정값과 동일. 시크릿 원문은 Secrets Manager에만(코드·state 금지, CLAUDE.md §5).

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
  - 이름이 유일해야 하는 것(EKS Cluster, Node Group, ECR, RDS, ElastiCache, S3) → 이름에 `team1-{env}-` 강제. 예: `team1-dev-chat`, `team1-prod-mysql`. (프로젝트 슬러그는 리소스 이름엔 안 들어감)
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

| ID  | 결정                      | 채택                                                                                                              | 검토한 대안                                                             | 근거                                                                                                                                                                          |
| --- | ------------------------- | ----------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| D1  | 검열 큐 구현              | Redis List                                                                                                        | Redis Streams(ack·at-least-once), SQS(완전관리형·DLQ·Pod Identity 활용) | fan-out용 Redis가 이미 필수 → 새 인프라 0. 검열 일감은 1차 통과분이라 드문 유실이 치명적이지 않음. 단순함·빠른 착수 우선. 견고성 필요 시 Streams/SQS로 확장                   |
| D2  | AI 큐 적재 범위           | 1차 통과한 모든 메시지                                                                                            | 샘플링, 위험 기반 선별                                                  | 큐 부하가 충분해 오토스케일 데모에 유리. 미탐(키워드가 놓친 우회형) 커버 최대화. 비용 최적화는 본 프로젝트 목적 아님                                                          |
| D3  | Worker 모델 호스팅        | in-process(워커 내 로드)                                                                                          | 별도 모델 서빙 파드(FastAPI/KServe)                                     | 작은 CPU 모델이라 단순. 스케일 단위가 워커 하나로 명확해 KEDA 데모가 깔끔. 대형/GPU 모델이면 분리가 정석                                                                      |
| D4  | 인증 깊이                 | 닉네임 로그인 + 액세스 전용 JWT (Spring Security 미사용)                                                          | 무인증, id/pw + JWT 풀 Spring Security(access+refresh)                  | 인증은 프로젝트 주제가 아님. 코드가 작아 검토 가능. 신원은 가볍게, 악용 방어는 검열(키워드+AI)이 담당 — 레이트리밋은 v1 범위 밖. WS는 핸드셰이크에서 토큰 검증 후 세션 바인딩 |
| D5  | SLO 목표치                | 가용성 99.9% / 전송 p95 300ms / 검열 반영 p95 5s **(출발값)**                                                     | 더 느슨·엄격한 값                                                       | 데모에서 달성·관찰 가능한 현실적 기준. **1차 부하테스트 후 수치 재확정**                                                                                                      |
| D6  | 알림 채널                 | Slack (dev/prod 채널 분리)                                                                                        | 웹훅·이메일·UI 발화만                                                   | 가이드라인 권장 패턴, 데모 가시성 좋음. Webhook URL은 Secret 관리                                                                                                             |
| D7  | 검열 품질 대시보드 데이터 | Prometheus 카운터                                                                                                 | Grafana→MySQL 직접 조회(스트레치)                                       | 단순·추가 데이터소스 불필요. 집계가 관측성 핵심. 개별 메시지 드릴다운 원하면 MySQL 추가                                                                                       |
| D8  | 네이밍·CIDR               | prefix `team1-{env}-` · 서비스 `chatguard`(리포명) · VPC `10.1.0.0/16` → dev `10.1.0.0/17` · prod `10.1.128.0/17` | (CIDR 운영진 배정 확인 완료)                                            | 공유 계정 충돌 방지(가이드라인 04·06). /17 반분할은 비겹침·정렬 정확, EKS Pod IP 수요(/20+)에도 여유                                                                          |
| D9  | prod 인프라 사양          | HA 구조 + 최소 인스턴스(노드 t3.large×3, db.t4g.small Multi-AZ)                                                   | 가이드라인 예시(m6i.large 등) 그대로                                    | 데모 목적이라 과한 사양 불필요. 평가는 설계(HA·변수화)지 용량이 아님. 작은 인스턴스가 오토스케일 데모에도 유리                                                                |
| D10 | 부하테스트 규모           | 스케일 동작·회복 증명 우선, 수치는 부하테스트로 확정                                                              | 대규모(수만 연결) 고정                                                  | 부트캠프 예산 한계. 올바른 신호 기반 스케일·회복이 포트폴리오 핵심. 부하 생성기는 대상과 분리                                                                                 |

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

---

## 부록: 결정 기록 상세 안내 (참고)

> D5 ~ D10의 배경·선택지·고려점은 이전 버전 부록에 상세히 정리돼 있었으며, 결정 완료에 따라 핵심만 위 "결정 기록" 표로 통합했다. 재논의가 필요하면 표의 "검토한 대안"과 "근거"를 출발점으로 삼는다.
> D11 ~ D18(scale-in)의 상세 개념·다이어그램·트레이드오프는 학습 문서 `scale-in 설계노트`에 별도 정리돼 있다.
> D25 ~ D30은 구현 착수 직전의 계약 점검(2026-06-10)에서 식별되어 2026-06-10 팀 합의로 반영됐다.
