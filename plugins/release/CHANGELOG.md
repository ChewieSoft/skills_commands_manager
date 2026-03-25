# Changelog — release

## [1.1.0] — 2026-03-25

### Fixed

- **Contributors listed wrong usernames** — git author names (`JorUge`, `Jorge Ferrari`) don't match GitHub usernames (`j0ruge`). Now resolves via `gh api` email lookup instead of `git log --format="%aN"`
- Exclude bot accounts (`noreply@anthropic.com`) from contributors list
- Same person with multiple git author names no longer appears as multiple contributors

### Changed

- Contributors section now uses `gh api /search/users?q=EMAIL` to resolve actual GitHub usernames
- Added quality rule #7: contributors must be resolved GitHub usernames, never fabricated

## [1.0.0] — 2026-03-16

### Added

- Initial release of the `release` plugin
- Automated GitHub Release creation via `gh CLI`
- Categorized release notes from git history (conventional commits)
- Dynamic project name detection (package.json or git directory name)
- PT-BR release notes with emoji-categorized sections
- Dependency diff analysis from package.json changes
- PR and contributor aggregation
