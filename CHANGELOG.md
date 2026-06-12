# Changelog

All notable changes to this spec. Format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/); versioning follows
[SemVer](https://semver.org) as defined in SPEC.md section 10.

## [1.0.0] - 2026-06-13

Stable. The PHP core library
([email-guard-core-php](https://github.com/Emailsherlock1/email-guard-core-php))
runs green against the full vector set as the reference implementation.
No behavior changes against rc.1; vector files now carry
`spec_version: 1.0.0`.

## [1.0.0-rc.1] - 2026-06-11

First public draft. Becomes 1.0.0 when the PHP core-lib runs green against
the full vector set.

### Added

- SPEC.md: normalization, local checks (syntax profile, reserved TLD,
  disposable snapshot), remote check semantics, decision model
  (block_on / review_on / fail_open), result object, conformance procedure,
  disposable snapshot format.
- 67 conformance vectors across six categories: syntax, reserved-tld,
  disposable, decision, api-mapping, fail-open.
- JSON Schema for the vector file format.
- Disposable snapshot test fixture.
