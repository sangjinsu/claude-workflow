# Changelog

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
