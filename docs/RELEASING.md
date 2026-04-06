---
summary: Tag-driven release flow — changelog → tag → push → GitHub Actions creates Release
read_when: Publishing a new version, creating a release, or updating the changelog for release
---

# Releasing

## Flow

1. **Update changelog** — use `update-changelog` skill or manually edit `CHANGELOG.md`.
2. **Commit** — `git add CHANGELOG.md && git commit -m "docs: update CHANGELOG for X.Y.Z"`.
3. **Tag** — `git tag X.Y.Z` (bare semver, no `v` prefix).
4. **Push** — `git push origin main --tags`.
5. **GitHub Actions** automatically creates a Release with the changelog body for that version.

## Tag Format

Bare semver: `0.1.0`, `1.0.0`, `2.3.1`. No `v` prefix.

## Changelog Format

The CI extracts the release body from `CHANGELOG.md` by matching the `## [X.Y.Z]` header. Make sure the version in the tag matches the version in the changelog heading exactly.
