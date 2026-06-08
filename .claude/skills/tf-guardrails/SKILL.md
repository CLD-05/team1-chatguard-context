---
name: tf-guardrails
description: Terraform 코드(infra 레포)를 작성·수정하거나 plan/apply 하기 전에, 팀 정책으로 막히는 항목을 자체 점검할 때 사용. .tf 파일 작성·tfvars 수정·apply 직전에 트리거.
---

# Terraform 가드레일 (apply 거부 방지)

infra 레포에서 코드/플랜 전에 아래를 확인한다. 하나라도 어기면 **생성이 계정 정책에 의해 거부**되어 apply가 멈춘다.

## 거부 유발 항목 (필수)

- 모든 리소스에 `Team=team1` 태그 (default_tags 동작 확인). 없거나 다른 팀 값이면 EC2/RDS/EKS 생성 거부.
- IAM role 이름 `team1-*` + 권한경계 부착:
  `iam_role_permissions_boundary = "arn:aws:iam::495599735720:policy/TeamRuntimeBoundary"`
  (EKS 클러스터·노드, LBC·ESO·ExternalDNS 등 Pod Identity/IRSA role 전부 해당). 없으면 role 생성 거부.
- 리전 `ap-northeast-2` 고정.

## 깨지기 쉬운 항목

- backend: 버킷 `tfstate-lionkdt5-team1`, key는 레이어별(`dev/infra/...` 등), `use_lockfile = true`. 부트스트랩 불필요.
- infra와 platform-addons를 **같은 apply에 섞지 않음**.
- ALB를 직접 만들지 않음(LBC가 생성) / Route53은 한쪽만 / Helm·add-on 버전 고정(`x.y.z`).
- 시크릿 원문이 state·코드에 없음 (Secrets Manager + ARN 참조).

## 작업 방식

- 최신 프로바이더/모듈 버전·문법은 `terraform` MCP로 확인한다.
- plan 결과를 사람에게 보여주고, **apply·destroy·prod 변경은 사람 승인 후** 진행.
