# workflow-builder 개발 가이드

Claude Code 플러그인으로 YAML 워크플로우를 정의하고 실행합니다.

## 아키텍처

순수 Prompt-Only. 스크립트 없음. Claude가 YAML을 Read로 직접 읽고, 변수를 치환하고, Bash로 실행.

```
/workflow-builder run <name>
  → Read로 .workflows/<name>.yaml 읽기
  → config.base 변수 추출 + ${VAR} 치환
  → depends_on 기반 실행 순서 결정
  → step별 commands를 Bash로 순차 실행
```

## 플러그인 구조

```
.claude-plugin/plugin.json    # 플러그인 manifest
.claude-plugin/marketplace.json # 마켓플레이스 메타
skills/workflow-builder/SKILL.md # 핵심 스킬 (모든 로직)
assets/templates/              # 워크플로우 템플릿
```

## MVP 범위

- `type: command` step만 지원
- 서브커맨드: run, create, list, show, validate
- 외부 의존성 없음

## Phase 2+ 로드맵

- config profiles (dev/staging/prod)
- type: api, skill, approval step
- 병렬 실행
- hooks (SessionStart, PreToolUse)
- MCP 서버 연동
- 워크플로우 레코더 (세션 자동 캡처)

## 워크플로우 YAML 스키마

```yaml
workflow:
  name: "이름"           # 필수
  description: "설명"    # 선택
  version: "1.0.0"      # 선택

config:
  base:                  # 선택
    VAR: "value"

steps:                   # 필수
  - id: step-id         # 필수, 고유
    type: command        # 필수
    commands:            # 필수
      - "cmd ${VAR}"
    depends_on: [other]  # 선택
    on_failure: abort    # 선택 (abort|continue)
```

## Trust Model

워크플로우 작성자 = 실행자 (self-trust). 공유받은 워크플로우는 commands를 반드시 확인.

## Skill routing

When the user's request matches an available skill, ALWAYS invoke it using the Skill
tool as your FIRST action. Do NOT answer directly, do NOT use other tools first.
The skill has specialized workflows that produce better results than ad-hoc answers.

Key routing rules:
- Product ideas, "is this worth building", brainstorming → invoke office-hours
- Bugs, errors, "why is this broken", 500 errors → invoke investigate
- Ship, deploy, push, create PR → invoke ship
- QA, test the site, find bugs → invoke qa
- Code review, check my diff → invoke review
- Update docs after shipping → invoke document-release
- Weekly retro → invoke retro
- Design system, brand → invoke design-consultation
- Visual audit, design polish → invoke design-review
- Architecture review → invoke plan-eng-review
- Save progress, checkpoint, resume → invoke checkpoint
- Code quality, health check → invoke health
