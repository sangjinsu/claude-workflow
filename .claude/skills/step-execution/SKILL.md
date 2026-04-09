---
name: step-execution
description: >
  계획된 병렬 레벨을 순서대로 실행하고, command/ai/approval step을 각 타입에 맞게
  처리하며, step output을 캡처하고 on_failure 정책을 따르는 스킬. step-executor
  에이전트가 사용. "step 실행", "명령 실행", "파이프라인 run", "병렬 실행",
  "on_failure", "output 캡처" 언급 시 반드시 이 스킬로 처리. 워크플로우 실행의
  심장부이므로 run 요청에는 무조건 트리거.
---

# Step Execution Skill

계획된 파이프라인을 실제로 실행하는 절차. `step-executor` 에이전트가 사용한다.

## 언제 이 스킬을 사용하나

- Planner가 `_workspace/02_planner_output.yaml`을 생성한 후
- 사용자가 `/workflow-builder run <name>`을 요청했을 때의 실행 단계
- 부분 재실행 (특정 step부터 다시)

## 기본 원칙 (Why)

**1. commands는 계약이다**: 사용자가 YAML에 쓴 명령을 수정하면 결정성과 신뢰가 모두 깨진다. "이게 더 안전할 것 같다"는 유혹은 거부한다.

**2. exit code가 진실이다**: stdout에 "OK"가 있어도 exit code가 0이 아니면 실패다. 반대도 마찬가지다.

**3. 병렬은 디스패치 단위**: "병렬"은 같은 메시지에서 여러 Bash 호출을 한다는 뜻이다. 실행 결과 수집은 모두 완료된 후에 한다.

**4. approval은 단독**: 여러 approval이 동시에 뜨면 UX가 붕괴한다. 같은 레벨이어도 solo_execution 플래그가 있으면 순차 처리한다.

**5. 실행 전 확인이 보안이다**: Trust Model상 commands는 사용자 권한으로 실행된다. 치환 완료 후 사용자에게 최종 commands를 보여주고 승인을 받는 것이 공유받은 워크플로우의 마지막 방어선이다.

## 절차

### Step 0 — 입력 로드

- `_workspace/01_parser_output.yaml` 읽기 → `merged_variables`, `steps[]`
- `_workspace/02_planner_output.yaml` 읽기 → `levels[]`, `step_metadata`
- 실행 로그 초기화: `_workspace/03_executor_log.jsonl` (기존 파일이면 백업)
- `step_outputs: dict` 초기화 (메모리)

### Step 1 — 변수 치환 (실행 직전)

**각 레벨 실행 직전에** 해당 레벨의 모든 step에 대해 치환:

1. `commands[]` 각 문자열에서:
   - `${KEY}` → `merged_variables[KEY]` (없으면 경고+원본 유지)
   - `${steps.<id>.output}` → `step_outputs[id]` (없으면 빈 문자열)
2. `$()`, `$VAR`, 백틱은 **건드리지 않음**
3. AI step의 `prompt`, approval step의 `message`에도 동일 규칙 적용

**크기 검증:**
- `step_outputs[id]`가 10KB 초과 → 경고
- 100KB 초과 → 에러, 전체 중단 (on_failure 무시)

### Step 2 — 실행 전 확인 (interactive 모드만)

모든 step의 치환된 commands를 한 번에 표시:

```
실행할 워크플로우: <workflow.name>
사용된 파일: <resolved path>

Step 1: <step.name> (id: <step.id>)
  $ <치환된 command 1>
  $ <치환된 command 2>

Step 2: ...

계속 실행하시겠습니까?
```

사용자가 거부하면 즉시 종료. `auto` 모드는 이 단계를 건너뛴다.

### Step 3 — 레벨 루프

Planner가 계산한 순서대로 레벨 0, 1, 2, ... 실행.

**각 레벨의 처리:**

```
for level in levels:
    print(f"⚡ 레벨 {level.level} 시작: {level.steps}")
    
    # solo step 분리
    solo_steps = [s for s in level.steps if metadata[s].solo_execution]
    parallel_steps = [s for s in level.steps if not metadata[s].solo_execution]
    
    # 병렬 실행 (같은 메시지에 여러 Bash 호출)
    if parallel_steps:
        dispatch_parallel(parallel_steps)
    
    # solo 순차 실행
    for s in solo_steps:
        dispatch_single(s)
```

### Step 4 — type: command 실행

```
▶ Step [N]/[total]: <step.name> (id: <step.id>)
  [description if exists]
```

각 command를 Bash 도구로 실행 (timeout: 300000ms).

**exit code 처리:**
- `0`: 다음 command로 진행
- `!=0`:
  - `on_failure == abort`:
    ```
    Step <id> 실패 (exit code: <code>). 실패한 명령: <command>. 워크플로우를 중단합니다.
    ```
    → 전체 중단
  - `on_failure == continue`:
    ```
    Step <id> 실패했지만 continue 설정으로 다음 step으로 진행합니다.
    ```
    → 해당 step은 fail로 마킹, 다음 step 진행

**Output 캡처:** 마지막 command의 stdout 전체. trailing newline 제거. `step_outputs[id]`에 저장.

### Step 5 — type: ai 실행

```
🤖 Step [N]/[total]: <step.name> (id: <step.id>, type: ai)
```

치환된 prompt를 처리하고 응답을 생성. 응답 텍스트를 `step_outputs[id]`에 저장.

AI step은 병렬 실행 불가 — 같은 레벨이어도 단독 처리.

**실패 (응답 생성 불가):**
- `on_failure == abort` → 중단
- `on_failure == continue` → `step_outputs[id] = ""`, 진행

### Step 6 — type: approval 실행

```
🛑 Step [N]/[total]: <step.name> (id: <step.id>, type: approval)
```

AskUserQuestion 도구 사용:
- question: 치환된 message
- options:
  - `default: approve` → ["승인 (계속 진행)", "거부 (워크플로우 중단)"]
  - `default: deny` → ["거부 (워크플로우 중단)", "승인 (계속 진행)"]

**응답 처리:**
- 승인 → `step_outputs[id] = "approved"`, 다음 step
- 거부:
  - `on_failure == abort` → 중단, `step_outputs[id] = "denied"`
  - `on_failure == continue` → 계속, `step_outputs[id] = "denied"`

**병렬 제약:** approval은 항상 solo. 동시 승인 요청 금지.

**Timeout:** 무기한 blocking. 자동화가 필요하면 외부 override (Phase 4+).

### Step 7 — 스킵된 step

지원하지 않는 type의 step:

```
⏭ Step [N]/[total]: <step.name> (건너뜀 - 미지원 타입: <type>)
```

`step_outputs[id] = ""`, 의존성 해결에서는 "완료"로 간주 (Planner가 이미 처리).

### Step 8 — 로그 기록

각 step 실행 직후 `_workspace/03_executor_log.jsonl`에 한 줄:

```json
{"step_id":"welcome","level":0,"type":"command","status":"ok","exit_code":0,"output":"...","duration_ms":123,"on_failure":"abort","error":null}
```

### Step 9 — 최종 요약

```
워크플로우 '<workflow.name>' 실행 완료 (<N>/<total> steps 성공, <M> 건너뜀)
```

`_workspace/03_executor_summary.yaml`에 구조화된 요약 저장. Verifier가 이 파일로 검증 시작.

## 병렬 실행 예시

레벨 0에 `[check-disk, check-memory, check-uptime]`가 있으면:

```
⚡ 레벨 0 병렬 실행: check-disk, check-memory, check-uptime
```

세 Bash 호출을 **같은 메시지**에 배치한다. 세 결과 모두 수신 후 다음 레벨로.

## 테스트 체크리스트

| 시나리오 | 기대 |
|---------|------|
| command 성공 | log status=ok, 다음 step |
| command 실패 + abort | 중단, summary.status=aborted |
| command 실패 + continue | fail 마킹, 진행 |
| AI step | 응답 캡처, output 저장 |
| approval 승인 | output="approved" |
| approval 거부 + abort | 중단 |
| 같은 레벨 4개 command | 같은 메시지에서 4번 Bash 호출 |
| 같은 레벨에 approval + command | approval만 solo, command는 병렬 |
| `${steps.x.output}` 참조 | 치환 성공 |
| output 15KB | warning, 진행 |
| output 150KB | error, 중단 |

## 실패 시 행동

- 치명적 (100KB 초과, 치환 실패 등): 즉시 중단, summary.status=aborted
- 개별 step 실패: on_failure 정책 따름
- 사용자 Ctrl+C: 현재 레벨 완료 후 graceful stop (prompt-only이므로 실제로는 Claude Code의 중단 메커니즘에 의존)
