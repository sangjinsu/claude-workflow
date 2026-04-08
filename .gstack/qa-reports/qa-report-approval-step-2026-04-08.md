# QA Report: type: approval step

**Date:** 2026-04-08
**Version:** 0.3.1
**Scope:** approval-demo.yaml execution with deny path

## Test Case: Deny path verification

### Setup
- Workflow: `.workflows/approval-demo.yaml`
- Profile: none (base values used)
- Expected: confirm step prompts user, deny triggers on_failure: abort

### Execution

**Step 1/5 announce (command):** PASS
```
===============================
재시작 작업 시작
Service: my-service
Namespace: production
===============================
```

**Step 2/5 pre-check (command):** PASS
```
현재 시각: Wed Apr  8 21:35:57 KST 2026
사용자: sangjinsu
```

**Step 3/5 confirm (approval):** PASS
- Message after substitution: "my-service (production)에 재시작를 실행하시겠습니까?"
- Variable substitution verified: `${SERVICE_NAME}` → "my-service", `${NAMESPACE}` → "production", `${ACTION}` → "재시작"
- AskUserQuestion invoked with 2 options
- User selected: "B) 거부"
- Result: Step marked as failed with deny

**Expected abort triggered:** PASS
- on_failure: abort honored
- execute step (4/5) NOT executed ✓
- summary step (5/5) NOT executed ✓
- Workflow stopped cleanly

### Follow-up QA (Completed 2026-04-08, same day)

All 4 untested scenarios verified in a follow-up run:

**1. Approve path: PASS**
- Re-ran approval-demo, selected approve
- execute step ran (echo "재시작 완료")
- summary AI step ran (suggested next health check)
- 5/5 steps succeeded

**2. Timeout behavior: DOCUMENTED**
- SKILL.md updated: "type: approval은 사용자 응답까지 무기한 blocking. prompt-only 아키텍처에서 timeout 메커니즘은 지원하지 않는다. 자동화 시나리오는 Phase 4+."
- Decision: MVP supports interactive use only.

**3. Parallel exception: PASS**
- New workflow: `.workflows/approval-parallel-test.yaml`
- Level 0 contains [parallel-cmd-a, parallel-cmd-b, approval-at-level-0]
- Verification: command 2 executed in same message (parallel Bash calls), approval executed in separate message (alone)
- After-all step ran after all 3 completed
- 4/4 steps succeeded

**4. on_failure: continue: PASS**
- New workflow: `.workflows/approval-continue-test.yaml`
- approval step has `on_failure: continue`
- User selected deny
- after-approval step ran despite deny
- 3/3 steps progressed

### Final Health Score

| Category | Score |
|----------|-------|
| Approval step execution (deny) | 100 |
| Approval step execution (approve) | 100 |
| Variable substitution in message | 100 |
| AskUserQuestion integration | 100 |
| on_failure: abort (deny → stop) | 100 |
| on_failure: continue (deny → next) | 100 |
| Parallel exception (approval alone) | 100 |
| Timeout behavior documented | 100 |
| **Overall** | **100/100** |

## Issues Found

None. `type: approval` works as specified.

## Health Score

| Category | Score |
|----------|-------|
| Approval step execution | 100 |
| Variable substitution in message | 100 |
| AskUserQuestion integration | 100 |
| on_failure: abort honored | 100 |
| **Overall** | **100/100** |

## Recommendation

Ship as-is. Consider adding follow-up QA for the untested scenarios above in next release.
