wwwww# [ENG-ADR-004] CI Verification

**Document ID**: ENG-ADR-004
**Version**: 1.0.0
**Status**: Accepted
**Authority Level**: Normative
**Document Type**: Architecture Decision Record
**Owner**: Engineering
**Last Reviewed**: 2026-08-08
**Next Review**: 2027-02-08
**Review Cadence**: 6 Months
**Date**: 2026-08-08
**Decision Makers**: Principal Software Architect
**Supersedes**: None
**Related ADRs**: [ENG-ADR-001](./ENG-ADR-001-local-first-quality-gates.md)

## Status
Accepted

## Context
With the introduction of strict Tier A (Local) Quality Gates, a developer might theoretically bypass local hooks using `--no-verify` or due to an incorrectly configured local environment. If the remote repository trusts the local validation implicitly, broken code will merge into `main`.

## Decision
Continuous Integration (Tier B) is designated as the Ultimate Source of Truth. The remote CI environment operates on a zero-trust model regarding the developer's local state.

CI MUST fully re-execute all Tier A checks (linting, formatting, typing) via the canonical `pnpm validate:ci` command before proceeding to Tier B heavy checks (integration tests, database migrations, security scans) included in the same graph.

## Alternatives Considered
- **Trust Local Validation**: Skipping lint/type checks in CI if the developer's pre-push hook passed. Rejected due to the inability to cryptographically verify local execution and the risk of environment drift.

## Consequences
- **Positive**: Absolute guarantee of code quality before merging. Eliminates the "it works on my machine" failure mode.
- **Negative**: Redundant execution of linting/typing in CI slightly increases pipeline duration, though caching strategies mitigate this impact.

## Related Documents
[ENG-005](../04-ci-cd-architecture.md), [ENG-004](../03-quality-gates.md)
