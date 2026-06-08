---
name: pr-review
description: 팀원의 Pull Request나 변경 diff를 team1-chatguard 규칙·설계 기준으로 검토할 때 사용. "PR 리뷰", "이 변경 점검해줘", 머지 전 확인 요청 시 트리거.
---

# PR 리뷰 체크리스트

목적: 팀원 PR을 `CLAUDE.md` 규칙과 `DESIGN.md` 설계 기준으로 점검해 **위반·위험만** 짚는다. 코드를 직접 고치지 말고, 사람이 코멘트로 옮길 수 있게 항목별로 보고한다.

## 먼저 읽기

- 해당 레포의 `CLAUDE.md` (+ 가능하면 `../team1-chatguard-context/CLAUDE.md`, `DESIGN.md`)
- PR의 변경 파일(diff)

## 점검 항목 — 위반은 `[BLOCK]`, 권고는 `[NIT]`

### 절대 규칙

- 서울 리전(ap-northeast-2) 외 리소스 없음
- 리소스 이름 `team1-{env}-`, 태그 `Team=team1` 부착
- (Terraform) IAM role 이름 `team1-*` + 권한경계 `TeamRuntimeBoundary` 부착
- 시크릿·Access Key가 코드·state에 없음 (`.gitignore` 확인)
- AWS 인증 OIDC (Access Key를 Secret에 넣지 않음)

### 소유권 경계 (Terraform vs GitOps)

- 한 리소스를 양쪽이 동시에 만들지 않음
- ALB를 Terraform이 직접 만들지 않음(LBC가 생성)
- Route53은 ExternalDNS / Terraform 중 한쪽만
- Deployment 등 앱 리소스는 config(GitOps)에만

### 구조

- 환경을 브랜치로 나누지 않음(디렉터리/오버레이/이미지 태그)
- infra / platform-addons 분리 유지
- Helm·add-on 버전 고정(`x.y.z`)

### 설계 일치 (DESIGN.md)

- 메시지 스키마·메트릭 이름·ULID 발급 등 명세와 일치
- 컴포넌트 책임 경계 준수

## 출력 형식

- 항목별 `[BLOCK]`/`[NIT]` + `파일:라인` + 한 줄 사유 + 근거(`CLAUDE.md ○` / `DESIGN.md ○절`)
- 마지막에 "머지 가능 여부" 한 줄 결론
