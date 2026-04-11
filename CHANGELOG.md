# Changelog

## [0.5.0] - 2026-04-11

### Added
- **type: api step**: YAML에서 선언적으로 HTTP API를 호출하는 새로운 step 타입
  - `method`, `url`, `headers`, `body`, `timeout` 지원
  - `expect.status` 기본 `[200-299]`로 HTTP 에러 자동 감지
  - `retry` 설정: 숫자 shorthand (`retry: 3`) 또는 객체 (`{max, delay, on}`)
  - `retry.on` 기본 `[429, 500, 502, 503, 504]` + exponential backoff
  - curl exit code→한국어 에러 메시지 매핑 (6=DNS, 7=연결 거부, 28=타임아웃, 35=SSL)
  - `---HTTP_STATUS---` sentinel 기반 response body/status code 분리
  - heredoc 기반 body 전달로 shell injection 방지
  - 민감 헤더 패턴 기반 마스킹 (auth/key/token/secret/cookie)
  - step output pipeline과 완전 통합 (`${steps.<id>.output}` → response body)
  - 병렬 실행 지원 (command와 동일)
- `api-demo.yaml`: JSONPlaceholder API를 사용한 GET/POST 예시 워크플로우
- `api-basic` 템플릿: `/workflow-builder create api-basic`으로 빠른 시작
- `validate --repeat N`에서 api step은 curl 명령 비교만 수행 (비결정적 응답 제외)

### Why
/autoplan 리뷰(CEO/Eng/DX 3-phase)에서 에러 처리가 type: api의 핵심 가치로 확인됨. Approach A(minimal curl wrapper)로 시작하되, expect.status 기본값, curl exit code 매핑, sentinel 분리, heredoc injection 방지를 포함하여 raw curl 대비 확실한 차별점을 제공.

## [0.4.0] - 2026-04-09

### Added
- **Step Output Pipeline**: 이전 step의 실행 결과를 다음 step에서 `${steps.<id>.output}`로 참조
  - type: command의 마지막 command stdout 캡처
  - type: ai의 응답 텍스트 캡처
  - type: approval의 "approved"/"denied" 결과 캡처
- 검증 규칙:
  - 참조는 반드시 depends_on에 (직접/전이적) 포함되어야 함
  - 존재하지 않는 step id 참조 시 에러
  - 10KB 초과 시 경고, 100KB 초과 시 에러
- validate 서브커맨드가 step output 참조를 검증
- step-output-demo.yaml: 2개 독립 step → 결합 step → AI 분석 파이프라인

### Why
/autoplan review (v0.3.0)에서 type: api/skill의 "숨겨진 비용"으로 지목된 step output pipeline이 이제 구현되었습니다. 이것은 Phase 4+ 기능들(type: api 등)의 prerequisite 파운데이션입니다.

## [0.3.2] - 2026-04-08

### Added
- approval-parallel-test.yaml: 병렬 레벨 예외 검증용 워크플로우
- approval-continue-test.yaml: on_failure continue 모드 검증용
- SKILL.md: type: approval timeout 동작 명세 (무기한 blocking, MVP는 대화형만)

### QA
- type: approval 4개 시나리오 전수 검증 완료 (approve, deny, parallel exception, continue)
- Health: 100/100 (8개 검증 항목)

## [0.3.1] - 2026-04-08

### Added
- `type: approval` step — 사용자 확인 게이트로 중요 작업 전 승인 요청
- AskUserQuestion UI를 통한 명시적 옵션 제공 (read -p 보다 안전)
- `default: approve | deny` 설정 (기본: deny)
- `${VAR}` 변수 치환을 message에서 지원
- 병렬 실행 시 approval step은 단독 처리 (다중 승인 UX 방지)
- approval-demo.yaml: profile + approval + AI 요약 조합 예시

### Notes
- /autoplan 리뷰에 따라 Phase 3 scope를 `type: approval` 단일 기능으로 축소
- type: api, type: skill, type: mcp는 Phase 4+ 또는 별도 플러그인으로 연기

## [0.3.0] - 2026-04-08

### Added
- 병렬 실행: 같은 의존성 레벨의 step들이 동시에 실행됨
- 4단계 알고리즘이 병렬 레벨을 계산 (topological levels)
- `validate --repeat N` 옵션으로 결정성 검증
- 결정성 테스트 워크플로우 (`.workflows/determinism-test.yaml`)
- 병렬 실행 데모 워크플로우 (`.workflows/parallel-demo.yaml`)

### Changed
- 6단계 실행 루프가 병렬 레벨 단위로 실행
- 같은 레벨의 command step은 같은 메시지에서 동시 호출

## [0.2.1] - 2026-04-07

### Added
- `config.profiles` — 환경별 변수 세트 정의 (dev/staging/prod 등)
- `--profile <name>` 옵션 — run 시 프로파일 선택
- 우선순위: `--set` > `--profile` > `config.base`
- 다중 환경 배포 예시 워크플로우 (`.workflows/multi-env-deploy.yaml`)
- show 서브커맨드가 정의된 profiles 표시

## [0.2.0] - 2026-04-06

### Added
- `type: ai` step — Claude AI가 워크플로우 step에서 분석, 판단, 요약을 수행
- `record` subcommand — 자연어 설명으로 워크플로우를 대화형 생성
- AI 로그 분석 예시 워크플로우 (`.workflows/ai-analysis.yaml`)

## [0.1.1] - 2026-04-05

### Fixed
- allowed-tools changed from restrictive prefix patterns to unrestricted Bash for workflow command execution
- Template path resolution from broken `${CLAUDE_SKILL_DIR}` reference to explicit `assets/templates/` lookup
- Skipped steps (unsupported type) now treated as completed for dependency resolution

### Added
- Error message templates with search paths, exit codes, and actionable guidance
- Variable substitution rules: `${VAR}` only, `$(cmd)` and `$VAR` left untouched
- Concrete cycle detection algorithm with formatted error output
- Quick start section in README with expected output example
- YAML parsing error handling in SKILL.md
- `argument-hint` in SKILL.md frontmatter
- Sample workflows: hello.yaml, prompt-test.yaml
- gstack skill routing rules in CLAUDE.md

## [0.1.0] - 2026-04-04

### Added
- Initial release
- 5 subcommands: run, create, list, show, validate
- Variable substitution with `${VAR}` syntax
- DAG-based step ordering with `depends_on`
- Unsupported step type graceful skip (treated as completed for dependency resolution)
- Two templates: deploy, blank
- Project-local (`.workflows/`) and global (`~/.workflows/`) storage
- Trust model documentation
