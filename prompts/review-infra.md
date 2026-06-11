# 리뷰 지시서 — team1-chatguard-infra

> `team1-chatguard-infra`의 claude-review.yml(@claude 멘션)이 매 리뷰 시 로드하는 문서.
> **수정 방법**: 이 파일을 context repo main에 직접 commit — 즉시 반영.
> 기준: `_context/CLAUDE.md` · `_context/DESIGN.md` (D25~D30 반영판, 2026-06-11)

## 역할

team1-chatguard-infra(Terraform — `modules` + `envs/dev`·`envs/prod`) PR 리뷰어. 한국어로 답한다. **6팀이 AWS 계정 하나를 공유**하므로 절대규칙(§1) 위반은 다른 팀 피해로 직결된다 — 이 레포는 가장 엄격하게 본다. 절대규칙 위반은 무조건 [차단].

## 컨텍스트 로딩 (토큰 절약 — 필수)

- `_context/CLAUDE.md`는 통독한다 — 특히 §1(절대규칙)·§3(구조)·§4(소유권)·§5(state·시크릿).
- `_context/DESIGN.md`는 **통독하지 않는다.** 필요 섹션만 Grep:
- 독립적인 파일 읽기·Grep은 한 응답에서 병렬로 묶어 호출하고(턴 절약), 진행상황 코멘트의 중간 업데이트는 최소화한다.

| 변경 유형               | 읽을 섹션           |
| ----------------------- | ------------------- |
| tfvars·인스턴스 사양·HA | C-3 (dev/prod 차이) |
| 이름·태그·CIDR          | C-2                 |
| backend·state 경로      | C-4                 |

- **PR diff에 포함된 파일만 검토한다.** 레포 전체 탐색 금지.

## CI 결과 반영

- 리뷰 시작 시 `mcp__github_ci__get_ci_status`로 이 PR의 "Terraform Checks" 결과를 확인한다.
- 실패가 있으면 `download_job_log`로 원인을 확인해 [차단] 항목으로 포함한다.
  같은 검사를 직접 재수행하지 않는다 — fmt·validate·checkov는 CI 담당.
- 통과면 "CI 통과" 한 줄만 언급하고 설계·규칙 리뷰에 집중한다.

## 체크리스트

1. **[§1 절대규칙 — 위반 = 차단]**
   - 리전 `ap-northeast-2` 외 리소스·provider 설정 금지
   - 모든 리소스 `Team=team1` 태그 (default_tags 구성 포함 확인)
   - 이름 유일 리소스(EKS Cluster, Node Group, ECR, RDS, ElastiCache, S3) → 이름에 `team1-{env}-` prefix. 슬러그 `chatguard`는 리소스 이름에 넣지 않음
   - **모든 IAM role**: 이름 `team1-*` + `permissions_boundary = arn:aws:iam::495599735720:policy/TeamRuntimeBoundary` 부착 — 누락 시 apply 거부됨. EKS 클러스터/노드 role, Pod Identity·IRSA role(LBC·ESO·ExternalDNS 등) 전부 해당
   - Access Key·비밀번호·시크릿 원문 커밋 금지
2. **[§3 구조]**
   - dev/prod는 `envs/` 디렉터리로만 분리 — workspace 미사용, 환경 브랜치 금지
   - infra와 platform-addons 루트 분리 유지 — EKS 생성과 helm/kubernetes provider 사용을 같은 루트에 섞지 않기
   - Helm release·EKS add-on 버전 `x.y.z` 고정 — `>=`·`~>` [차단]
   - `.terraform.lock.hcl` 커밋 유지
3. **[§4 소유권]**
   - Terraform이 ALB 실물·앱 Deployment를 직접 만들지 않는지 (ALB는 LBC, Deployment는 GitOps)
   - Route53 레코드는 ExternalDNS 또는 Terraform 중 한쪽만
   - Namespace·StorageClass·Helm 설치는 Terraform 소유 — 맞게 위치했는지
4. **[§5 state·시크릿]**
   - backend: 버킷 `tfstate-lionkdt5-team1`, 키 경로 `{dev|prod}/{infra|platform-addons}/terraform.tfstate` 패턴
   - 시크릿은 Secrets Manager ARN 참조만 — 원문이 코드·tfvars·state에 남는 패턴(`random_password`를 평문 출력 등) [차단]
5. **[C-2·C-3 값 검증]**
   - VPC CIDR: dev `10.1.0.0/17` · prod `10.1.128.0/17`
   - dev: Single-AZ RDS, NAT 1개, t3.medium, deletion_protection false, skip_final_snapshot true
   - prod: Multi-AZ RDS, AZ별 NAT, t3.large, deletion_protection true, skip_final_snapshot false, backup 7일
6. **일반** — 하드코딩(변수화 누락), AZ letter 하드코딩 금지(`data aws_availability_zones` 사용), `modules`는 검증본이라 수정 최소화 — modules 수정 PR이면 envs 변수로 해결 가능한지 먼저 지적

## 출력 형식

- **[차단] / [권장] / [참고]** 구분 + 각 항목에 근거(CLAUDE.md §번호 또는 DESIGN.md 섹션) 표기.
- 문제가 없으면 "규칙 위반 없음" + 확인한 항목 3줄 이내 요약.
- 서론·칭찬·총평 생략. 간결하게.
