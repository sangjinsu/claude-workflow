# Changelog

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
