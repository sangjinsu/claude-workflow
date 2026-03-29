# Claude Code 워크플로우 플러그인 개발 계획 v2

> **프로젝트명**: workflow-builder  
> **목적**: Claude Code에서 `/workflow-builder`로 호출하여 재사용 가능한 워크플로우 YAML 파일을 생성·관리하는 스킬  
> **작성일**: 2026-03-28  
> **개정**: v2 — 스킬 체이닝, API 호출, 환경 프로파일, 스텝 타입 확장

---

## 1. 개요

개발 업무 중 반복되는 작업 흐름(배포, 인시던트 대응, 인프라 축소 등)을 YAML 워크플로우 파일로 정의하고, Claude Code 내에서 슬래시 커맨드(`/workflow-builder`)로 생성·실행·관리할 수 있는 스킬을 만든다.

### 핵심 기능

- 워크플로우 YAML 템플릿 생성 (대화형으로 경로 지정)
- 기존 워크플로우 조회·수정·삭제
- 워크플로우 단계별 실행 가이드
- 프로젝트별 / 글로벌 워크플로우 관리
- **[v2] 다른 Claude Code 스킬을 step에서 호출**
- **[v2] REST API 호출 step 지원 (인증·헤더·바디 설정)**
- **[v2] 환경 프로파일 (dev/staging/prod) 분리**
- **[v2] 승인 게이트, 조건 분기, 병렬 실행**

---

## 2. v1 설계의 보완점 분석

### 2.1 구조적 한계

| 문제                            | 설명                                                                      | 개선 방향                                                             |
| ------------------------------- | ------------------------------------------------------------------------- | --------------------------------------------------------------------- |
| **step 타입이 `commands`뿐**    | 쉘 커맨드만 지원하므로 API 호출, 스킬 실행, 수동 승인 등을 표현할 수 없음 | `type` 필드 도입 → `command`, `skill`, `api`, `approval`, `condition` |
| **환경 설정이 단순 key-value**  | dev/staging/prod 환경별 분리 불가, 시크릿 관리 미흡                       | `config` 섹션 + 프로파일 시스템 + `.env` 파일 참조                    |
| **스텝 간 데이터 전달 없음**    | 이전 스텝 결과를 다음 스텝에서 사용 불가                                  | `outputs` / `inputs` + `${steps.STEP_ID.output}` 참조                 |
| **순차 실행만 가능**            | `depends_on`이 있지만 병렬 실행 개념 없음                                 | `parallel` 그룹 + DAG 기반 실행 순서                                  |
| **수동 승인 게이트 없음**       | 프로덕션 배포 같은 위험 작업에 사람 확인 필요                             | `type: approval` 스텝 도입                                            |
| **재시도 로직 단순**            | `on_failure` 4가지만 존재                                                 | `retry` 블록 (횟수, 간격, 백오프)                                     |
| **dry-run 모드 없음**           | 실행 전 전체 흐름 미리보기 불가                                           | `run --dry-run` 서브커맨드                                            |
| **Claude Code hooks 연동 없음** | PreToolUse/PostToolUse 훅과 분리됨                                        | `hooks` 섹션으로 연결                                                 |
| **MCP 서버 참조 불가**          | 외부 도구(Slack, Notion 등) 호출을 위한 MCP 연동 없음                     | `type: mcp` 스텝 또는 `mcp_servers` 설정                              |
| **서브 워크플로우 없음**        | 공통 로직 재사용 불가                                                     | `type: workflow` (다른 워크플로우 import)                             |

### 2.2 실용성 개선

| 문제                      | 설명                                                               | 개선 방향                                                    |
| ------------------------- | ------------------------------------------------------------------ | ------------------------------------------------------------ |
| **description 길이 제한** | Claude Code description 최대 1024자 → 현 설계의 description이 장황 | 핵심 트리거 키워드 중심으로 간결하게                         |
| **SKILL.md 500줄 권장**   | 본문이 길어지면 성능 저하                                          | 핵심 로직만 SKILL.md에, 상세 스키마는 `references/`로 분리   |
| **템플릿 확장성**         | 템플릿 4개 고정 → 사용자 정의 템플릿 추가 어려움                   | `templates/` 디렉토리 스캔 방식 + 커스텀 템플릿 등록         |
| **실행 이력 없음**        | 언제 어떤 워크플로우를 실행했는지 추적 불가                        | `.workflows/.runs/` 디렉토리에 실행 로그 저장                |
| **검증이 스키마만**       | YAML 구조만 확인, 실제 실행 가능성 미검증                          | 커맨드 존재 여부, API 엔드포인트 도달 가능 여부 등 사전 검증 |

---

## 3. 스킬 디렉토리 구조 (v2)

```
workflow-builder/
├── SKILL.md                         # 스킬 핵심 정의 (500줄 이내)
├── scripts/
│   ├── create_workflow.py           # YAML 생성
│   ├── validate_workflow.py         # 스키마 + 실행 가능성 검증
│   ├── list_workflows.py            # 목록 조회
│   ├── resolve_profile.py           # 프로파일 병합 (base + env overlay)
│   └── run_dry.py                   # dry-run 시뮬레이션
├── references/
│   ├── SCHEMA.md                    # 워크플로우 YAML 스키마 전체 명세
│   ├── STEP_TYPES.md                # step 타입별 상세 가이드
│   ├── CONFIG_GUIDE.md              # config/프로파일/시크릿 설정 가이드
│   └── EXAMPLES.md                  # 예제 워크플로우 모음
└── assets/
    └── templates/
        ├── deploy.yaml              # 배포 워크플로우
        ├── incident.yaml            # 인시던트 대응
        ├── infra-scale.yaml         # 인프라 축소/확장
        ├── api-health-check.yaml    # [v2] API 헬스체크
        ├── skill-chain.yaml         # [v2] 스킬 체이닝 예제
        └── blank.yaml               # 빈 템플릿
```

---

## 4. SKILL.md 설계 (v2)

### 4.1 YAML Frontmatter

```yaml
---
name: workflow-builder
description: >
  워크플로우 YAML을 생성·관리·실행하는 스킬. "워크플로우", "workflow",
  "배포 플로우", "작업 흐름", "YAML 워크플로우", "파이프라인 정의",
  "스킬 체이닝", "API 호출 플로우" 언급 시 사용.
  /workflow-builder로 직접 호출 가능.
disable-model-invocation: false
allowed-tools:
  - Bash(python3:*)
  - Bash(cat:*)
  - Bash(mkdir:*)
  - Bash(ls:*)
  - Bash(curl:*)
  - Read
  - Write
  - Skill
---
```

> **변경점**: `Bash(curl:*)` 추가 (API 호출 검증용), `Skill` 추가 (스킬 체이닝 시 다른 스킬 호출), description 간결화 (1024자 이내)

---

## 5. 워크플로우 YAML 스키마 (v2)

### 5.1 전체 구조

```yaml
workflow:
  name: "워크플로우 이름"
  description: "설명"
  version: "1.0.0"
  author: "jinsu"
  created_at: "2026-03-28"
  tags: [deploy, japan]

# ─── [v2] 환경 설정 ─────────────────────────────────
config:
  profiles:
    default: dev
    available: [dev, staging, prod]

  # 기본 환경 변수 (모든 프로파일 공통)
  base:
    SERVICE_NAME: "oshi-no-ko-puzzle"
    HEALTH_CHECK_PATH: "/api/health"

  # 프로파일별 오버라이드
  dev:
    REGION: "ap-northeast-1"
    CLUSTER: "dev-jp"
    API_BASE_URL: "https://dev-api.example.com"
    DB_INSTANCE: "dev-oshi-db"

  staging:
    REGION: "ap-northeast-1"
    CLUSTER: "staging-jp"
    API_BASE_URL: "https://staging-api.example.com"
    DB_INSTANCE: "staging-oshi-db"

  prod:
    REGION: "ap-northeast-1"
    CLUSTER: "prod-jp"
    API_BASE_URL: "https://api.example.com"
    DB_INSTANCE: "prod-oshi-db"

  # 시크릿 참조 (값은 YAML에 절대 작성하지 않음)
  secrets:
    - name: AWS_ACCESS_KEY_ID
      source: env # 시스템 환경변수에서 읽기
    - name: SLACK_WEBHOOK_URL
      source: env
    - name: DB_PASSWORD
      source: file # 파일에서 읽기
      path: ~/.secrets/db_password

  # [v2] API 공통 설정
  api_defaults:
    timeout: 30s
    retry:
      max_attempts: 3
      backoff: exponential
      initial_delay: 1s
    headers:
      Content-Type: "application/json"
      User-Agent: "workflow-builder/1.0"

  # [v2] MCP 서버 참조
  mcp_servers:
    - name: slack
      description: "Slack 알림 전송용"
    - name: notion
      description: "Notion 문서 업데이트용"

  # [v2] 스킬 의존성
  required_skills:
    - name: "deploy-helper"
      description: "배포 자동화 보조 스킬"
    - name: "db-migration"
      description: "DB 마이그레이션 스킬"

# ─── 스텝 정의 ──────────────────────────────────────
steps:
  # ── 타입 1: command (기존과 동일) ──
  - id: pre-check
    name: "사전 점검"
    type: command
    description: "배포 전 상태 확인"
    commands:
      - "kubectl get pods -n ${SERVICE_NAME} --context ${CLUSTER}"
      - "curl -sf ${API_BASE_URL}${HEALTH_CHECK_PATH}"
    on_failure: abort
    timeout: 60s
    outputs:
      pod_status: "kubectl 결과를 저장"

  # ── 타입 2: api (REST API 호출) ──
  - id: check-deploy-lock
    name: "배포 잠금 확인"
    type: api
    description: "배포 관리 API에서 현재 잠금 상태 확인"
    request:
      method: GET
      url: "${API_BASE_URL}/deploy/lock"
      headers:
        Authorization: "Bearer ${DEPLOY_API_TOKEN}"
      query_params:
        service: "${SERVICE_NAME}"
    expect:
      status: 200
      body_contains: '"locked": false'
    on_failure: abort
    depends_on: [pre-check]
    outputs:
      lock_status: "$.locked" # JSONPath로 응답에서 값 추출

  # ── 타입 3: skill (다른 Claude Code 스킬 호출) ──
  - id: run-migration
    name: "DB 마이그레이션"
    type: skill
    description: "db-migration 스킬을 호출하여 마이그레이션 실행"
    skill:
      name: "db-migration"
      arguments: "run --target ${DB_INSTANCE} --version ${MIGRATION_VERSION}"
    depends_on: [check-deploy-lock]
    on_failure: abort

  # ── 타입 4: approval (수동 승인 게이트) ──
  - id: prod-approval
    name: "프로덕션 배포 승인"
    type: approval
    description: "프로덕션 배포 전 수동 승인 필요"
    approval:
      message: "프로덕션에 v${VERSION}을 배포합니다. 계속하시겠습니까?"
      timeout: 600s # 10분 대기
      required_profile: prod # prod 프로파일일 때만 활성화
    depends_on: [run-migration]

  # ── 타입 5: condition (조건 분기) ──
  - id: check-canary
    name: "카나리 배포 여부 판단"
    type: condition
    description: "프로파일이 prod면 카나리 배포, 아니면 일반 배포"
    condition:
      if: "${PROFILE} == prod"
      then: canary-deploy
      else: full-deploy

  # ── 타입 6: parallel (병렬 실행 그룹) ──
  - id: post-deploy-checks
    name: "배포 후 검증 (병렬)"
    type: parallel
    description: "여러 검증을 동시에 실행"
    parallel:
      - id: health-check
        type: api
        request:
          method: GET
          url: "${API_BASE_URL}${HEALTH_CHECK_PATH}"
        expect:
          status: 200

      - id: smoke-test
        type: command
        commands:
          - "python3 scripts/smoke_test.py --env ${PROFILE}"

      - id: metric-check
        type: api
        request:
          method: GET
          url: "https://grafana.example.com/api/ds/query"
          headers:
            Authorization: "Bearer ${GRAFANA_API_TOKEN}"
        expect:
          status: 200
    depends_on: [canary-deploy]
    on_failure: rollback

  # ── 타입 7: workflow (서브 워크플로우 호출) ──
  - id: notify-all
    name: "전체 알림 발송"
    type: workflow
    description: "공통 알림 워크플로우를 재사용"
    workflow:
      path: ".workflows/common-notify.yaml"
      inputs:
        title: "${workflow.name} 배포 완료"
        channel: "#deploy-alerts"
    depends_on: [post-deploy-checks]

  # ── 타입 8: mcp (MCP 서버 도구 호출) ──
  - id: update-notion
    name: "Notion 배포 기록 업데이트"
    type: mcp
    description: "Notion MCP 서버를 통해 배포 로그 페이지 업데이트"
    mcp:
      server: notion
      tool: "notion-update-page"
      arguments:
        page_id: "${NOTION_DEPLOY_PAGE}"
        content: "v${VERSION} 배포 완료 - ${PROFILE}"
    depends_on: [notify-all]
    on_failure: continue # 알림 실패해도 워크플로우는 성공

# ─── retry 설정 (step 레벨) ─────────────────────────
# 개별 step에 retry 블록을 추가할 수 있음
# 예시는 위의 api step에서 config.api_defaults.retry를 상속하되,
# step 레벨에서 오버라이드 가능:
#
#   retry:
#     max_attempts: 5
#     backoff: linear
#     initial_delay: 2s
#     max_delay: 30s

# ─── 알림 ────────────────────────────────────────────
notifications:
  on_success:
    - type: slack
      channel: "#deploy-alerts"
      message: "${workflow.name} (${PROFILE}) 배포 완료 ✅"
    - type: mcp
      server: notion
      tool: "notion-create-comment"
      arguments:
        page_id: "${NOTION_DEPLOY_PAGE}"
        content: "배포 성공"
  on_failure:
    - type: slack
      channel: "#deploy-alerts"
      message: "${workflow.name} (${PROFILE}) 배포 실패 ❌ - ${failed_step.name}"

# ─── [v2] 훅 연동 ───────────────────────────────────
hooks:
  pre_run:
    - "echo '워크플로우 시작: ${workflow.name}'"
    - "git stash" # 실행 전 작업 내용 보관
  post_run:
    - "git stash pop"
  pre_step:
    - "echo '▶ Step: ${current_step.name}'"
  post_step:
    - "echo '✓ Step 완료: ${current_step.name} (${current_step.duration})'"
```

### 5.2 스텝 타입 요약

| 타입        | 용도            | 필수 필드                     | 설명                        |
| ----------- | --------------- | ----------------------------- | --------------------------- |
| `command`   | 쉘 명령 실행    | `commands[]`                  | 기존 방식, bash 커맨드 실행 |
| `api`       | REST API 호출   | `request` (method, url)       | HTTP 요청 + 응답 검증       |
| `skill`     | 다른 스킬 호출  | `skill` (name, arguments)     | Claude Code 스킬 체이닝     |
| `approval`  | 수동 승인 대기  | `approval` (message)          | 사람 확인 후 진행           |
| `condition` | 조건 분기       | `condition` (if, then, else)  | 조건에 따라 다음 step 결정  |
| `parallel`  | 병렬 실행       | `parallel[]` (하위 스텝 목록) | 여러 step 동시 실행         |
| `workflow`  | 서브 워크플로우 | `workflow` (path)             | 다른 YAML 워크플로우 import |
| `mcp`       | MCP 서버 도구   | `mcp` (server, tool)          | 외부 MCP 서버 도구 호출     |

### 5.3 config 섹션 상세

```
config
├── profiles                  # 환경 프로파일 관리
│   ├── default               # 기본 프로파일 (미지정 시 사용)
│   └── available[]           # 사용 가능한 프로파일 목록
├── base                      # 모든 프로파일 공통 변수
├── <profile_name>            # 프로파일별 변수 오버라이드
├── secrets[]                 # 시크릿 참조 (값은 외부에서)
│   ├── source: env           #   시스템 환경변수
│   ├── source: file          #   파일에서 읽기
│   └── source: vault         #   [향후] HashiCorp Vault 등
├── api_defaults              # API 호출 공통 설정
│   ├── timeout               #   기본 타임아웃
│   ├── retry                 #   재시도 정책
│   └── headers               #   공통 헤더
├── mcp_servers[]             # 사용할 MCP 서버 목록
└── required_skills[]         # 의존 스킬 목록
```

### 5.4 변수 참조 체계

| 참조 패턴                 | 설명                      | 예시                                   |
| ------------------------- | ------------------------- | -------------------------------------- |
| `${VAR_NAME}`             | config에서 정의한 변수    | `${SERVICE_NAME}`                      |
| `${PROFILE}`              | 현재 활성 프로파일명      | `dev`, `prod`                          |
| `${steps.STEP_ID.output}` | 이전 스텝의 출력값        | `${steps.pre-check.output.pod_status}` |
| `${steps.STEP_ID.status}` | 이전 스텝의 실행 상태     | `success`, `failed`, `skipped`         |
| `${workflow.name}`        | 워크플로우 메타 정보      | `일본 서비스 배포`                     |
| `${failed_step.name}`     | 실패한 스텝 정보 (알림용) | `DB 백업`                              |
| `${current_step.*}`       | 현재 스텝 정보 (훅용)     | `${current_step.name}`                 |
| `${secrets.SECRET_NAME}`  | 시크릿 값 (런타임 해석)   | `${secrets.DB_PASSWORD}`               |

---

## 6. 사용자 인터랙션 플로우 (v2)

### 6.1 워크플로우 생성 — 프로파일 포함

```
사용자: /workflow-builder create

Claude: 워크플로우를 만들겠습니다.

1. 워크플로우 이름은?
   > 일본 서비스 배포

2. 저장 경로는?
   - [1] .workflows/ (프로젝트)
   - [2] ~/.workflows/ (글로벌)
   - [3] 직접 입력
   > 1

3. 템플릿 선택:
   - [1] deploy (배포)
   - [2] incident (인시던트 대응)
   - [3] infra-scale (인프라 축소/확장)
   - [4] api-health-check (API 헬스체크)
   - [5] skill-chain (스킬 체이닝)
   - [6] blank (빈 템플릿)
   > 1

4. 환경 프로파일을 설정하시겠습니까?
   - [1] 단일 환경 (env만 사용)
   - [2] 다중 프로파일 (dev/staging/prod)
   > 2

5. 프로파일별 차이점을 알려주세요:
   (대화형으로 base → dev → staging → prod 변수 수집)

6. API 호출 step이 필요한가요? (y/n)
   > y
   (API 엔드포인트, 인증 방식, 예상 응답 수집)

7. 다른 스킬을 호출하는 step이 필요한가요? (y/n)
   > y
   (스킬 이름, 인자 수집)

Claude: ✅ 워크플로우 생성 완료
  📁 .workflows/deploy-japan-service.yaml
  📋 프로파일: dev, staging, prod (기본: dev)
  🔗 연동: slack(MCP), db-migration(스킬)
  ⚠️  시크릿 2개 필요: AWS_ACCESS_KEY_ID, SLACK_WEBHOOK_URL
```

### 6.2 프로파일 지정 실행

```
사용자: /workflow-builder run deploy-japan-service --profile prod

Claude: 🚀 "일본 서비스 배포" (prod 프로파일)

  적용된 설정:
  ┌──────────────────┬─────────────────────────────┐
  │ REGION           │ ap-northeast-1              │
  │ CLUSTER          │ prod-jp                     │
  │ API_BASE_URL     │ https://api.example.com     │
  │ DB_INSTANCE      │ prod-oshi-db                │
  └──────────────────┴─────────────────────────────┘

  ⚠️  시크릿 확인:
  - AWS_ACCESS_KEY_ID: ✅ (환경변수 존재)
  - SLACK_WEBHOOK_URL: ✅ (환경변수 존재)
  - DB_PASSWORD: ❌ (파일 없음: ~/.secrets/db_password)

  계속하시겠습니까? (y/n)
```

### 6.3 dry-run 모드

```
사용자: /workflow-builder run deploy-japan-service --profile prod --dry-run

Claude: 🔍 "일본 서비스 배포" (prod) — Dry Run

  Step 1/8: [pre-check] 사전 점검 (command)
    → kubectl get pods -n oshi-no-ko-puzzle --context prod-jp
    → curl -sf https://api.example.com/api/health

  Step 2/8: [check-deploy-lock] 배포 잠금 확인 (api)
    → GET https://api.example.com/deploy/lock?service=oshi-no-ko-puzzle
    → 기대: 200, body에 "locked": false 포함

  Step 3/8: [run-migration] DB 마이그레이션 (skill)
    → /db-migration run --target prod-oshi-db --version ${MIGRATION_VERSION}

  Step 4/8: [prod-approval] 프로덕션 배포 승인 (approval) ⏸️
    → "프로덕션에 v${VERSION}을 배포합니다. 계속하시겠습니까?"

  Step 5/8: [check-canary] 카나리 배포 여부 (condition)
    → prod이므로 → canary-deploy 분기

  Step 6/8: [post-deploy-checks] 배포 후 검증 (parallel)
    → ├─ health-check: GET https://api.example.com/api/health
    → ├─ smoke-test: python3 scripts/smoke_test.py --env prod
    → └─ metric-check: GET https://grafana.example.com/api/ds/query

  Step 7/8: [notify-all] 전체 알림 (workflow)
    → .workflows/common-notify.yaml 호출

  Step 8/8: [update-notion] Notion 업데이트 (mcp)
    → notion > notion-update-page

  ✅ Dry run 완료. 실제 실행하시겠습니까?
```

---

## 7. 서브커맨드 정의 (v2)

| 커맨드                          | 인자                                    | 설명                               |
| ------------------------------- | --------------------------------------- | ---------------------------------- |
| `create`                        | `[템플릿명]`                            | 새 워크플로우 생성 (대화형)        |
| `list`                          | `[--tag TAG]`                           | 등록된 워크플로우 목록 (태그 필터) |
| `show <파일명>`                 | `[--profile PROFILE]`                   | 상세 보기 (프로파일 적용 결과)     |
| `edit <파일명>`                 | —                                       | 워크플로우 수정                    |
| `run <파일명>`                  | `[--profile P] [--dry-run] [--env K=V]` | 실행 가이드                        |
| `validate <파일명>`             | `[--profile P] [--strict]`              | 스키마 + 실행 가능성 검증          |
| `delete <파일명>`               | —                                       | 워크플로우 삭제                    |
| `export <파일명>`               | `[--format md\|json]`                   | 내보내기                           |
| **[v2]** `history <파일명>`     | `[--last N]`                            | 실행 이력 조회                     |
| **[v2]** `template add`         | `<경로>`                                | 커스텀 템플릿 등록                 |
| **[v2]** `config show <파일명>` | `[--profile P]`                         | 병합된 설정값 확인                 |

---

## 8. 설치 위치

| 범위     | 스킬 경로                            | 워크플로우 파일 경로 |
| -------- | ------------------------------------ | -------------------- |
| 프로젝트 | `.claude/skills/workflow-builder/`   | `.workflows/`        |
| 글로벌   | `~/.claude/skills/workflow-builder/` | `~/.workflows/`      |
| 커스텀   | —                                    | 사용자 지정 경로     |

워크플로우 관련 부가 파일:

```
.workflows/
├── deploy-japan-service.yaml      # 워크플로우 정의
├── common-notify.yaml             # 서브 워크플로우
├── .runs/                         # [v2] 실행 이력
│   └── 2026-03-28T14-30-00_deploy-japan-service_prod.log
└── .templates/                    # [v2] 사용자 커스텀 템플릿
    └── my-custom-template.yaml
```

---

## 9. 구현 단계 (로드맵 v2)

### Phase 1: 기본 스킬 구조 (MVP)

- [ ] `SKILL.md` 작성 (frontmatter + 기본 지침, 500줄 이내)
- [ ] `references/SCHEMA.md` 전체 스키마 명세
- [ ] `references/STEP_TYPES.md` 스텝 타입 가이드
- [ ] 빈 템플릿 + deploy 템플릿 작성
- [ ] `create` 커맨드: 대화형 워크플로우 생성 (command 타입만)
- [ ] `list` / `show` 커맨드
- [ ] `validate_workflow.py` 기본 스키마 검증

### Phase 2: config 시스템 + API step

- [ ] `config` 섹션: profiles, base, secrets
- [ ] `resolve_profile.py` 프로파일 병합 스크립트
- [ ] `type: api` step 지원 (request, expect)
- [ ] `config.api_defaults` 공통 설정
- [ ] `run --profile` 프로파일 지정 실행
- [ ] `config show` 서브커맨드
- [ ] `references/CONFIG_GUIDE.md` 작성

### Phase 3: 스킬 체이닝 + 고급 step

- [ ] `type: skill` step (다른 스킬 호출)
- [ ] `type: approval` step (수동 승인 게이트)
- [ ] `type: condition` step (조건 분기)
- [ ] `type: parallel` step (병렬 실행)
- [ ] `type: workflow` step (서브 워크플로우)
- [ ] step 간 `outputs` / `inputs` 데이터 전달
- [ ] `retry` 블록 (횟수, 백오프)

### Phase 4: MCP 연동 + 운영 기능

- [ ] `type: mcp` step (MCP 서버 도구 호출)
- [ ] `config.mcp_servers` 설정
- [ ] `hooks` 섹션 (pre_run, post_run, pre_step, post_step)
- [ ] `run --dry-run` 시뮬레이션
- [ ] `.workflows/.runs/` 실행 이력 로깅
- [ ] `history` 서브커맨드
- [ ] `template add` 커스텀 템플릿 등록

### Phase 5: 생태계 연동

- [ ] n8n 워크플로우 변환 (`export --format n8n`)
- [ ] whichcli 연동 (CLI 도구 설치를 step으로)
- [ ] 실행 가능성 사전 검증 (커맨드 존재, API 도달 가능)
- [ ] 워크플로우 시각화 (Mermaid 다이어그램 생성)
- [ ] 팀 공유: Git 기반 워크플로우 버전 관리 가이드

---

## 10. 보안 고려 사항

### 시크릿 관리 원칙

1. **YAML 파일에 시크릿 값을 절대 직접 기입하지 않음**
2. `secrets` 섹션에는 참조(source + path)만 기록
3. 시크릿 소스 우선순위: `env`(환경변수) → `file`(파일) → `vault`(향후)
4. `run` 실행 시 시크릿 존재 여부를 사전 확인하고 누락 시 경고

### 실행 안전성

1. **모든 커맨드는 실행 전 사용자 확인** (approval 외에도 매 step 확인 옵션)
2. `--dry-run`으로 전체 흐름 사전 검토
3. `prod` 프로파일에는 자동으로 `approval` step 삽입 권장
4. `allowed-tools`로 스킬이 접근 가능한 도구 제한
5. `rollback_commands`를 모든 위험 step에 권장

### Git 관리

1. `.workflows/` 디렉토리는 Git 추적 (팀 공유)
2. `.workflows/.runs/`는 `.gitignore`에 추가
3. 시크릿 파일 경로는 `.gitignore`에 추가

---

## 11. 실전 예제: 인시던트 대응 워크플로우

```yaml
workflow:
  name: "CMI RDS 장애 대응"
  description: "RDS 하이퍼바이저 메모리 장애 시 대응 워크플로우"
  version: "1.0.0"
  tags: [incident, rds, database]

config:
  base:
    SERVICE_NAME: "oshi-no-ko-puzzle"
    SLACK_CHANNEL: "#incident-response"
  profiles:
    default: prod
    available: [prod]
  prod:
    DB_PRIMARY: "prod-oshi-primary"
    DB_REPLICA: "prod-oshi-replica"
    GRAFANA_DASHBOARD: "https://grafana.example.com/d/rds-overview"
  secrets:
    - name: AWS_ACCESS_KEY_ID
      source: env
    - name: PAGERDUTY_TOKEN
      source: env

steps:
  - id: assess
    name: "상황 파악"
    type: parallel
    parallel:
      - id: check-rds-status
        type: command
        commands:
          - "aws rds describe-db-instances --db-instance-identifier ${DB_PRIMARY}"
      - id: check-app-health
        type: api
        request:
          method: GET
          url: "${API_BASE_URL}/health"
        expect:
          status: 200
      - id: check-connections
        type: command
        commands:
          - "aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name DatabaseConnections"

  - id: notify-team
    name: "팀 알림"
    type: mcp
    mcp:
      server: slack
      tool: "send-message"
      arguments:
        channel: "${SLACK_CHANNEL}"
        message: "🚨 RDS 장애 감지: ${DB_PRIMARY} 확인 필요"
    depends_on: [assess]

  - id: decide-failover
    name: "페일오버 결정"
    type: approval
    approval:
      message: |
        DB 상태 확인 결과:
        - Primary: ${steps.check-rds-status.output}
        - App Health: ${steps.check-app-health.status}
        - Connections: ${steps.check-connections.output}

        페일오버를 진행하시겠습니까?
    depends_on: [notify-team]

  - id: execute-failover
    name: "페일오버 실행"
    type: command
    commands:
      - "aws rds failover-db-cluster --db-cluster-identifier ${DB_PRIMARY}-cluster"
    depends_on: [decide-failover]
    on_failure: abort
    rollback_commands:
      - "echo '수동 복구 필요: AWS 콘솔에서 확인'"
    retry:
      max_attempts: 2
      backoff: linear
      initial_delay: 10s

  - id: verify-recovery
    name: "복구 검증"
    type: parallel
    parallel:
      - id: verify-db
        type: command
        commands:
          - "aws rds describe-db-instances --db-instance-identifier ${DB_PRIMARY}"
      - id: verify-app
        type: api
        request:
          method: GET
          url: "${API_BASE_URL}/health"
        expect:
          status: 200
      - id: verify-grafana
        type: command
        commands:
          - "echo '📊 Grafana 확인: ${GRAFANA_DASHBOARD}'"
    depends_on: [execute-failover]

  - id: write-postmortem
    name: "사후 분석 문서 생성"
    type: skill
    skill:
      name: "internal-comms"
      arguments: "인시던트 리포트 작성: RDS 하이퍼바이저 메모리 장애"
    depends_on: [verify-recovery]

  - id: update-notion
    name: "Notion 인시던트 로그 업데이트"
    type: mcp
    mcp:
      server: notion
      tool: "notion-update-page"
      arguments:
        page_id: "${NOTION_INCIDENT_PAGE}"
        content: "장애 복구 완료 - 사후 분석 문서 생성됨"
    depends_on: [write-postmortem]
    on_failure: continue
```

---

## 부록: 참고 자료

- [Claude Code Skills 공식 문서](https://code.claude.com/docs/en/skills)
- [Agent Skills 오픈 스탠다드](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)
- [Skill Authoring Best Practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)
- [커스텀 Skills 생성 가이드](https://support.claude.com/en/articles/12512198-how-to-create-custom-skills)
