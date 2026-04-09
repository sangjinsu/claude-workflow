---
name: dependency-planning
description: >
  파싱된 워크플로우의 depends_on을 DAG로 해석하고, Kahn 알고리즘으로 병렬 실행
  레벨을 계산하며, 순환 참조와 step output 참조의 전이성을 검증하는 스킬.
  dependency-planner 에이전트가 사용. "DAG", "병렬 레벨", "실행 순서", "의존성",
  "순환 감지", "토폴로지 정렬", "step output 참조 검증" 언급 시 반드시 이 스킬로
  처리. Parser 출력을 Executor 입력으로 변환하는 필수 중간 단계.
---

# Dependency Planning Skill

`depends_on` 그래프를 실행 가능한 병렬 레벨로 변환하는 절차. `dependency-planner` 에이전트가 사용한다.

## 언제 이 스킬을 사용하나

- Parser가 `_workspace/01_parser_output.yaml`을 생성한 직후
- `validate` 서브커맨드에서 순환 감지 단계
- Executor가 동적 스킵을 알렸을 때의 재계산

## 기본 원칙 (Why)

**1. 결정성**: 같은 입력 → 같은 레벨. 같은 레벨 내에서는 YAML 정의 순서를 유지한다. 이 원칙이 깨지면 `validate --repeat N`이 의미를 잃는다.

**2. 전이 검증 필수**: `${steps.<id>.output}`이 참조하는 id가 `depends_on`에 **전이적으로** 있어야 한다. 직접 의존이 아니어도 되지만, 같은 레벨 또는 나중 레벨이면 안 된다. Executor가 변수를 치환할 시점에 output이 존재해야 하기 때문이다.

**3. Approval 격리**: approval step이 같은 레벨에 있어도 단독 실행되도록 플래그를 부여한다. 계산은 같이 해도, 실행은 분리한다.

**4. 스킵은 "완료"**: Parser가 skipped로 마킹한 step은 레벨 계산에서 "즉시 완료된 step"으로 간주한다. 하류 step의 레벨 계산에 영향을 주지 않는다.

## 절차

### Step 1 — 그래프 구축

Parser 출력의 `steps[]`에서:

```
nodes = {step.id for step in steps}
edges = {(from, to) for step in steps for from in step.depends_on}
in_degree[id] = len(step.depends_on)
```

`depends_on`에 존재하지 않는 id가 있으면 즉시 에러 (Parser가 놓쳤을 경우의 안전망).

### Step 2 — Kahn 알고리즘으로 레벨 계산

```
level = 0
assigned = {}
while nodes:
    ready = [id for id in nodes if in_degree[id] == 0]
    if not ready:
        # 순환
        break
    # YAML 정의 순서로 정렬 (결정성)
    ready.sort(key=lambda id: original_index[id])
    for id in ready:
        assigned[id] = level
        nodes.remove(id)
        for (_, to) in edges_from(id):
            in_degree[to] -= 1
    level += 1
```

종료 후 `nodes`가 비어있지 않으면 **순환 참조**. 남아있는 모든 id가 순환에 포함된다.

### Step 3 — 순환 리포트

남아있는 id들에 대해, 실제 순환 경로를 추출:

```
for id in remaining:
    visited = set()
    path = []
    current = id
    while current not in visited:
        visited.add(current)
        path.append(current)
        # 의존하는 것 중 아직 미할당인 것 선택
        next = pick_unassigned_dependency(current)
        if next is None:
            break
        current = next
    if current in path:
        cycle = path[path.index(current):]
        cycles.append(cycle)
```

중복 제거 후 `cycles[]`에 기록.

**에러 메시지:**
```
순환 참조 감지: 다음 step들이 순환에 포함됩니다: [a, b, c].
워크플로우를 실행할 수 없습니다.
```

### Step 4 — Step Output 참조 전이 검증

각 step의 `commands`, `prompt`, `message`에서 `${steps.<id>.output}` 패턴을 추출한다.

각 참조에 대해:
1. 참조 대상 id가 존재하는가? (없으면 에러)
2. 참조 대상이 현재 step의 **전이적 upstream**에 포함되는가?

전이적 upstream 계산:
```
def upstream_closure(id):
    result = set()
    stack = list(step_map[id].depends_on)
    while stack:
        up = stack.pop()
        if up not in result:
            result.add(up)
            stack.extend(step_map[up].depends_on)
    return result
```

참조 대상이 `upstream_closure(current_id)`에 없으면:
```
에러: step '<current>'가 '${steps.<ref>.output}'을 참조하지만 depends_on에 (전이적으로도) 없습니다.
```

### Step 5 — Solo 플래그 부여

각 레벨의 step 중 하나라도 `type: approval`이면, 그 step만 `solo_execution: true`로 마킹한다. 레벨 자체는 그대로 유지 — Executor가 solo 플래그를 보고 순차 실행한다.

### Step 6 — 메타데이터 출력

각 step에 대해 `step_metadata`에 기록:

```yaml
step_metadata:
  <id>:
    level: <int>
    type: command | ai | approval
    upstream: [<direct depends_on>]
    upstream_closure: [<transitive closure>]
    output_refs: [<ids referenced via ${steps...}>]
    solo_execution: bool
```

레벨 리스트는 `levels[]`에 rationale과 함께:

```yaml
levels:
  - level: 0
    steps: [a, b]
    has_approval: false
    rationale: "no dependencies"
```

## 예시

**입력 (Parser 출력 요약):**
```yaml
steps:
  - id: a, depends_on: []
  - id: b, depends_on: []
  - id: c, depends_on: [a]
  - id: d, depends_on: [a, b]
  - id: summary, type: approval, depends_on: [c, d]
```

**출력:**
```yaml
levels:
  - level: 0, steps: [a, b], has_approval: false
  - level: 1, steps: [c, d], has_approval: false
  - level: 2, steps: [summary], has_approval: true
step_metadata:
  a: { level: 0, upstream: [], upstream_closure: [] }
  b: { level: 0, upstream: [], upstream_closure: [] }
  c: { level: 1, upstream: [a], upstream_closure: [a] }
  d: { level: 1, upstream: [a,b], upstream_closure: [a,b] }
  summary: { level: 2, upstream: [c,d], upstream_closure: [a,b,c,d], solo_execution: true }
```

## 테스트 체크리스트

| 시나리오 | 기대 |
|---------|------|
| 선형 체인 (a→b→c) | 3개 레벨 |
| 팬아웃 (a→b, a→c) | 2개 레벨, b/c 병렬 |
| 다이아몬드 (a→b, a→c, b→d, c→d) | 3개 레벨 |
| 순환 (a→b→a) | cycles=[[a,b]], errors |
| `${steps.x.output}` 참조가 depends_on에 없음 | reference error |
| `${steps.x.output}` 참조가 전이적으로만 있음 | 통과 |
| approval step이 다른 step과 같은 레벨 | solo_execution=true만 부여, 레벨 유지 |

## 실패 시 행동

- 순환: 즉시 에러, Executor로 진행 금지
- 참조 위반: 즉시 에러, fix_hint에 "depends_on에 `<ref>`를 추가하세요" 포함
- 재계산 요청: 스킵 이벤트 수신 시 해당 id를 "완료" 처리 후 Kahn만 다시 수행 (결과는 원래와 동일해야 정상)
