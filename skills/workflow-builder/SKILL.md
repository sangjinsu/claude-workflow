---
name: workflow-builder
description: >
  워크플로우 YAML을 생성하고 실행하는 스킬.
  "워크플로우", "workflow", "배포 플로우", "작업 흐름" 언급 시 사용.
  /workflow-builder로 직접 호출 가능.
argument-hint: <run|create|record|list|show|validate> [name]
allowed-tools: [Read, Write, Bash, Glob, Grep]
---

# Workflow Builder

YAML 워크플로우를 정의하고 Claude Code 안에서 실행하는 스킬.
외부 의존성 없음. 스크립트 없음. Claude가 YAML을 직접 읽고 실행.

## 서브커맨드

`$ARGUMENTS`를 파싱하여 서브커맨드를 결정한다:
- `run <name> [--profile <name>] [--set KEY=VALUE ...]` → 워크플로우 실행
- `create [template]` → 새 워크플로우 생성
- `record` → 대화형 워크플로우 레코더
- `list` → 워크플로우 목록
- `show <name>` → 워크플로우 상세 보기
- `validate <name>` → 워크플로우 검증
- 인자 없음 → 사용 가능한 서브커맨드 안내

### 인자 검증

서브커맨드 파싱 후 반드시 검증한다:
- 서브커맨드가 `run`, `create`, `record`, `list`, `show`, `validate` 중 하나가 아니면:
  "알 수 없는 서브커맨드: [cmd]. 사용 가능: run, create, record, list, show, validate" 출력 후 종료.
- `<name>` 인자에 `/`, `\`, `..` 이 포함되어 있으면:
  "워크플로우 이름에 경로 구분자를 포함할 수 없습니다." 출력 후 종료.
- `<name>` 인자는 영문, 숫자, 하이픈, 언더스코어, 한글만 허용한다.

---

## run <name> [--profile <name>] [--set KEY=VALUE ...]

워크플로우를 실행한다. 이 섹션의 지시를 **기계적으로** 따를 것.

**옵션:**
- `--profile <name>`: `config.profiles.<name>`의 변수를 `config.base`에 병합 (override)
- `--set KEY=VALUE`: 개별 변수를 런타임에서 override (여러 개 가능)

**우선순위 (높은 것이 우선):**
1. `--set KEY=VALUE` (CLI override)
2. `--profile <name>` (선택된 프로파일)
3. `config.base` (기본값)

예: `run deploy --profile prod --set REPLICAS=10`

### 1단계: 파일 찾기

다음 순서로 워크플로우 파일을 찾는다:
1. `.workflows/<name>.yaml`
2. `.workflows/<name>.yml`
3. `~/.workflows/<name>.yaml`
4. `~/.workflows/<name>.yml`

찾지 못하면 다음을 출력 후 종료:
```
워크플로우 '<name>'을 찾을 수 없습니다.
검색한 경로:
  1. .workflows/<name>.yaml
  2. .workflows/<name>.yml
  3. ~/.workflows/<name>.yaml
  4. ~/.workflows/<name>.yml
`/workflow-builder list`로 사용 가능한 워크플로우를 확인하세요.
```

### 2단계: YAML 읽기 및 파싱

Read 도구로 파일을 읽는다. YAML 문법이 올바르지 않으면(들여쓰기 오류, 잘못된 구조 등):
"워크플로우 파일의 YAML 형식이 올바르지 않습니다. `/workflow-builder validate <name>`으로 상세 오류를 확인하세요." 출력 후 종료.

YAML 내용에서 다음을 추출:

- `workflow.name` — 워크플로우 이름 (필수)
- `config.base` — 기본 변수 맵 (선택)
- `config.profiles` — 환경별 변수 맵 (선택, dev/staging/prod 등)
- `steps[]` — 실행할 단계 목록 (필수, 1개 이상)

각 step에서 추출할 필드:
- `id` (필수) — 고유 식별자
- `name` (선택, 없으면 id 사용) — 표시 이름
- `type` (필수) — `command`, `ai`, 또는 `approval`
- `commands[]` (type: command일 때 필수) — 실행할 쉘 명령 목록
- `prompt` (type: ai일 때 필수) — Claude가 처리할 프롬프트
- `message` (type: approval일 때 필수) — 사용자에게 표시할 확인 메시지
- `default` (type: approval일 때 선택, 기본값 `deny`) — `approve` 또는 `deny`
- `depends_on[]` (선택) — 선행 step id 목록
- `on_failure` (선택, 기본값 `abort`) — `abort` 또는 `continue`
- `description` (선택) — 단계 설명

### 3단계: 검증

다음을 확인한다:
- `workflow.name` 존재 여부
- `steps`가 비어있지 않은지
- 각 step에 `id`, `type`, `commands`가 있는지
- `type`이 `command`, `ai`, 또는 `approval`인지 (아니면 "지원하지 않는 타입입니다: [type]" 경고 후 건너뜀)
  - 건너뛴 step은 의존성 해결 시 "완료"로 간주한다. 해당 step에 의존하는 다른 step은 정상적으로 실행된다.
- type: command인 step에 `commands`가 있는지
- type: ai인 step에 `prompt`가 있는지
- type: approval인 step에 `message`가 있는지
- step id 중복 없는지
- `depends_on`의 모든 참조가 존재하는 step id인지

검증 실패 시 구체적 에러 메시지를 출력하고 중단한다.

### 4단계: 실행 순서 결정 (병렬 레벨 계산)

`depends_on`을 분석하여 실행 순서를 결정한다. 결과는 **병렬 레벨**의 리스트다.

**병렬 레벨 계산 알고리즘:**
1. 모든 step을 "미할당" 상태로 시작
2. 레벨 0: 모든 의존성이 없는 step (또는 모든 의존성이 이미 할당된 step) — 같은 레벨
3. 레벨 N+1: 모든 의존성이 레벨 N 이하에서 할당된 step
4. 모든 step이 할당될 때까지 반복

**규칙:**
- 같은 레벨의 step은 YAML에 정의된 순서 유지 (순서는 의미만 가짐, 실행은 동시)
- 같은 레벨의 step은 **병렬로 실행** 가능 (6단계 참조)
- **순환 참조 감지**: 알고리즘이 진행되지 않는 step이 남아있으면 (즉, 모든 미할당 step의 의존성 중 하나라도 미할당이면) 순환이다. "순환 참조 감지: 다음 step들이 순환에 포함됩니다: [list]. 워크플로우를 실행할 수 없습니다." 출력 후 중단

**예시:**
```
steps: [a, b, c, d]
a: depends_on []
b: depends_on []
c: depends_on [a]
d: depends_on [a, b]

→ 레벨 0: [a, b]  (병렬)
→ 레벨 1: [c, d]  (병렬, 모두 a/b 완료 후)
```

### 5단계: 변수 치환

**변수 병합 순서 (낮은 우선순위 → 높은 우선순위):**

1. `config.base` 값으로 시작
2. `--profile <name>` 옵션이 지정되었으면:
   - `config.profiles.<name>`이 존재하는지 확인. 없으면 "프로파일 '<name>'이 정의되지 않았습니다. 사용 가능: [list]" 출력 후 종료.
   - `config.profiles.<name>`의 값을 base에 병합 (덮어쓰기)
3. `--set KEY=VALUE` 옵션 값을 병합 (덮어쓰기)

이 결과로 최종 변수 맵을 만든 후 치환을 시작한다.

각 step의 `commands[]` 문자열에서 `${VAR_NAME}` 패턴을 찾아 `config.base`의 값으로 치환한다.

**치환 규칙:**
- 치환 대상: `config.base`에 키가 존재하는 `${KEY}` 패턴만
- 치환 금지: `$(...)` (shell subcommand), `` `...` `` (backtick), `$VAR` (중괄호 없는 형태)
- 정의되지 않은 `${VAR_NAME}` 발견 시: "경고: 변수 '${VAR_NAME}'이 config.base에 정의되지 않았습니다" 출력. 원본 유지.

**보안 경고:**
- `config.base` 값에 쉘 메타문자(`;`, `|`, `&&`, `||`, `` ` ``)가 포함되어 있으면:
  "경고: 변수 '${KEY}'의 값에 쉘 메타문자가 포함되어 있습니다. 의도적인지 확인하세요." 출력.

### 5.5단계: 실행 전 확인

변수 치환이 완료된 후, 실행 전에 모든 step의 commands를 사용자에게 표시한다:

```
실행할 워크플로우: [workflow.name]
사용된 파일: [resolved_file_path]

Step 1: [step.name] (id: [step.id])
  $ [command1 after substitution]
  $ [command2 after substitution]

Step 2: [step.name] (id: [step.id])
  $ [command1 after substitution]
  ...

계속 실행하시겠습니까?
```

사용자가 확인하면 6단계로 진행한다. 거부하면 "워크플로우 실행이 취소되었습니다." 출력 후 종료.

### 6단계: 실행 (병렬 레벨 루프)

4단계에서 계산된 병렬 레벨 순서대로 실행한다.

**각 레벨 처리:**
1. 같은 레벨에 step이 1개면: 단일 실행
2. 같은 레벨에 step이 2개 이상이면: **병렬 실행**
   - command step의 commands를 같은 메시지에서 여러 Bash tool call로 동시 호출
   - ai step은 단독 처리 (병렬 호출 불가)
   - approval step은 단독 처리 (병렬 호출 불가, 동시 승인 요청은 UX 붕괴)
   - 모든 step이 완료된 후 다음 레벨로 진행

레벨이 끝나기 전에 다음 레벨을 시작하지 않는다.

**병렬 실행 안내:**
```
⚡ 레벨 [N] 병렬 실행: [step.id1], [step.id2], [step.id3]
```

각 command 실행 시 Bash 도구의 timeout을 300000 (5분)으로 설정한다.

각 step에 대해:

**type: command인 경우:**
```
a) 출력: "▶ Step [N]/[total]: [step.name] (id: [step.id])"
b) description이 있으면 출력: "  [description]"
c) step.commands 배열의 각 command를 순서대로 Bash로 실행
d) 각 command 실행 직후 exit code 확인:
   - 성공 (exit 0): 다음 command로 진행
   - 실패 (exit != 0):
     - on_failure == "abort": "Step [id] 실패 (exit code: [code]). 실패한 명령: [command]. 워크플로우를 중단합니다." 출력 후 전체 중단
     - on_failure == "continue": "Step [id] 실패했지만 continue 설정으로 다음 step으로 진행합니다." 출력
e) 모든 command 성공 시: "  Step [id] 완료"
```

**type: ai인 경우:**
```
a) 출력: "🤖 Step [N]/[total]: [step.name] (id: [step.id], type: ai)"
b) description이 있으면 출력: "  [description]"
c) step.prompt의 ${VAR} 변수를 치환한다 (5단계와 동일 규칙)
d) 치환된 prompt를 처리하고 결과를 출력한다
e) "  Step [id] 완료 (AI)"
```

AI step의 prompt는 이전 step의 실행 컨텍스트(출력 결과)를 참조할 수 있다.
예: "위 로그를 분석하고 에러가 있으면 요약하세요"

**type: approval인 경우:**
```
a) 출력: "🛑 Step [N]/[total]: [step.name] (id: [step.id], type: approval)"
b) description이 있으면 출력: "  [description]"
c) step.message의 ${VAR} 변수를 치환한다 (5단계와 동일 규칙)
d) AskUserQuestion 도구로 사용자에게 승인 요청:
   - question: 치환된 message
   - options:
     - "승인 (계속 진행)" (default가 approve이면 첫 번째)
     - "거부 (워크플로우 중단)" (default가 deny이면 첫 번째, 기본값)
e) 사용자 응답에 따라:
   - 승인: "  Step [id] 승인됨" 출력 후 다음 step 진행
   - 거부:
     - on_failure == "abort": "Step [id] 거부됨. 워크플로우를 중단합니다." 출력 후 전체 중단
     - on_failure == "continue": "Step [id] 거부됨. continue 설정으로 다음 step으로 진행합니다." 출력
```

**병렬 실행 제약**: type: approval step은 단독 처리한다. 같은 레벨에 다른 step이 있어도 approval은 별도로 실행. 동시에 여러 approval이 뜨면 UX가 깨진다.

type이 command/ai/approval가 아닌 step은:
```
⏭ Step [N]/[total]: [step.name] (건너뜀 - 미지원 타입: [type])
```

모든 step 완료 후: "워크플로우 '[workflow.name]' 실행 완료 ([N]/[total] steps 성공, [M] 건너뜀)"

### 핵심 규칙

**반드시 지킬 것:**
- commands에 적힌 명령을 **정확히 그대로** 실행할 것. 수정, 추가, 생략 금지.
- exit code를 **반드시** 확인할 것. 성공을 가정하지 말 것.
- step 순서를 **절대** 바꾸지 말 것. 4단계에서 결정된 순서를 따를 것.
- 실행 중 "도움"을 주려 하지 말 것. 에러가 나면 사실만 보고할 것.
- 대화형 프롬프트가 나오면 사용자에게 알리고 진행할 것.

---

## create [template]

새 워크플로우를 생성한다.

### 절차:

1. 사용자에게 워크플로우 이름을 물어본다
2. 저장 위치를 물어본다:
   - `.workflows/` (현재 프로젝트)
   - `~/.workflows/` (글로벌)
3. 사용 가능한 템플릿을 보여준다:
   - `deploy` — 배포 사전 점검 (kubectl + curl)
   - `blank` — 최소 구조 템플릿

4. 인자로 template이 지정되었으면 해당 템플릿 사용, 아니면 선택하게 한다
5. 프로젝트 루트의 `assets/templates/` 디렉토리에서 템플릿을 찾아 선택한 위치에 복사한다. 템플릿 검색 순서:
   1. 현재 작업 디렉토리의 `assets/templates/<template>.yaml`
   2. 플러그인 소스의 `assets/templates/<template>.yaml`

```bash
mkdir -p <destination_dir>
cp "<found_template_path>" "<destination_dir>/<workflow_name>.yaml"
```

6. 안내: "워크플로우가 생성되었습니다. 파일을 열어 config.base 변수와 commands를 수정하세요."

---

## record

대화형으로 워크플로우를 생성한다. YAML을 직접 작성하는 대신, 자연어로 설명하면 워크플로우를 생성한다.

### 절차:

1. 사용자에게 워크플로우의 목적을 물어본다:
   "어떤 작업을 워크플로우로 만들고 싶으신가요? (예: 배포 전 점검, DB 백업, 테스트 실행)"

2. 사용자의 설명을 바탕으로 다음을 결정한다:
   - 워크플로우 이름
   - 필요한 변수 (config.base)
   - step 목록 (각 step의 id, name, type, commands 또는 prompt)
   - step 간 의존 관계 (depends_on)
   - 실패 처리 전략 (on_failure)

3. 생성할 워크플로우를 미리보기로 보여준다:
   ```
   생성할 워크플로우:
   ---
   [YAML 내용]
   ---
   ```

4. 사용자에게 확인을 받는다:
   - 수정이 필요하면 사용자 의견을 반영하여 다시 생성한다
   - 확인되면 저장 위치를 물어본다 (`.workflows/` 또는 `~/.workflows/`)

5. Write 도구로 파일을 생성한다

6. 안내: "워크플로우 '[name]'이 [path]에 생성되었습니다. `/workflow-builder run [name]`으로 실행하세요."

### type: ai step 활용 안내

사용자의 설명에 분석, 판단, 요약 등의 키워드가 포함되면 `type: ai` step을 제안한다.
- "로그를 분석해서" → `type: ai` + `prompt: "위 로그를 분석하고..."`
- "결과를 판단해서" → `type: ai` + `prompt: "위 결과를 바탕으로 판단..."`
- "요약해줘" → `type: ai` + `prompt: "위 출력을 요약..."`

---

## list

등록된 워크플로우 목록을 보여준다.

### 절차:

1. `.workflows/*.yaml`과 `~/.workflows/*.yaml` 파일을 검색한다
2. 각 파일을 Read로 읽어 `workflow.name`과 step 수를 추출한다
3. 테이블로 출력한다:

```
| 파일명 | 이름 | Steps | 위치 |
|--------|------|-------|------|
```

파일이 없으면: "워크플로우가 없습니다. `/workflow-builder create`로 새로 만드세요."

---

## show <name>

워크플로우의 상세 내용을 보여준다.

### 절차:

1. `run`의 1단계와 같은 순서로 파일을 찾는다
2. Read로 파일을 읽는다
3. 다음을 정리하여 출력한다:
   - 워크플로우 이름, 버전, 설명
   - config.base 변수 목록
   - config.profiles 목록 (정의된 프로파일 이름과 각 프로파일이 override하는 변수)
   - step 목록 (id, name, type, depends_on, on_failure)
   - 실행 순서 (depends_on 기반)

---

## validate <name> [--repeat N]

워크플로우의 유효성을 검증한다. `--repeat N`이 지정되면 결정성 검증도 수행한다.

### 절차:

1. `run`의 1단계와 같은 순서로 파일을 찾는다
2. Read로 파일을 읽는다
3. `run`의 3단계 검증을 수행한다
4. 추가 검증:
   - commands에서 참조하는 `${VAR}` 변수가 config.base 또는 모든 profiles에 정의되어 있는지
   - depends_on 순환 참조 여부
   - 4단계 병렬 레벨 계산 시도 (순환 감지)
5. 결과 출력:
   - 성공: "워크플로우 '<name>' 검증 통과"
   - 실패: 각 오류를 번호 매겨 출력

### 결정성 검증 (`--repeat N`):

`--repeat 3` 같은 옵션이 있으면:

1. 위의 기본 검증을 먼저 수행
2. 변수 치환을 N번 반복 실행
3. N번의 결과가 모두 동일한지 비교
4. 각 step의 commands(치환 후)와 실행 순서가 N번 모두 일치하는지 확인
5. 결과 출력:
   - 일치: "결정성 검증 통과 (N회 동일 결과)"
   - 불일치: "비결정성 감지: [차이점 설명]" 출력. 어떤 step의 어떤 부분이 달라졌는지 표시.

이 검증은 prompt-only 아키텍처가 동일 입력에 대해 동일 출력을 생성하는지 확인한다.

---

## 워크플로우 YAML 스키마 (MVP)

```yaml
workflow:
  name: "워크플로우 이름"      # 필수
  description: "설명"          # 선택
  version: "1.0.0"            # 선택
  tags: [tag1, tag2]          # 선택

config:
  base:                        # 선택
    VAR_NAME: "value"          # ${VAR_NAME}으로 commands에서 참조

  profiles:                    # 선택, 환경별 변수 세트
    dev:                       # --profile dev로 활성화
      VAR_NAME: "dev-value"    # base 값을 override
    prod:                      # --profile prod로 활성화
      VAR_NAME: "prod-value"
      EXTRA_VAR: "only-in-prod"

steps:                         # 필수, 1개 이상
  - id: step-id               # 필수, 고유
    name: "표시 이름"          # 선택 (없으면 id 사용)
    type: command              # 필수 (command | ai | approval)
    description: "설명"        # 선택
    commands:                  # type: command일 때 필수
      - "shell command ${VAR_NAME}"
    prompt: "AI에게 요청할 내용"  # type: ai일 때 필수
    message: "사용자 확인 메시지"  # type: approval일 때 필수
    default: deny              # type: approval일 때 선택 (approve | deny, 기본: deny)
    depends_on: [other-id]     # 선택
    on_failure: abort          # 선택 (abort | continue, 기본: abort)
```

---

## 변수 Override 와 Profiles

### Profiles (환경별 변수 세트)

YAML에서 `config.profiles`로 환경별 변수를 미리 정의한다:

```yaml
config:
  base:
    NAMESPACE: "default"
    REPLICAS: "1"
  profiles:
    dev:
      NAMESPACE: "dev"
    prod:
      NAMESPACE: "production"
      REPLICAS: "5"
```

`--profile` 옵션으로 활성화:
```
/workflow-builder run deploy --profile prod
```

### CLI Override

`--set KEY=VALUE`로 실행 시 변수를 직접 override:
```
/workflow-builder run deploy --set NAMESPACE=production
```

### 우선순위

`--set` > `--profile` > `config.base`

조합 사용:
```
/workflow-builder run deploy --profile prod --set REPLICAS=10
```
이 경우 `prod` 프로파일의 NAMESPACE는 적용되고, REPLICAS만 10으로 override.

---

## Trust Model

워크플로우의 commands는 사용자의 권한으로 실행됩니다.
- 실행 전 변수 치환된 commands 목록이 표시되며, 사용자 확인 후 실행됩니다
- Git으로 공유받은 워크플로우는 실행 전 commands를 반드시 확인하세요
- 신뢰할 수 없는 출처의 YAML 파일은 실행하지 마세요
- 워크플로우 이름에 경로 구분자(`/`, `\`, `..`)를 사용할 수 없습니다
