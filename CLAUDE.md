# CLAUDE.md — team1 / chatguard 기본 규칙

> 이 파일은 사람과 AI(Claude)가 작업 전에 **항상 먼저 읽는 규칙 문서**입니다.
> 규칙은 모든 대화·모든 레포에 항상 적용됩니다. 바꿀 때는 임의로 고치지 말고 **PR 리뷰를 거쳐** 이 파일을 수정합니다.
> **상세 설계(컴포넌트·데이터 모델·메시지 스키마·SLO·부하테스트)는 `DESIGN.md`가 단일 진실 공급원(SSoT)입니다.** 이 파일과 충돌하면 DESIGN.md를 우선합니다.

---

## 0. 프로젝트 한눈에

- **팀:** team1 · **서비스:** chatguard (라이브 채팅 + 실시간 AI 검열 플랫폼)
- **기간:** 2026.06.02 ~ 07.08 · 중간점검 06.17 · 발표 07.09 13:00
- **AWS 접속 가능: 6/8(월)부터, 매일 09:00 ~ 18:00** — 이 시간에만 클라우드 리소스 작업이 된다.
- **리전:** 서울(ap-northeast-2) 단독
- **핵심 스택:** EKS 1.35, Terraform 1.14.x, GitHub Actions(OIDC), ECR, ArgoCD, KEDA, Prometheus/Grafana
- **VPC CIDR:** 10.1.0.0/16 (dev 10.1.0.0/17 · prod 10.1.128.0/17)
- **AWS 계정:** Account `495599735720` · 리전 `ap-northeast-2` · CLI 프로필 `final` (`export AWS_PROFILE=final`) · IAM 유저 `team1-cjc`
- **평가 기준:** 기능이 동작하느냐보다 **"설계 의사결정 + 실무 적합성"** → 모든 선택에는 이유를 남긴다 (DESIGN.md 하단 "결정 기록").

**컴포넌트 요약** (상세·책임은 DESIGN.md A-1):

| 컴포넌트          | 기술        | 한 줄 책임                                                            |
| ----------------- | ----------- | --------------------------------------------------------------------- |
| Frontend          | React       | 채팅 UI, WebSocket, `moderation.hide` 수신 시 블러/삭제               |
| Chat Server       | Spring Boot | WebSocket+REST, 1차 키워드 검열, ULID 발급, Redis pub/sub, MySQL 저장 |
| Moderation Worker | Python      | 검열 큐 소비, AI 판정(in-process), **KEDA 스케일 대상**               |
| Redis             | ElastiCache | pub/sub 전파 + 검열 큐(List)                                          |
| MySQL             | RDS         | users · rooms · messages · moderation_logs                            |

---

## 1. 절대 규칙 (어기면 다른 팀 피해 / 작업 거부)

하나의 AWS 계정을 6팀이 공유한다. 예외 없이 지킨다.

1. **서울 리전(ap-northeast-2)에서만** 리소스를 만든다.
2. 모든 리소스에 팀 식별 → 태그 `Team=team1`, 이름 prefix `team1-`.
3. **다른 팀 리소스는 조회·변경·삭제 금지.**
4. **콘솔(웹 화면)에서 손으로 만들지 않는다.** 모든 리소스는 Terraform 코드로만.
5. **비밀번호·Access Key를 코드에 커밋하지 않는다.** (`.gitignore` + Secret 참조)
6. **본인 IAM 유저로만** 작업하고, 자격증명·pem 키를 **공유하지 않는다.**
7. **학생이 만드는 모든 IAM role**은 이름이 `team1-*`이고, **권한 경계 `TeamRuntimeBoundary`를 반드시 부착**해야 한다. 안 붙이면 role 생성이 **거부**됨 → apply 실패.
   - `iam_role_permissions_boundary = "arn:aws:iam::495599735720:policy/TeamRuntimeBoundary"`
   - EKS 클러스터/노드 role, Pod Identity·IRSA role(LBC·ESO·ExternalDNS 등)이 전부 해당. 검증 모듈에 이 변수가 연결됐는지 확인하고 tfvars에 ARN을 넣는다.
8. `Team` 태그(대문자)가 없거나 다른 팀 값이면 EC2/RDS/EKS 생성이 **거부**된다(정책 강제).

---

## 2. 이름·태그 규칙

- 태그 케이스 **PascalCase**. 표준 세트 `Team=team1 / Environment / Project=chatguard / Owner` → `default_tags`로 자동 부착.
- **리소스 이름 prefix = `team1-{env}-`** (예: `team1-dev-api-server`, `team1-prod-mysql`).
  - ⚠️ 서비스 슬러그 `chatguard`는 **리포 이름에만** 쓴다. 리소스 이름·prefix에는 넣지 않는다.
- **컴포넌트 이름은 역할 기반으로 확정 (D36)** — Chat Server `team1-{env}-api-server` · Worker `team1-{env}-ai-worker` · Frontend `team1-{env}-frontend`.
  - app `deploy.yml` · config `images` · infra ECR **3곳이 글자 단위로 일치**해야 함(불일치 시 통합 깨짐).
- **이름이 유일해야 하는 것**(EKS Cluster, Node Group, ECR, RDS, ElastiCache, S3) → 이름에 `team1-{env}-` 강제.
- **ID로 식별되는 것**(VPC, Subnet, NAT, IGW, SG, Bastion) → `Name` 태그 + `Team=team1` 태그.
- team 값 하나가 모든 걸 파생: VPC `10.1.0.0/16`, prefix `team1-`, 태그 `Team=team1`, state 버킷 `tfstate-lionkdt5-team1`

---

## 3. 구조 원칙 (이 프로젝트의 핵심)

- **환경은 브랜치가 아니다.** dev/prod를 git 브랜치로 나누지 않는다.
  - 브랜치 = 코드 성숙도(feature → main). 환경 = 디렉터리 / 오버레이 / 이미지 태그.
  - app: `main` + `feature`, 환경은 이미지 태그(git SHA)로 구분
  - config: `main` 단일, `overlays/dev`·`overlays/prod`
  - infra: `main` 단일, `envs/dev`·`envs/prod`
- dev/prod는 **디렉터리로 분리**(`envs/dev`, `envs/prod`). Terraform workspace 미사용.
- **infra와 platform-addons를 분리**한다.
  - 이유: EKS 생성과 그 위 Helm 설치를 한 apply에 섞으면 첫 apply가 실패한다(클러스터가 없는데 provider가 접속 시도).
  - 순서: ① infra apply(VPC/EKS/RDS/IAM) → ② platform-addons apply(ArgoCD/ESO/Prometheus 등)
- `modules` = 환경 무관 부품(검증본, 수정 최소화), `envs/*` = 실제 실행 위치.
- `.terraform.lock.hcl` 커밋(재현성). Helm·add-on 버전 **고정**(`x.y.z`, `>=` 금지).
- 같은 모듈에 **변수로** dev/prod 차이를 만든다. prod = HA 구조 유지 + 인스턴스는 최소 사양. (상세 tfvars 차이: DESIGN.md C-3)

---

## 4. 누가 무엇을 소유하는가 — Terraform vs GitOps (chatguard)

한 리소스는 **한 도구만** 관리한다. 둘이 같이 건드리면 충돌한다.

- **Terraform이 소유:** VPC, EKS, RDS(MySQL), ElastiCache(Redis), IAM, KMS, ECR, S3(state), EKS Add-on, Helm 설치(ArgoCD / KEDA / kube-prometheus-stack / ESO / LBC / redis_exporter), Namespace, StorageClass
- **GitOps(ArgoCD ← config repo)가 소유:** 앱 Deployment/Service/Ingress, HPA, **KEDA ScaledObject**, ServiceMonitor/PodMonitor, PrometheusRule, Grafana 대시보드, ExternalSecret
- **누구도 직접 안 만듦:** ALB 실물(= Load Balancer Controller가 자동 생성), 민감값 원문(= Secrets Manager에만)
- Route53 레코드는 **ExternalDNS 또는 Terraform 중 한쪽만**

**자주 나는 사고(피할 것):**

1. Terraform이 ALB를 직접 만들고 Ingress도 작성 → ALB 중복 생성
2. Terraform Route53 + ExternalDNS 동시 → 서로 덮어씀
3. Terraform Deployment + ArgoCD Application 동시 → 무한 충돌
   > destroy(삭제) 전에는 반드시 `kubectl delete ingress`를 먼저 한다.

---

## 5. 상태(state)·시크릿

- **state 버킷은 부트캠프가 미리 제공**: `tfstate-lionkdt5-team1` (ap-northeast-2). → 별도 부트스트랩 불필요. `backend.tf`만 연결 후 `terraform init`.
- 백엔드: **S3 + 네이티브 락**(use_lockfile, DynamoDB 미사용). Terraform 1.10+ 필요.
- 버킷은 팀 공통이고 **환경 × 레이어마다 key만 다르게** 분리:
  - `dev/infra/terraform.tfstate`
  - `dev/platform-addons/terraform.tfstate`
  - `prod/infra/terraform.tfstate` · `prod/platform-addons/terraform.tfstate`
- backend 블록은 변수를 못 쓴다 → 루트마다 static `backend.tf`(또는 `-backend-config`). 예시는 온보딩 안내 참조.
- **dev/infra apply는 팀에서 한 명이 운전**(state 공유) — 나머지는 PR로 기여. 동시 apply는 네이티브 락이 막지만, 운전자는 사람이 정한다.
- **시크릿 원문은 state·코드에 절대 남기지 않는다.** Secrets Manager에 두고 Terraform은 ARN만 참조. 런타임 주입은 ESO가 K8s Secret으로 동기화.

---

## 6. 레포 4개와 역할

- `team1-chatguard-context` : **본부** — `DESIGN.md`(설계 SSoT), `CLAUDE.md`(이 파일), 공용 `skills/`, `README.md`
- `team1-chatguard-app` : 앱 소스(React, Spring Boot, Python worker), Dockerfile, CI (배포 이미지 태그 = git SHA)
- `team1-chatguard-config` : k8s 매니페스트(`overlays/dev`·`overlays/prod`), ArgoCD Application — **배포의 단일 진실 공급원(SSoT)**
- `team1-chatguard-infra` : Terraform(`modules` + `envs/dev`·`envs/prod`)

---

## 7. CI/CD·인증 규칙 (요약)

- AWS 인증은 **OIDC** 사용 (GitHub Secret에 Access Key 금지). Trust Policy의 sub로 repo/branch 제한.
- **prod apply·prod 배포는 GitHub Environment 승인 게이트**를 통과해야 한다.
- app: PR = build+test+lint(배포 X) / main merge = Docker build(태그=SHA) → ECR push → config 이미지 태그 갱신
- infra: PR = `terraform plan`(코멘트) / dev = 자동 apply / prod = 승인 후 apply (OIDC role을 dev·prod로 분리)
- config: dev = 자동 동기화 / prod = 이미지 태그 변경 PR + 승인으로 승격

---

## 8. AI(Claude) 작업 규칙

- 위 1~7번 규칙을 **항상** 지킨다. 규칙과 부딪치는 요청이 오면 **멈추고 먼저 확인**한다.
- 되돌릴 수 없는 작업(destroy·삭제·prod 변경) 전에는 **반드시 사람에게 확인**받는다.
- **쓰기는 한 번에 한 레포·한 PR**에만 한다. 여러 레포를 _읽어서_ 참고하는 것은 괜찮다.
- 상세 설계가 필요하면 임의로 정하지 말고 **`DESIGN.md`를 따른다.** 거기에 없으면 팀에 묻고, 정해지면 DESIGN.md에 추가한다.
- 작업을 끝내면 무엇을·왜 했는지 **PR 설명에** 남긴다. (별도 로그 파일은 만들지 않는다 — git이 곧 기록이다.)
- **PR 리뷰를 할 때는 이 문서를 체크리스트로 사용한다** (리전 위반 / `team1-{env}-` prefix 누락 / 소유권 경계 침범 / 환경을 브랜치로 분리 / 시크릿 커밋 등).
