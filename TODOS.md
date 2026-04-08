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

- **Priority:** P3
  type: api, skill, approval step 지원

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
