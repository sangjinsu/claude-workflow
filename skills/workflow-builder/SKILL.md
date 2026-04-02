---
name: workflow-builder
description: >
  워크플로우 YAML을 생성하고 실행하는 스킬.
  "워크플로우", "workflow", "배포 플로우", "작업 흐름" 언급 시 사용.
  /workflow-builder로 직접 호출 가능.
allowed-tools:
  - Bash(cat:*)
  - Bash(mkdir:*)
  - Bash(ls:*)
  - Bash(cp:*)
  - Read
  - Write
---

# Workflow Builder

YAML 워크플로우를 정의하고 Claude Code 안에서 실행하는 스킬.
외부 의존성 없음. 스크립트 없음. Claude가 YAML을 직접 읽고 실행.

## 서브커맨드

`$ARGUMENTS`를 파싱하여 서브커맨드를 결정한다:
- `run <name>` → 워크플로우 실행
- `create [template]` → 새 워크플로우 생성
- `list` → 워크플로우 목록
- `show <name>` → 워크플로우 상세 보기
- `validate <name>` → 워크플로우 검증
- 인자 없음 → 사용 가능한 서브커맨드 안내

---

## run <name>

워크플로우를 실행한다. 이 섹션의 지시를 **기계적으로** 따를 것.

### 1단계: 파일 찾기

다음 순서로 워크플로우 파일을 찾는다:
1. `.workflows/<name>.yaml`
2. `.workflows/<name>.yml`
3. `~/.workflows/<name>.yaml`
4. `~/.workflows/<name>.yml`

찾지 못하면: "워크플로우 '<name>'을 찾을 수 없습니다. `.workflows/` 디렉토리를 확인하세요." 출력 후 종료.

### 2단계: YAML 읽기 및 파싱

Read 도구로 파일을 읽는다. YAML 내용에서 다음을 추출:

- `workflow.name` — 워크플로우 이름 (필수)
- `config.base` — 변수 맵 (선택)
- `steps[]` — 실행할 단계 목록 (필수, 1개 이상)

각 step에서 추출할 필드:
- `id` (필수) — 고유 식별자
- `name` (선택, 없으면 id 사용) — 표시 이름
- `type` (필수) — MVP에서는 `command`만 지원
- `commands[]` (필수) — 실행할 쉘 명령 목록
- `depends_on[]` (선택) — 선행 step id 목록
- `on_failure` (선택, 기본값 `abort`) — `abort` 또는 `continue`
- `description` (선택) — 단계 설명

### 3단계: 검증

다음을 확인한다:
- `workflow.name` 존재 여부
- `steps`가 비어있지 않은지
- 각 step에 `id`, `type`, `commands`가 있는지
- `type`이 `command`인지 (아니면 "MVP에서는 command 타입만 지원합니다" 경고 후 건너뜀)
- step id 중복 없는지
- `depends_on`의 모든 참조가 존재하는 step id인지

검증 실패 시 구체적 에러 메시지를 출력하고 중단한다.

### 4단계: 실행 순서 결정

`depends_on`을 분석하여 실행 순서를 결정한다:
- depends_on이 없는 step → 먼저 실행
- depends_on이 있는 step → 선행 step 이후에 실행
- 같은 레벨의 step은 YAML에 정의된 순서 유지
- **순환 참조 감지**: A→B→A 같은 순환이 있으면 에러 출력 후 중단

### 5단계: 변수 치환

각 step의 `commands[]` 문자열에서 `${VAR_NAME}` 패턴을 찾아 `config.base`의 값으로 치환한다.
- 정의되지 않은 변수 발견 시: "경고: 변수 '${VAR_NAME}'이 config.base에 정의되지 않았습니다" 출력. 원본 유지.

### 6단계: 실행 (기계적 루프)

정렬된 순서대로 각 step을 실행한다. **이 루프를 정확히 따를 것.**

각 step에 대해:

```
a) 출력: "▶ Step [N]/[total]: [step.name] (id: [step.id])"
b) description이 있으면 출력: "  [description]"
c) step.commands 배열의 각 command를 순서대로 Bash로 실행
d) 각 command 실행 직후 exit code 확인:
   - 성공 (exit 0): 다음 command로 진행
   - 실패 (exit != 0):
     - on_failure == "abort": "Step [id] 실패. 워크플로우를 중단합니다." 출력 후 전체 중단
     - on_failure == "continue": "Step [id] 실패했지만 continue 설정으로 다음 step으로 진행합니다." 출력
e) 모든 command 성공 시: "  Step [id] 완료"
```

모든 step 완료 후: "워크플로우 '[workflow.name]' 실행 완료 ([N]/[total] steps 성공)"

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

!`ls ${CLAUDE_SKILL_DIR}/../../assets/templates/`

4. 인자로 template이 지정되었으면 해당 템플릿 사용, 아니면 선택하게 한다
5. 템플릿을 선택한 위치에 복사한다:

```bash
mkdir -p <destination_dir>
cp "${CLAUDE_SKILL_DIR}/../../assets/templates/<template>.yaml" "<destination_dir>/<workflow_name>.yaml"
```

6. 안내: "워크플로우가 생성되었습니다. 파일을 열어 config.base 변수와 commands를 수정하세요."

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
   - step 목록 (id, name, type, depends_on, on_failure)
   - 실행 순서 (depends_on 기반)

---

## validate <name>

워크플로우의 유효성을 검증한다.

### 절차:

1. `run`의 1단계와 같은 순서로 파일을 찾는다
2. Read로 파일을 읽는다
3. `run`의 3단계 검증을 수행한다
4. 추가 검증:
   - commands에서 참조하는 `${VAR}` 변수가 config.base에 모두 정의되어 있는지
   - depends_on 순환 참조 여부
5. 결과 출력:
   - 성공: "워크플로우 '<name>' 검증 통과"
   - 실패: 각 오류를 번호 매겨 출력

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

steps:                         # 필수, 1개 이상
  - id: step-id               # 필수, 고유
    name: "표시 이름"          # 선택 (없으면 id 사용)
    type: command              # 필수, MVP에서는 command만
    description: "설명"        # 선택
    commands:                  # 필수, 1개 이상
      - "shell command ${VAR_NAME}"
    depends_on: [other-id]     # 선택
    on_failure: abort          # 선택 (abort | continue, 기본: abort)
```

---

## Trust Model

워크플로우의 commands는 사용자의 권한으로 실행됩니다.
- MVP에서는 워크플로우 작성자 = 실행자 (self-trust)
- Git으로 공유받은 워크플로우는 실행 전 commands를 반드시 확인하세요
- 신뢰할 수 없는 출처의 YAML 파일은 실행하지 마세요
