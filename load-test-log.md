# 부하테스트 실험 로그 (load-test-log)

**규율: 실험 1건 = 1줄, 실험 직후 즉시 append.** (사후 일괄 기입 금지 — 판정·병목·다음 조정을 잊기 전에 그 자리에서 기록한다.)

## 실험 로그

| ID  | 시각       | 시나리오 | 부하 프로파일      | 조정상태(SHA/인스턴스/설정)                               | 검열지연 p95 | 전송 p95              | 에러율 | 스케일(N→M, 시간)     | 판정(SLO)           | 병목/원인 | 다음 조정                          |
| --- | ---------- | -------- | ------------------ | --------------------------------------------------------- | ------------ | --------------------- | ------ | --------------------- | ------------------- | --------- | ---------------------------------- |
| E1  | 7/02 17:25 | smoke    | VU3 · 1m · 방1,2,3 | app PR#77 SHA / prod 기본(chat min3·worker floor2·cap200) | –            | (http p95 135ms 참고) | 0%     | 스케일 없음(예상대로) | 통과(3/3 threshold) | –         | 내일 베이스라인 A(접속 램프업)부터 |

기입 예시(형식 참고용):
`| B-01 | 07-03 14:20 | scenario-b | 50VU·1msg/s·15m | config@abc1234 / m7g.large / WS_CAP=200 | 3.2s | 180ms | 0.1% | 1→4, 3m | PASS | 큐 픽업 지연 | 송신율 2배 |`

## 발견 사항 (DESIGN 드리프트 — 7/02 선스캔)

코드와 DESIGN 서술이 어긋나는 4건. **모두 코드가 진실(SSoT 갱신 필요)** — 부하테스트는 아래 실코드 기준으로 진행한다.

① **WS 토큰 전달 = `Sec-WebSocket-Protocol` 헤더** (쿼리는 `room_id`만) — DESIGN D21·D26은 `?token=` 쿼리로 서술, app CLAUDE.md("WS 접속 `?token=&room_id=`")도 동일하게 stale. 근거: `WebSocketAuthInterceptor.java:34`, `frontend/src/hooks/useChat.js:127`. _DESIGN 반영은 별도 PR(팀 합의) 대상._

② **시드 계정 = `admin` + `cjc`/`ykh`/`ssm`/`lhc`/`kwy`** — DESIGN A-2는 `user1`~`user6`으로 서술. 근거: `V1__InitialSchema.sql:74-81`. (평문 비밀번호는 미커밋 원칙 — 이 로그에도 기재 금지.) _DESIGN 반영은 별도 PR(팀 합의) 대상._

③ **close `1008`은 서버가 발신하지 않음** — 인증 실패는 WS close frame이 아니라 핸드셰이크 **HTTP 401 거부**로 처리. 서버 발신 close는 `1013`(cap 초과)·`1001`(드레인)뿐 — DESIGN A-3 close code 표와 상이. 근거: `WebSocketAuthInterceptor.java:68-71`, `ChatRoomSessionRegistry.java:44-50, :135`. _DESIGN 반영은 별도 PR(팀 합의) 대상._

④ **`POST /api/rooms` 실존** — DESIGN A-2는 "방 생성 API 없음"으로 서술. 근거: `RoomRestController.java:36-39`. (부하테스트 방 샤딩용 방 추가 생성에 활용 가능.) _DESIGN 반영은 별도 PR(팀 합의) 대상._

## SLO 기준 (판정 컬럼 근거 — DESIGN B-1)

- **검열 반영 p95 < 5s (핵심)**
- 전송 p95 < 300ms
- 성공률 ≥ 99.9%

※ 위 수치는 출발값 합의(D5) — 1차 부하테스트 측정 후 재확정.

## 참고 링크

- k6 스크립트: [app PR #77](https://github.com/CLD-05/team1-chatguard-app/pull/77) — `load-test/{smoke.js, scenario-a-connections.js, scenario-b-messages.js, lib/common.js}`
- 대시보드 B-3① 충족: [config PR #62](https://github.com/CLD-05/team1-chatguard-config/pull/62) — 헤드라인 큐 깊이 패널(`redis_key_size{key="mod:queue"}`) + prod 라벨 정규식화
