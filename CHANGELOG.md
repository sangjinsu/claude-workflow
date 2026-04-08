# Changelog

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
