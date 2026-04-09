---
name: pipeline-orchestration
description: >
  YAML 워크플로우 파이프라인 실행을 조율하는 오케스트레이터 스킬. workflow-parser,
  dependency-planner, step-executor, workflow-verifier 4명의 전문 에이전트 팀을
  구성하여 파싱 → 계획 → 실행 → 검증 라이프사이클을 수행한다. "워크플로우 실행",
  "pipeline", "파이프라인 조율", "run workflow", "워크플로우 파이프라인", "팀으로 워크플로우
  실행", "workflow-builder run"의 고급 경로, "다시 실행", "재실행", "부분 재실행",
  "이전 결과 기반으로", "보완", "수정"이 언급되면 반드시 이 스킬로 처리. 단일
  workflow-builder SKILL.md를 사용하는 단순 실행과 달리, 이 스킬은 전문가 팀 분산
  실행을 제공하며 중간 산출물을 파일로 남겨 감사 추적과 부분 재실행을 가능하게 한다.
---

# Pipeline Orchestration Skill

YAML 워크플로우 파이프라인을 전문 에이전트 팀으로 조율하는 오케스트레이터. 파이프라인 실행 라이프사이클 전체 — 파싱, 계획, 실행, 검증 — 를 관장한다.

## 언제 이 스킬을 사용하나

- 사용자가 **"파이프라인 실행"**, **"워크플로우를 팀으로 실행"** 같이 팀 기반 실행을 명시했을 때
- 이전 실행이 실패했거나 부분 수정 요청("특정 step부터 다시")이 들어왔을 때
- `/workflow-builder run`의 단순 경로 대신 **감사 추적과 검증 리포트**가 필요할 때
- 결정성 검증(`validate --repeat N`)을 팀 협업으로 수행할 때
- 사용자가 "하네스로 실행해줘", "전문가 팀으로 돌려줘"라고 말할 때

> **단순 실행 vs 오케스트레이션**: `skills/workflow-builder/SKILL.md`는 단일 스킬에서 전체 실행을 처리하는 MVP 경로다. 이 오케스트레이터는 중간 산출물 파일, 교차 검증, 재실행을 지원하는 고급 경로다. 사용자가 속도만 원하면 전자를, 신뢰도·추적성·팀 기반 조율을 원하면 후자를 쓴다.

## 팀 구성

| 역할 | 에이전트 | 스킬 | 담당 |
|------|---------|------|------|
| 파서 | `workflow-parser` | `workflow-parsing` | YAML 로드, 변수 병합, 스키마 검증 |
| 플래너 | `dependency-planner` | `dependency-planning` | DAG 분석, 병렬 레벨, 참조 검증 |
| 실행자 | `step-executor` | `step-execution` | command/ai/approval 실행, output 캡처 |
| 검증자 | `workflow-verifier` | `workflow-verification` | 결과 검증, QA 리포트 |
| 리더 | (이 스킬) | — | 조율, 데이터 전달, 에러 처리, 사용자 보고 |

**실행 모드:** 에이전트 팀. 팀 통신 도구(`TeamCreate`, `SendMessage`, `TaskCreate`)가 사용 가능하면 팀 모드로 실행한다. 사용 불가능하면 **서브 에이전트 폴백**(`Agent` 도구로 순차 호출 + 파일 기반 전달)으로 동작한다. 폴백은 팀 통신만 없을 뿐 파이프라인과 산출물은 동일하다.

## 작업 공간

모든 중간 산출물은 `_workspace/` 폴더에 저장. 최종 리포트만 사용자 지정 위치에 출력.

```
_workspace/
├── 01_parser_output.yaml
├── 02_planner_output.yaml
├── 03_executor_log.jsonl
├── 03_executor_summary.yaml
├── 04_verifier_report.md
└── 04_verifier_verdict.yaml
```

**네이밍 컨벤션:** `{phase}_{agent}_{artifact}.{ext}`. 파일은 사후 검증과 부분 재실행을 위해 실행이 끝나도 보존한다.

## Phase 0 — 컨텍스트 확인 (초기 / 후속 / 부분 재실행 판별)

파이프라인을 시작하기 전에 `_workspace/` 상태를 확인한다:

| 상태 | 판별 조건 | 행동 |
|------|----------|------|
| **초기 실행** | `_workspace/` 미존재 | 새로 생성 후 Phase 1부터 |
| **새 실행** | `_workspace/` 존재 + 사용자가 **다른 워크플로우** 또는 **다른 profile/override** 지정 | `_workspace/`를 `_workspace_prev_<timestamp>/`로 이동하고 Phase 1부터 |
| **부분 재실행** | `_workspace/` 존재 + 사용자가 **"실패한 step부터 다시"** 같은 요청 | Parser/Planner 산출물 재사용, Executor만 재호출 (재개 모드) |
| **재검증** | `_workspace/` 존재 + 사용자가 **"검증만 다시"** 요청 | Verifier만 재호출 |
| **부분 수정** | `_workspace/` 존재 + 사용자가 **"YAML 수정했어"** 알림 | 파일 mtime 비교 후 영향받는 Phase부터 재실행 |

어느 경우에도 `_workspace_prev_*`는 **삭제하지 않는다** (감사 추적).

## Phase 1 — 파싱

**입력:**
- `workflow_name`: 사용자가 지정한 이름
- `profile`: (선택) profile 이름
- `overrides`: (선택) `KEY=VALUE` 맵

**행동 (팀 모드):**
```
TeamCreate(name="pipeline-team", members=[workflow-parser, dependency-planner, step-executor, workflow-verifier])
TaskCreate(
  subject="YAML 파싱 및 변수 병합",
  owner="workflow-parser",
  metadata={input: {workflow_name, profile, overrides}, output: "_workspace/01_parser_output.yaml"}
)
```

**행동 (서브 폴백):**
```
Agent(
  subagent_type="workflow-parser",
  model="opus",
  prompt="<workflow_name, profile, overrides>를 파싱하고 _workspace/01_parser_output.yaml에 저장하라."
)
```

**완료 조건:** `_workspace/01_parser_output.yaml` 존재 + `status: ok`.

**실패 처리:** `status: error`면 사용자에게 errors[] 보고 후 중단. 재시도하지 않는다 (Parser 에러는 대부분 결정적).

## Phase 2 — 계획

**입력:** `_workspace/01_parser_output.yaml`

**행동 (팀 모드):**
```
SendMessage(
  to="dependency-planner",
  content="Parser 출력이 _workspace/01_parser_output.yaml에 준비됐다. DAG 계산 후 _workspace/02_planner_output.yaml에 저장하라."
)
TaskCreate(subject="DAG 계획 수립", owner="dependency-planner", blockedBy=[<parser task id>])
```

**완료 조건:** `_workspace/02_planner_output.yaml` 존재 + `status: ok` + `cycles: []`.

**실패 처리:**
- 순환 감지 → 사용자에게 cycles 목록 보고 후 중단
- 참조 오류 → fix_hint와 함께 보고 후 중단
- Parser 재조회 불필요 (이미 검증 완료)

## Phase 3 — 실행

**입력:** `_workspace/01_parser_output.yaml`, `_workspace/02_planner_output.yaml`, `mode: interactive|auto`

**행동 (팀 모드):**
```
SendMessage(
  to="step-executor",
  content="Planner 출력이 준비됐다. 레벨 순서대로 실행하고 _workspace/03_executor_*에 기록하라."
)
TaskCreate(subject="워크플로우 실행", owner="step-executor", blockedBy=[<planner task id>])
```

**실행 중 실시간 업데이트:** Executor는 각 레벨 시작/종료 시 오케스트레이터에게 `SendMessage`로 진행 상황을 알린다. 오케스트레이터는 이 메시지를 사용자에게 투명하게 전달한다.

**완료 조건:** `_workspace/03_executor_summary.yaml` 존재. `status: completed | aborted | partial` 중 하나.

**실패 처리:**
- `aborted`: 중단 사유와 `aborted_at`을 사용자에게 보고. Verifier는 **여전히 실행**(부분 검증 모드)
- `partial`: on_failure=continue로 일부 실패. Verifier가 이를 리포트에 반영
- Executor가 Planner/Parser에게 재조회 요청 → 해당 에이전트만 재호출, 전체 재시작 아님

## Phase 4 — 검증

**입력:** 1~3단계의 모든 산출물

**행동 (팀 모드):**
```
SendMessage(
  to="workflow-verifier",
  content="실행 완료. 모든 _workspace/* 파일을 교차 검증하고 04_verifier_*에 리포트를 남겨라. mode=final."
)
TaskCreate(subject="QA 검증", owner="workflow-verifier", blockedBy=[<executor task id>])
```

**완료 조건:** `_workspace/04_verifier_report.md` + `04_verifier_verdict.yaml` 존재.

**결과 처리:**
- `pass` (100): 성공 보고
- `warn` (80~99): 이슈 목록과 함께 성공 보고
- `fail` (< 80): 실패 보고, 사용자에게 재실행 여부 문의 (Phase 5-5 자동 재실행 정책 참조)

## Phase 5 — 최종 보고

사용자에게 출력:

```
✅ 파이프라인 '<workflow.name>' 실행 완료
──────────────────────────────────
Verdict:     <pass|warn|fail>
Health:      <score>/100
Steps:       <succeeded>/<total> (skipped: <N>)
Duration:    <executor summary duration>
Report:      _workspace/04_verifier_report.md
```

이슈가 있으면 주요 이슈 상위 3개를 요약한다. 전체 리포트는 파일 경로로 안내.

**팀 정리 (팀 모드):** 실행이 끝나면 `TeamDelete`로 팀을 해체. 다음 실행은 새 팀을 구성한다 (Phase 간 독립성).

## 에러 핸들링 정책

| 에러 유형 | 정책 |
|----------|------|
| Parser 에러 (결정적) | 1회 재시도 금지. 사용자에게 errors[] 보고 후 중단 |
| Planner 순환 감지 | 재시도 금지. cycles 그대로 보고 |
| Executor command 실패 | step의 on_failure 따름. abort면 중단, continue면 진행 |
| AI step 실패 | on_failure 따름. 재시도 없음 (응답은 결정적이지 않을 수 있음) |
| approval 거부 | on_failure 따름 |
| Verifier fail (< 80) | 사용자에게 재실행 여부 문의. 자동 재실행 금지 (부작용 위험) |
| 에이전트 간 메시지 실패 | 1회 재시도. 재실패 시 해당 에이전트 산출물 없이 진행, 리포트에 누락 명시 |
| 파일 I/O 실패 | 치명적. 즉시 중단 |

**상충 데이터 처리:** Parser와 Executor가 같은 step의 정보를 다르게 가졌다면(예: Parser는 type=command인데 Executor 로그는 type=ai로 실행), 삭제하지 말고 **출처 병기**로 Verifier 리포트에 명시.

## 자동 재실행 정책 (기본: 비활성화)

기본값은 **자동 재실행하지 않는다**. 이유:
- command step은 멱등하지 않을 수 있다 (DB 쓰기, 파일 생성)
- AI step 재실행은 비결정적 응답을 만들 수 있다
- approval은 사용자가 거부한 결정을 존중한다

**예외:** 사용자가 명시적으로 `--auto-retry` 같은 플래그를 지정했거나, 워크플로우 자체가 `retry: true`를 선언했을 때만 재실행. 그럼에도 같은 step은 최대 1회만 재시도.

## 테스트 시나리오

### 시나리오 1: 정상 흐름 (hello.yaml)

1. 사용자: "파이프라인으로 hello 워크플로우 실행"
2. Phase 0: `_workspace/` 없음 → 초기 실행
3. Phase 1 (Parser): `hello.yaml` 로드, 변수 병합, `01_parser_output.yaml` 작성
4. Phase 2 (Planner): 3개 step 선형 체인 → 레벨 0, 1, 2
5. Phase 3 (Executor): 레벨 순서대로 실행, `03_executor_log.jsonl` + `summary` 작성
6. Phase 4 (Verifier): 모든 체크 통과, `verdict: pass`, `score: 100`
7. Phase 5: 성공 보고, `04_verifier_report.md` 경로 안내

### 시나리오 2: 순환 참조

1. 사용자: "파이프라인으로 cyclic-bad 실행"
2. Phase 1: Parser 통과 (depends_on 참조 자체는 유효)
3. Phase 2: Planner가 Kahn 알고리즘 진행 불가 → `cycles: [[a, b, c]]`
4. Phase 3: 시작 안 함
5. 사용자에게 cycles 보고 후 중단

### 시나리오 3: 부분 재실행

1. 이전 실행이 step `deploy`에서 실패 (on_failure=abort)
2. 사용자: "deploy부터 다시 실행"
3. Phase 0: 부분 재실행 판별, Parser/Planner 산출물 재사용
4. Phase 3: Executor에 `resume_from: deploy` 플래그 전달. 이전 `step_outputs`를 복원
5. Phase 4: Verifier가 전체 로그(이전 + 이번)를 교차 검증
6. Phase 5: 보고

### 시나리오 4: 결정성 검증

1. 사용자: "parallel-demo 3회 실행해서 결정성 확인"
2. 오케스트레이터가 Phase 1-3을 3회 반복, 각 run을 `_workspace_run1/`, `_workspace_run2/`, `_workspace_run3/`로 분리 저장
3. Phase 4: Verifier를 `mode: determinism`으로 호출, `previous_runs`에 3개 경로 전달
4. Verifier가 step_outputs와 치환된 commands를 비교
5. 일치 시 `determinism: pass`

### 시나리오 5: Verifier fail

1. 실행은 완료되나 Verifier가 V-301 (output reference) 발견
2. `verdict: warn`, `score: 85`
3. Phase 5: 사용자에게 이슈 보고, 재실행 여부 문의 **금지**하고 fix_hint 제공
4. 사용자가 YAML 수정 후 다시 요청하면 "부분 수정" 경로로 재진입

## 재호출 키워드 (description 트리거 보조)

이 스킬은 다음 표현에도 반응한다:
- "다시 실행", "재실행", "retry"
- "실패한 step부터", "이어서 실행"
- "검증만 다시", "리포트 다시"
- "YAML 수정했어", "보완해서 실행"
- "이전 결과 기반으로 개선"
- "결정성 확인", "3회 실행"
- "팀으로 실행", "전문가 팀으로"

## 실패 시 행동

- Phase 0 판별 오류: 안전 기본값(초기 실행)으로 진행, 로그에 기록
- 중간 에이전트 응답 없음: 1회 재시도 → 실패 시 해당 Phase 산출물을 "누락"으로 표시하고 사용자에게 결정 위임
- `_workspace/` 쓰기 실패: 치명적, 즉시 중단 및 권한·디스크 공간 점검 안내
