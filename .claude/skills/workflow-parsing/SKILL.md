---
name: workflow-parsing
description: >
  YAML 워크플로우 파일을 찾고, 파싱하고, config.base/profiles/--set의 우선순위대로
  변수를 병합하고, 스키마를 검증하는 스킬. workflow-parser 에이전트가 사용. 파이프라인
  실행 전처리, "워크플로우 로드", "변수 병합", "스키마 검증", "YAML 파싱" 언급 시
  반드시 이 스킬로 처리. 단순 파일 읽기가 아니라 워크플로우 구조를 이해해야 하는
  모든 작업에 우선 트리거.
---

# Workflow Parsing Skill

워크플로우 YAML을 안전하고 결정적으로 로드하기 위한 절차. 이 스킬은 `workflow-parser` 에이전트가 사용하며, 파이프라인의 첫 단계다.

## 언제 이 스킬을 사용하나

- 사용자가 `/workflow-builder run <name>`을 요청했을 때 첫 번째로
- 파이프라인 오케스트레이터가 파싱 단계를 위임했을 때
- `validate` 서브커맨드의 내부 단계로 (실행은 생략하더라도 파싱은 공유)
- 워크플로우 파일의 구조적 문제를 조사할 때

## 기본 원칙 (Why)

**1. 기계적 준수**: 사용자의 commands는 사용자의 계약이다. Parser가 "더 나은 형태"로 고치려 하면 워크플로우의 결정성이 깨진다. 치환 규칙도 문서화된 패턴 외에는 건드리지 않는다.

**2. 단계 분리**: 파싱, 변수 병합, 스키마 검증, 치환은 **논리적으로 분리된** 단계다. 한 단계의 실패가 다른 단계의 에러 메시지를 오염시키면 디버깅이 어렵다.

**3. 보안 우선**: 변수 값에 쉘 메타문자가 있다는 것은 버그이거나 의도적 코드 주입이다. 둘 다 사용자 경고가 가치 있다.

## 절차

### Step 1 — 파일 탐색

순서대로 시도하고 **처음 발견되는** 파일을 사용:

1. `.workflows/<name>.yaml`
2. `.workflows/<name>.yml`
3. `~/.workflows/<name>.yaml`
4. `~/.workflows/<name>.yml`

파일명 검증 (탐색 전):
- `/`, `\`, `..` 포함 시 **즉시 거부**
- 영문/숫자/하이픈/언더스코어/한글 외의 문자 포함 시 거부

**발견 실패 시 출력 템플릿:**
```
워크플로우 '<name>'을 찾을 수 없습니다.
검색한 경로:
  1. .workflows/<name>.yaml
  2. .workflows/<name>.yml
  3. ~/.workflows/<name>.yaml
  4. ~/.workflows/<name>.yml
```

### Step 2 — YAML 읽기

Read 도구로 파일 전체를 읽는다. 파싱 실패 시 YAML 에러 메시지를 **그대로** 인용한다. "아마 이런 의도였을 것" 같은 추측 금지.

### Step 3 — 필드 추출

| 키 | 필수 | 비고 |
|----|------|------|
| `workflow.name` | Y | 비어있으면 거부 |
| `workflow.description` | N | |
| `workflow.version` | N | |
| `config.base` | N | 키-값 맵 |
| `config.profiles` | N | `{profile: {key: value}}` |
| `steps[]` | Y | 최소 1개 |

각 step:

| 키 | 필수 | 조건 |
|----|------|------|
| `id` | Y | 전체 step에서 고유 |
| `name` | N | 없으면 `id` 사용 |
| `type` | Y | `command`, `ai`, `approval` (다른 값은 건너뛰기 후보) |
| `commands[]` | type=command | 비어있으면 거부 |
| `prompt` | type=ai | |
| `message` | type=approval | |
| `default` | type=approval, 선택 | `approve` 또는 `deny`, 기본 `deny` |
| `depends_on[]` | N | 모든 참조가 유효한 id여야 함 |
| `on_failure` | N | `abort` 또는 `continue`, 기본 `abort` |
| `description` | N | |

### Step 4 — 변수 병합

우선순위(낮음 → 높음):

```
config.base  <  config.profiles[profile]  <  --set KEY=VALUE
```

**병합 절차:**
1. `merged = dict(config.base or {})`
2. profile이 지정되었으면:
   - `config.profiles[profile]`이 없으면 에러 (사용 가능한 프로파일 목록 출력)
   - `merged.update(config.profiles[profile])`
3. CLI override가 있으면:
   - 각 `KEY=VALUE`를 `merged[KEY] = VALUE`

**결과는 `merged_variables`로 출력 프로토콜에 포함.** 이 값이 이후 치환 단계의 유일한 진실 원천이다.

### Step 5 — 스키마 검증

- 모든 `depends_on` 참조가 존재하는 step id인지
- step id 중복 없음
- type=command인데 commands 없음 → 거부
- type=ai인데 prompt 없음 → 거부
- type=approval인데 message 없음 → 거부
- 지원하지 않는 type → 거부 아닌 **경고**, 그리고 step을 "skipped" 후보로 마킹 (의존성 해결 시 "완료"로 간주)

### Step 6 — 변수 치환 규칙 (원칙만)

**치환 대상:**
- `${KEY}` — `merged_variables`에 있는 키
- `${steps.<id>.output}` — 이 스킬에서는 치환 **안 함**. Planner의 참조 검증을 거친 후 Executor가 실행 직전에 치환한다

**치환 금지 (절대):**
- `$(command)` — shell subcommand
- `` `command` `` — backtick
- `$VAR` — 중괄호 없는 형태

**정의되지 않은 `${VAR}`:**
- 경고 출력: `경고: 변수 '${VAR}'이 config.base에 정의되지 않았습니다`
- 원본 유지 (치환하지 않음)

**쉘 메타문자 경고:**
- 변수 값에 `;`, `|`, `&&`, `||`, `` ` `` 가 포함되면:
  ```
  경고: 변수 '${KEY}'의 값에 쉘 메타문자가 포함되어 있습니다. 의도적인지 확인하세요.
  ```

### Step 7 — 출력 작성

`_workspace/01_parser_output.yaml`에 쓰기. 존재하지 않으면 디렉토리 생성. 스키마는 에이전트 정의의 "출력 프로토콜" 참조.

## 테스트 체크리스트

| 시나리오 | 기대 |
|---------|------|
| `hello.yaml` 파싱 | status=ok, warnings 없음 |
| 존재하지 않는 name | status=error, 4개 경로 나열 |
| 순환 depends_on | Planner에서 탐지되므로 여기서는 통과 |
| profile 미정의 | status=error, 사용 가능 profile 나열 |
| `type: prompt` (미지원) | warning, step을 skipped 후보로 마킹 |
| `${UNDEFINED_VAR}` 포함 | warning, 원본 유지 |
| `${KEY}` 값에 `;` 포함 | shell_metachar warning |
| `../../etc/passwd` 이름 | 즉시 거부 |

## 실패 시 행동

- 치명적 에러: `status: error`, `errors[]`에 이유 상세화, 즉시 반환 (재시도 불필요 — 결정적)
- 경고: `warnings[]`에 기록하고 진행
- 재시도 금지: 파일이 바뀌지 않는 한 결과는 동일
