<!--
파일명 규칙: tasks/NNNN-짧은-제목.md   (예: tasks/0001-dev-infra-init.md)
- NNNN: 4자리 순번(생성 순서). 정렬·참조가 쉬움.
- 이 템플릿을 복사해서 새 작업마다 하나씩 만든다.
- 작성: Claude AI(웹)로 초안 → 이 레포 main에 commit/push → Claude Code가 읽고 실행.
-->

# [작업 제목]

- **상태:** todo <!-- todo | doing | done -->
- **대상 레포:** team1-chatguard-infra <!-- app | infra | config 중 하나 -->
- **생성일:** 2026-06-08
- **담당:** cjc

## 목표 (한 줄)

이 작업이 끝나면 무엇이 가능해지는지 한 문장으로.

## 배경 / 참조

- 관련 설계: `DESIGN.md` ○○절
- 관련 규칙: `CLAUDE.md` ○번
- (필요 시) 이전 작업: `tasks/000X-...md`

## 할 일 (구체적 단계)

1. …
2. …
3. …

## 완료 기준 (Definition of Done)

- [ ] 무엇이 동작/존재하면 끝인지 체크 가능한 형태로
- [ ] (예) `terraform plan`이 에러 없이 통과
- [ ] (예) PR 생성 + 리뷰 반영

## 제약 / 주의

- CLAUDE.md 절대 규칙 준수 (리전·prefix·권한경계 `TeamRuntimeBoundary` 등)
- 되돌릴 수 없는 작업(destroy/prod) 전 사람 확인
- 쓰기는 이 작업의 대상 레포 한 곳·한 PR
