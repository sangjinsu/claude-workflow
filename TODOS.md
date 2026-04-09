# TODOS

## Workflow Builder — Phase 2

- ~~**Priority:** P1
  워크플로우 레코더 (세션 자동 캡처)~~ **Completed v0.2.0**

- ~~**Priority:** P1
  `type: ai` step~~ **Completed v0.2.0**

- **Priority:** P2
  변수 override `--set KEY=VALUE` — 실행 시 config.base 값을 runtime에서 override
  *Deferred from /autoplan DX review*

- ~~**Priority:** P2
  config profiles (dev/staging/prod)~~ **Completed v0.2.1**

- **Priority:** P2
  `run` 실행 전 commands 표시+확인 절차 — 공유 워크플로우 보안
  *Deferred from /autoplan Eng review: CRITICAL security finding*

- ~~**Priority:** P2
  비결정성 검증~~ **Completed v0.3.0** (validate --repeat N)

- **Priority:** P3
  `.claude/commands/` 대비 차별점 문서화
  *Deferred from /autoplan CEO review*

- ~~**Priority:** P3
  type: approval step~~ **Completed v0.3.1** (autoplan review에 따라 단독 구현)

- ~~**Priority:** Phase 4+
  step output → next step pipeline (type: api 등의 prerequisite)~~ **Completed v0.4.0** (type: api의 파운데이션)

- **Priority:** Phase 4+
  type: api — step output pipeline 위에 구현 (now unblocked)
  *Deferred: 개별 타입 구현*

- **Priority:** Phase 4+
  type: skill — RFC 필요 (실행 메커니즘, 에러 신호 프로토콜, 결정성 검증 호환성 미정)
  *Deferred: /autoplan review found undefined execution model*

- **Priority:** 별도 플러그인
  type: mcp — "외부 의존성 없음" 원칙 위반
  *Deferred: claude-workflow-experimental 등 별도 플러그인이 적합*

- ~~**Priority:** P3
  병렬 실행 (depends_on이 없는 step 동시 실행)~~ **Completed v0.3.0**

- **Priority:** P4
  hooks (SessionStart, PreToolUse) 연동

- **Priority:** P4
  MCP 서버 연동

## Known Issues (from /qa 2026-04-08)

- **Priority:** P3
  ISSUE-001: Skill tool cache staleness — mid-session SKILL.md edits not reflected in `/workflow-builder` invocations. Workaround: restart session or read file directly.
  *Category: operational/external. Cannot be fixed in this project.*

- **Priority:** P4
  ISSUE-002: deploy.yaml template requires kubectl — first-experience friction for non-k8s users.
  *Category: content/design. Working as intended; blank.yaml is the zero-dep starting point.*

## Completed
