# Changelog

All notable changes to this project are documented here. This project adheres to
[Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] - 2026-07-21

### Added

- Initial release.
- Bounded wait on `kubectl rollout status --timeout`.
- Failure diagnostics: deployment & pod `describe`, current + previous container
  logs, and recent namespace events, grouped in the job log.
- Automatic rollback to the previous revision via `kubectl rollout undo`
  (toggle with `rollback: false`).
- Honest `$GITHUB_STEP_SUMMARY` reporting the real rollback outcome.
- Inputs: `deployment`, `namespace`, `timeout`, `rollback`, `log-tail`.
