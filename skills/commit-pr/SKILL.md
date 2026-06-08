---
name: commit-pr
description: 커밋 메시지나 Pull Request 설명을 작성할 때 사용. 변경을 커밋하거나 PR로 정리하는 순간 트리거.
---

# 커밋 · PR 규약

## 커밋 메시지

- 형식: `<type>(<scope>): <요약>`
  - type: `feat` | `fix` | `refactor` | `chore` | `docs` | `test` | `ci`
  - scope 예: `infra`, `app-chat`, `app-worker`, `app-web`, `config`, `monitoring`
- 요약은 한 줄 명령형. 본문엔 "왜"를 적는다.
- 예: `feat(infra): dev VPC + EKS 모듈 호출 추가`

## PR 설명 (템플릿)

- **무엇을** 바꿨나 (1~3줄)
- **왜** (배경 / 관련 task: `tasks/NNNN`)
- **확인 방법** (terraform plan 결과 / 스크린샷 / 테스트)
- **체크리스트**: 규칙 위반 없음(리전·prefix·태그·권한경계·소유권 경계), 시크릿 없음, 대상 레포 한 곳

## 원칙

- 쓰기는 한 번에 **한 레포·한 PR**.
- **환경=브랜치 금지** (브랜치는 feature→main, 환경은 디렉터리/오버레이/이미지 태그).
- **prod 관련 변경**은 승인 게이트 대상임을 PR에 명시.
