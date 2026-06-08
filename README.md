# team1-chatguard-context

team1 / **chatguard** 프로젝트의 **본부(허브) 레포**입니다.
배포되는 코드가 아니라, 사람과 AI(Claude)가 작업의 기준으로 삼는 **규칙·설계·작업지시·도구**를 모아둡니다. "어떻게 일할지"의 단일 진실 공급원.

## 무엇이 들어있나

| 항목        | 설명                                                                                        |
| ----------- | ------------------------------------------------------------------------------------------- |
| `CLAUDE.md` | 팀 기본 규칙. 작업 전 항상 먼저 읽는다. Claude AI 프로젝트 지침에도 같은 내용 반영.         |
| `DESIGN.md` | 설계 명세서(SSoT). 컴포넌트·데이터 모델·메시지 스키마·SLO·부하테스트·결정 기록.             |
| `tasks/`    | 작업 지시서. Claude AI로 만들어 올리면 Claude Code가 읽고 실행. (`TEMPLATE.md` 복사해 사용) |
| `skills/`   | Claude Code 공용 스킬. 쓰려면 `~/.claude/skills/`에 설치(아래 셋업 참고).                   |

## 레포 4개 구성

- **team1-chatguard-context** — (이 레포) 규칙·설계·작업지시·스킬
- **team1-chatguard-app** — 앱 소스(React / Spring Boot / Python worker), Dockerfile, CI
- **team1-chatguard-config** — k8s 매니페스트(overlays), ArgoCD — 배포의 SSoT
- **team1-chatguard-infra** — Terraform(modules + envs/dev·prod)

## 로컬 셋업 (CLI = Claude Code 사용자)

```bash
# 1) 4개 레포를 한 부모 폴더 아래 둔다
~/chatguard/
├── team1-chatguard-context
├── team1-chatguard-app
├── team1-chatguard-config
└── team1-chatguard-infra

# 2) 작업할 레포에서 Claude Code 실행 + 본부 문서 함께 보기
cd ~/chatguard/team1-chatguard-infra
claude
/add-dir ../team1-chatguard-context
```

## 작업(task) 흐름

1. **Claude AI(웹)** 로 작업 지시서 초안 작성 (프로젝트에 DESIGN·CLAUDE 로드됨).
2. `tasks/NNNN-제목.md` 로 저장 → **이 레포 main에 commit/push** (혼자 쓰는 레포라 PR 불필요).
3. 작업 레포에서 **Claude Code**: `git pull`(본부 최신화) → "`tasks/NNNN` 읽고 작업해줘" → 구현.
4. 작업 레포에서 **PR 생성**(팀원 리뷰) → 머지.
5. 해당 task 상태를 `done`으로 변경(또는 `tasks/archive/`로 이동).

> 작업 *기록*은 여기 쌓지 않는다 — git 커밋·PR이 곧 기록. `tasks/`엔 "할 일 지시서"만 둔다.

## MCP (CLI 사용자 로컬 설정 메모)

- **infra 레포:** `terraform`(HashiCorp), `aws-docs`
- **app 레포:** `context7`(라이브러리 문서), `k6`(부하테스트 스크립트)
- **나중에:** Grafana / Kubernetes / CloudWatch (해당 시스템이 뜬 뒤)
