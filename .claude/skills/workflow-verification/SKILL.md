---
name: workflow-verification
description: >
  실행된 워크플로우의 구조 정합성, step output 참조 계약, 결정성을 교차 비교로
  검증하고 QA 리포트를 생성하는 스킬. workflow-verifier 에이전트가 사용.
  "검증", "verify", "QA", "determinism", "워크플로우 리포트", "품질 체크" 언급 시
  반드시 이 스킬로 처리. 파이프라인 실행 후 결과 신뢰도 확인이 필요한 모든 상황에
  우선 트리거. validate 서브커맨드의 핵심 단계.
---

# Workflow Verification Skill

파이프라인 실행 결과를 증거 기반으로 검증하고 QA 리포트를 작성하는 절차. `workflow-verifier` 에이전트가 사용한다.

## 언제 이 스킬을 사용하나

- Executor가 `_workspace/03_executor_summary.yaml`을 생성한 직후 (final 모드)
- 각 레벨 완료 직후 부분 검증 (incremental 모드)
- `/workflow-builder validate <name> --repeat N` (determinism 모드)
- 사용자가 "이 실행이 정상이었는지 확인해줘"라고 요청할 때

## 기본 원칙 (Why)

**1. 교차 비교가 진실이다**: "step이 실행됐다"는 정보는 반쪽짜리다. Planner가 "이 step은 `${steps.x.output}`을 쓴다"고 했는데, Executor의 `step_outputs`에 x가 없다면 **경계면 버그**다. 이런 종류는 단일 파일만 봐서는 절대 잡히지 않는다.

**2. 증거 없는 결론 금지**: "아마 문제없어 보인다"는 리포트는 가치가 없다. 모든 verdict는 파일·라인·값을 인용한다.

**3. 검증은 읽기 전용**: 검증 중 파일을 바꾸거나 step을 재실행하지 않는다. 재실행은 오케스트레이터의 결정이다.

**4. Health score는 가중치**: 모든 체크가 동등하지 않다. 실행 실패는 치명적, 경고는 감점, 스타일은 무시.

## 체크 목록

### 1. Schema Conformance (30%)

- `_workspace/03_executor_summary.yaml`의 스키마가 유효한가?
- `total_steps`가 `01_parser_output.yaml`의 `steps[]` 길이와 일치하는가?
- `succeeded + failed + skipped == total_steps`인가?

실패 시 V-100번대 이슈, severity: high

### 2. Level Ordering (25%)

- `03_executor_log.jsonl`을 순서대로 읽었을 때, 각 step의 level이 **단조 증가** 또는 **동일**인가? (같은 레벨 내 임의 순서 허용)
- 레벨 N+1의 step이 레벨 N의 step 완료 전에 시작된 증거가 있는가?
- Planner의 `levels[]`에 명시된 step이 실제로 그 레벨에 실행됐는가?

실패 시 V-200번대, severity: critical (파이프라인 안전성 위반)

### 3. Output References (20%)

각 step의 `commands`/`prompt`/`message`에 포함된 `${steps.<id>.output}` 참조에 대해:

- 참조 대상 id가 `03_executor_summary.yaml`의 `step_outputs`에 존재하는가?
- 참조 대상이 현재 step 실행 **이전에** 완료됐는가? (로그 시간 대조)
- 참조 대상이 `skipped`면 빈 문자열이 예상되는가? (경고로 표시)

실패 시 V-300번대, severity: high

### 4. Determinism (10%) — determinism 모드에서만

`previous_runs`와 현재 `03_executor_summary.yaml`을 비교:

- 각 step의 `step_outputs[id]`가 모든 run에서 동일한가?
- 레벨별 step 배치가 동일한가?
- 치환된 commands가 모든 run에서 동일한가?

**주의:** `$(date)`, `$(whoami)` 같은 비결정적 쉘 호출은 **예외**. Parser/Planner 단계의 치환 결과만 비교하고, Bash 실행 stdout은 비교 대상에서 제외한다 (단, 사용자가 deterministic 가정으로 작성한 경우는 사용자 책임).

일치 시 `determinism: pass`, 불일치 시 어떤 step의 어떤 부분이 달라졌는지 V-400번대로 기록.

### 5. Side Effects (10%)

- `aborted_at`이 null이 아닌 경우, 중단 사유가 로그에 명시되어 있는가?
- `fail` 상태 step의 `on_failure`가 `continue`인데 하류 step이 실행됐는가? (정상)
- `on_failure == abort`인데 이후 step이 실행됐다면 버그

### 6. Security (5%)

- Parser의 `warnings[]`에 `shell_metachar`가 있었다면 security 카테고리로 재표시
- 치환되지 않은 `${}` 패턴이 commands에 남아있다면 경고

## 절차

### Step 1 — 입력 파일 로드

```
parser = read_yaml(inputs.parser_output)
planner = read_yaml(inputs.planner_output)
log = read_jsonl(inputs.executor_log)
summary = read_yaml(inputs.executor_summary)
```

파일이 손상됐거나 없으면 즉시 `verdict: fail`, V-000 기록 후 중단.

### Step 2 — 교차 비교

```
# Parser ↔ Summary
assert parser.steps.len == summary.total_steps

# Planner ↔ Log
for log_entry in log:
    expected_level = planner.step_metadata[log_entry.step_id].level
    assert log_entry.level == expected_level

# Planner ↔ Summary (output refs)
for step in parser.steps:
    for ref in extract_output_refs(step):
        assert ref in summary.step_outputs
        assert ref in planner.step_metadata[step.id].upstream_closure
```

### Step 3 — Health Score 계산

```
score = 100
for issue in issues:
    if issue.severity == "critical": score -= 25
    elif issue.severity == "high":   score -= 10
    elif issue.severity == "medium": score -= 5
    elif issue.severity == "low":    score -= 2
score = max(0, score)
```

**Verdict 매핑:**
- score == 100 → `pass`
- 80 ≤ score < 100 → `warn`
- score < 80 → `fail`

### Step 4 — 리포트 작성

`_workspace/04_verifier_report.md`:

```markdown
# QA Report: <workflow.name>

**Date:** <UTC ISO>
**Mode:** final | incremental | determinism
**Verdict:** <pass | warn | fail>
**Health Score:** <0-100>

## Summary
- Total steps: <N>
- Succeeded: <N>
- Failed: <N>
- Skipped: <N>

## Checks
| Check | Result | Weight | Notes |
|-------|--------|--------|-------|
| Schema Conformance | pass | 30% | - |
| Level Ordering | pass | 25% | - |
| Output References | warn | 20% | V-301: ... |
| Determinism | n/a | 10% | not in determinism mode |
| Side Effects | pass | 10% | - |
| Security | pass | 5% | - |

## Issues
### V-301 (medium) — Output Reference
- **Step:** report
- **Evidence:** `_workspace/03_executor_log.jsonl:15` step_id=fetch-count has empty output but referenced by report
- **Detail:** fetch-count was skipped; report referenced ${steps.fetch-count.output} which resolved to empty string
- **Fix Hint:** Add fallback value or make report depend on non-skipped step

## Recommendations
- ...
```

병렬로 구조화된 `_workspace/04_verifier_verdict.yaml`도 작성 (에이전트 간 통신용).

### Step 5 — 피드백 루프

심각한 이슈 발견 시 오케스트레이터에게 알림:
- `critical` severity → 재실행 필요 여부 문의
- `high` × 2 이상 → 하네스 진화 트리거 (에이전트/스킬 수정 제안)

## 테스트 체크리스트

| 시나리오 | 기대 verdict |
|---------|-------------|
| 모든 step 성공 | pass, score=100 |
| 1 step fail + on_failure=continue | warn (정상 분기) |
| 1 step fail + on_failure=abort 이후 실행 로그 존재 | fail (side effect 위반) |
| skipped step의 output을 하류가 참조 | warn, V-301 |
| determinism: 3회 run 일치 | pass |
| determinism: 2회 중 1회 commands 달라짐 | fail, V-401 |
| executor_summary 누락 | fail, V-000 |

## 실패 시 행동

- 입력 파일 손상 → V-000, 중단
- 검증 중 자체 에러 → 리포트에 "verifier internal error" 기록, 통과/실패 판정 보류
- 재호출 시 이전 리포트 보존 (덮어쓰기 금지, `_report_strict.md` 같이 새 파일)
