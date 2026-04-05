# TODOS

## Workflow Builder — Phase 2

- **Priority:** P1
  워크플로우 레코더 (세션 자동 캡처) — 자연어 작업을 YAML 워크플로우로 자동 변환
  *Deferred from /autoplan CEO review: 킬러 기능, 10x 사용자 가치*

- **Priority:** P1
  `type: ai` step — Claude AI 능력을 워크플로우 step에서 활용 (예: 로그 분석 후 조건부 실행)
  *Deferred from /autoplan CEO review: 경쟁 방어 가능한 유일한 차별점*

- **Priority:** P2
  변수 override `--set KEY=VALUE` — 실행 시 config.base 값을 runtime에서 override
  *Deferred from /autoplan DX review*

- **Priority:** P2
  config profiles (dev/staging/prod) — 환경별 변수 세트
  *From CLAUDE.md Phase 2+ 로드맵*

- **Priority:** P2
  `run` 실행 전 commands 표시+확인 절차 — 공유 워크플로우 보안
  *Deferred from /autoplan Eng review: CRITICAL security finding*

- **Priority:** P2
  비결정성 검증 — 동일 워크플로우 반복 실행 테스트
  *Deferred from /autoplan CEO/Eng/DX cross-phase theme*

- **Priority:** P3
  `.claude/commands/` 대비 차별점 문서화
  *Deferred from /autoplan CEO review*

- **Priority:** P3
  type: api, skill, approval step 지원

- **Priority:** P3
  병렬 실행 (depends_on이 없는 step 동시 실행)

- **Priority:** P4
  hooks (SessionStart, PreToolUse) 연동

- **Priority:** P4
  MCP 서버 연동

## Completed
