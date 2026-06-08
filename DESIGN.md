# 설계 명세서 (Design Spec) — chatguard (라이브 채팅 + 실시간 AI 검열 플랫폼)

> 이 문서는 팀의 **단일 진실 공급원(SSoT)**. 모든 구현과 각자의 AI 지침은 이 문서를 기준으로 한다.
v1 동결 후 변경은 팀 합의로만 한다.
표기 — 🟢 확정 · 🟡 팀 결정 필요.
상태 — **A·B·C·D 섹션 v1 동결 · E 섹션 초안 (2026-06-04).** 모든 결정 확정. (단 D5 SLO 수치·D10 부하 규모는 1차 부하테스트 후 재확정.) 근거·대안은 하단 "결정 기록" 참조.
식별 — 팀 `team1` · 서비스 `chatguard` · VPC CIDR `10.1.0.0/16` (dev `10.1.0.0/17` · prod `10.1.128.0/17`).
> 

---

## A. 컴포넌트 책임 · 데이터 모델 · 메시지 스키마

### A-1. 컴포넌트 책임

| 컴포넌트 | 기술 | 책임 |
| --- | --- | --- |
| Frontend | React | 채팅 UI, WebSocket 클라이언트, 메시지 렌더링, `moderation.hide` 수신 시 해당 메시지 블러/삭제, REST로 방·히스토리 조회 |
| Chat Server | Spring Boot (WebSocket + REST) | 연결 수락·유지, 인증, 채팅 수신, **1차 키워드 검열(동기)**, message id 발급, Redis 채널 publish/subscribe, 자기 클라이언트에 push, 의심 메시지 큐 적재, MySQL 영속화, 메트릭 노출 |
| Moderation Worker | Python (transformers) | 검열 큐 소비, AI 모델 판정(in-process), `moderation_logs` 기록, 악성 시 `messages.status` 갱신 + `moderation.hide` publish, 메트릭 노출. **KEDA가 큐 깊이로 스케일하는 대상** |
| Redis (ElastiCache) | — | Pub/Sub 채널(전파) + 검열 큐(Redis List) |
| MySQL (RDS) | — | users · rooms · messages · moderation_logs 영속 저장 |

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
  message_id    CHAR(26) FK -> messages.id
  stage         ENUM('KEYWORD','AI')
  verdict       ENUM('PASS','BLOCK')
  score         FLOAT       NULL       -- AI 판정 점수
  model_version VARCHAR(50) NULL
  reason        VARCHAR(200) NULL
  checked_at    DATETIME
  INDEX (message_id)
```

- `messages.status`가 렌더링을 결정: VISIBLE 노출 / BLURRED 블러 / DELETED 숨김.
- `moderation_logs`는 감사 추적이자 대시보드 재료(단계별 차단율 등).
- 🟢 **인증 = 닉네임 단순 로그인 + 액세스 전용 JWT** (Spring Security 미사용, 자체 AuthContext — REST는 ThreadLocal 홀더, WS는 핸드셰이크에서 검증 후 세션 바인딩). 비밀번호·리프레시·풀 구현은 미채택. (근거·대안: 결정 기록 D4)

### A-3. 메시지 · 이벤트 스키마

공통 봉투: `{ "type": "...", "payload": { ... } }`

**WebSocket — Client → Server**

```json
{ "type": "chat.send", "payload": { "room_id": 1, "content": "안녕하세요" } }
```

(user_id는 토큰에서 검증된 AuthContext에서 추출 — body로 받지 않음)

**WebSocket — Server → Client**

```json
{ "type": "chat.message",
  "payload": { "id": "01J9...", "room_id": 1, "user_id": 7,
               "display_name": "viewer7", "content": "안녕하세요",
               "created_at": "2026-06-04T12:00:00Z" } }

{ "type": "moderation.hide",
  "payload": { "id": "01J9...", "action": "blur" } }   // action: "blur" | "delete"
```

**Redis Pub/Sub**

- 🟢 채널 = `room:{room_id}`. Chat Server들이 `chat.message`·`moderation.hide` 봉투를 publish, 모든 Chat Server가 subscribe하여 해당 방 클라이언트에 forward.

**검열 큐**

- 🟢 **큐 구현 = Redis List** (`mod:queue`, LPUSH/BRPOP) + KEDA Redis Lists 스케일러. 견고성 필요 시 Streams/SQS로 확장(스트레치). (근거·대안: 결정 기록 D1)
- job 포맷:

```json
{ "message_id": "01J9...", "room_id": 1, "content": "안녕하세요",
  "enqueued_at": "2026-06-04T12:00:00Z" }
```

- 🟢 **AI 큐 적재 = 1차 통과한 모든 메시지.** (명백한 욕설은 1차에서 즉시 차단, 나머지는 AI 비동기 확인.) 큐 부하가 충분해 오토스케일 데모에 유리. (근거·대안: 결정 기록 D2)

**REST API (최소)**

| 메서드 | 경로 | 용도 |
| --- | --- | --- |
| POST | /api/login | 닉네임 로그인 → user_id, 액세스 JWT |
| GET | /api/rooms | 방 목록 |
| GET | /api/rooms/{id}/messages?before={ULID}&limit=50 | 히스토리 (DELETED 제외) |
| GET | /actuator/health | 헬스체크 |
| GET | /actuator/prometheus | 메트릭 |

### A-4. 처리 흐름 (시퀀스 요약)

1. Client `chat.send` → Chat Server
2. Chat Server: ULID 발급 → 1차 키워드 검열
    - **차단**: `moderation_logs(KEYWORD, BLOCK)` 기록, 전파 안 함 (끝)
    - **통과**: MySQL 저장(status=VISIBLE) → `room:{id}`에 `chat.message` publish → `mod:queue`에 job 적재
3. 모든 Chat Server: 구독 채널에서 `chat.message` 수신 → 해당 방 클라이언트에 push (**즉시 노출**)
4. Worker: 큐 소비 → 모델 판정(in-process) → `moderation_logs(AI, ...)` 기록
    - **BLOCK**: `messages.status = BLURRED` → `room:{id}`에 `moderation.hide` publish → 클라이언트 사후 블러
5. KEDA: `mod:queue` 길이 감시 → 워커 수 조절

---

## B. 관측성 · SLO · 메트릭 계획

### B-1. SLI / SLO

> 골든 시그널(지연·트래픽·오류·포화)을 재료로, 사용자 체감 지표 3개를 SLI로 정한다.
> 

| SLI | 정의 | SLO (출발값) | 측정 |
| --- | --- | --- | --- |
| 채팅 전송 성공률 | 성공 broadcast / 전체 chat.send | ≥ 99.9% (월 에러버짓 ≈ 43분) | Chat Server |
| 채팅 전송 지연 | chat.send 수신 → broadcast 푸시 | p95 < 300ms | Chat Server |
| 검열 반영 지연 | 큐 적재 → moderation.hide 발행(차단 건) | p95 < 5s | Worker |
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
- `moderation_e2e_seconds` (histogram) — 적재→hide 발행, **검열 반영 지연 SLI**

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

| 항목 | dev | prod |
| --- | --- | --- |
| name_prefix | team1-dev | team1-prod |
| VPC CIDR | 10.1.0.0/17 | 10.1.128.0/17 |
| AZ 수 | 2 | 3 |
| NAT Gateway | 1 (비용) | AZ별 다중(HA) |
| EKS 노드 | 2 × t3.medium | 3 × t3.large |
| EKS endpoint | public + private | private 또는 IP 제한 |
| RDS | db.t4g.micro, Single-AZ | db.t4g.small, Multi-AZ |
| RDS deletion_protection | false | true |
| RDS skip_final_snapshot | true | false |
| Backup retention | 1일 | 7일 |
| Redis | cache.t4g.micro | cache.t4g.small |
| 관측성 retention | 7일 | 30일+ |
| 알람 채널 | dev | prod |
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

| 실험 | 자극 | 관찰(스케일) | 핵심 메트릭 |
| --- | --- | --- | --- |
| A. 접속 계층 | WebSocket 연결 램프업 | Chat Server 레플리카(①) | 접속 수, 전송 지연 |
| B. 검열 워커 (헤드라인) | 메시지 처리량 폭주 | 워커 레플리카(②, KEDA) | 큐 깊이, 검열 반영 지연 |
| (서사) 경기일 통합 | 접속 + 채팅 동시 폭주 | 둘 다 | 발표 데모용. 본 측정은 A·B로 |
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

---

## E. 중간점검(6/17) Done 정의

> 6/17 중간점검에서 "무엇이 동작함을 보여줄지"의 합의. 이 목표를 기준으로 역산해 일한다. 가이드라인 21(Week 2 = CI/CD + 중간점검 통과)에 맞춤. 🟡 팀 확인 후 동결.
> 

### E-1. 목표 한 줄

dev 환경에서 **"채팅이 흐르고, 파이프라인이 돌고, 위험한 가정이 깨졌음"**을 end-to-end로 증명. (완성도가 아니라 토대의 동작 증명이 목표 — 풀 데모·관측성·prod는 3~4주차.)

### E-2. Done 체크리스트

**반드시 (토대):**

1. dev 인프라 `terraform apply` 성공 + `kubectl`로 EKS 접속 확인 (위험 제거 1)
2. 채팅 워킹 스켈레톤 — React 입력 → Chat Server(WebSocket) → Redis Pub/Sub → 다른 클라이언트 표시. **멀티 파드 fan-out 동작**(위험 제거 2). 1차 키워드 검열 즉시 차단 동작.
3. CI/CD 한 바퀴 — app main merge → GitHub Actions(OIDC) → ECR push → config repo 이미지 태그 갱신 → ArgoCD가 dev에 배포 (가이드라인 16)
4. 관측성 최소 — Prometheus/Grafana 기동, 최소 한 서비스 스크레이프 + 기본 패널(접속 수 또는 큐 깊이)

**동작 증명 (스파이크 수준, 폭은 작아도 — 위험 제거):**
5. AI 검열 워커가 큐 job을 꺼내 판정 → `moderation.hide` 전파 → 사후 블러 1회 (모델 CPU 서빙 스파이크 통과 — 위험 제거 3)
6. KEDA가 큐 깊이로 워커 1→N 1회 확장 확인 (오토스케일 스파이크 통과 — 위험 제거 4)

### E-3. 시연 시나리오 (그날)

1. `kubectl get nodes/pods`로 dev 클러스터 접속 보여주기
2. 두 브라우저로 실시간 채팅 + 금칙어 즉시 차단 1회
3. 악성 채팅 1개 → 잠시 후 사후 블러 (검열 흐름)
4. 코드 한 줄 변경 push → 파이프라인 → ArgoCD dev 반영 (라이브 또는 녹화)
5. Grafana 기본 대시보드 한 화면
6. k6로 큐 부하 → 워커 1→N 늘어남 1회 (스파이크)

### E-4. 6/17엔 기대하지 않음 (3~4주차로)

prod 환경, 풀 부하·오토스케일 데모 완성, SLO/에러버짓 대시보드, OpenTelemetry, 우회형 탐지 개선, hardening 다수, 사후 블러 UX 폴리시.

### E-5. 역산 일정 (대략)

- ~Week1 끝(≈6/12): dev 인프라 apply + 접속, 채팅 스켈레톤 동작
- Week2(6/15~16): CI/CD 연결, ArgoCD dev 배포, 관측성 기동, 검열·KEDA 스파이크 통과
- 6/16: 리허설·시연 점검
- 6/17: 중간점검

---

## 결정 기록 (Decision Log)

> ADR(아키텍처 결정 기록) 경량판. 
각 결정의 채택안·검토한 대안·근거를 남긴다. (A·B·C·D: 2026-06-04 동결)
> 

| ID | 결정 | 채택 | 검토한 대안 | 근거 |
| --- | --- | --- | --- | --- |
| D1 | 검열 큐 구현 | Redis List | Redis Streams(ack·at-least-once), SQS(완전관리형·DLQ·Pod Identity 활용) | fan-out용 Redis가 이미 필수 → 새 인프라 0. 검열 일감은 1차 통과분이라 드문 유실이 치명적이지 않음. 단순함·빠른 착수 우선. 견고성 필요 시 Streams/SQS로 확장 |
| D2 | AI 큐 적재 범위 | 1차 통과한 모든 메시지 | 샘플링, 위험 기반 선별 | 큐 부하가 충분해 오토스케일 데모에 유리. 미탐(키워드가 놓친 우회형) 커버 최대화. 비용 최적화는 본 프로젝트 목적 아님 |
| D3 | Worker 모델 호스팅 | in-process(워커 내 로드) | 별도 모델 서빙 파드(FastAPI/KServe) | 작은 CPU 모델이라 단순. 스케일 단위가 워커 하나로 명확해 KEDA 데모가 깔끔. 대형/GPU 모델이면 분리가 정석 |
| D4 | 인증 깊이 | 닉네임 로그인 + 액세스 전용 JWT (Spring Security 미사용) | 무인증, id/pw + JWT 풀 Spring Security(access+refresh) | 인증은 프로젝트 주제가 아님. 코드가 작아 검토 가능. 신원은 가볍게, 악용 방어는 검열+레이트리밋으로. WS는 핸드셰이크에서 토큰 검증 후 세션 바인딩 |
| D5 | SLO 목표치 | 가용성 99.9% / 전송 p95 300ms / 검열 반영 p95 5s **(출발값)** | 더 느슨·엄격한 값 | 데모에서 달성·관찰 가능한 현실적 기준. **1차 부하테스트 후 수치 재확정** |
| D6 | 알림 채널 | Slack (dev/prod 채널 분리) | 웹훅·이메일·UI 발화만 | 가이드라인 권장 패턴, 데모 가시성 좋음. Webhook URL은 Secret 관리 |
| D7 | 검열 품질 대시보드 데이터 | Prometheus 카운터 | Grafana→MySQL 직접 조회(스트레치) | 단순·추가 데이터소스 불필요. 집계가 관측성 핵심. 개별 메시지 드릴다운 원하면 MySQL 추가 |
| D8 | 네이밍·CIDR | prefix `team1-{env}-` · 서비스 `chatguard`(리포명) · VPC `10.1.0.0/16` → dev `10.1.0.0/17` · prod `10.1.128.0/17` | (CIDR 운영진 배정 확인 완료) | 공유 계정 충돌 방지(가이드라인 04·06). /17 반분할은 비겹침·정렬 정확, EKS Pod IP 수요(/20+)에도 여유 |
| D9 | prod 인프라 사양 | HA 구조 + 최소 인스턴스(노드 t3.large×3, db.t4g.small Multi-AZ) | 가이드라인 예시(m6i.large 등) 그대로 | 데모 목적이라 과한 사양 불필요. 평가는 설계(HA·변수화)지 용량이 아님. 작은 인스턴스가 오토스케일 데모에도 유리 |
| D10 | 부하테스트 규모 | 스케일 동작·회복 증명 우선, 수치는 부하테스트로 확정 | 대규모(수만 연결) 고정 | 부트캠프 예산 한계. 올바른 신호 기반 스케일·회복이 포트폴리오 핵심. 부하 생성기는 대상과 분리 |

---

## 부록: 결정 기록 상세 안내 (참고)

> D5~D10의 배경·선택지·고려점은 이전 버전 부록에 상세히 정리돼 있었으며, 결정 완료에 따라 핵심만 위 "결정 기록" 표로 통합했다. 재논의가 필요하면 표의 "검토한 대안"과 "근거"를 출발점으로 삼는다.
>