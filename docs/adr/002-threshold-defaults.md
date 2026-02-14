# ADR 002: Default Failure Threshold at High

## Status

Accepted

## Date

2026-02-14

## Context

ZAP produces alerts at four risk levels: Informational, Low, Medium, and High. The `FAIL_ON` parameter controls which alert level causes the Concourse job to fail.

When introducing security scanning into a CI pipeline, teams must balance thoroughness against practicality. A threshold that is too strict causes alert fatigue and leads teams to ignore or disable scanning. A threshold that is too lenient misses important findings.

## Decision

The default `FAIL_ON` threshold is **High**. Only High-risk alerts cause the pipeline to fail by default.

## Reasons

### Avoid alert fatigue

Most applications produce Medium and Low findings on initial scans. Examples include missing security headers (CSP, X-Content-Type-Options), cookie flags, and version disclosure. While these are worth addressing, failing the build on every one of them makes the pipeline unusable until all are resolved.

Starting with `High` allows teams to adopt scanning incrementally.

### High alerts are actionable

High-risk alerts in ZAP indicate likely exploitable vulnerabilities: SQL injection, cross-site scripting, remote code execution. These findings warrant immediate attention and should block deployments.

### Teams can tighten over time

The threshold is configurable per pipeline. The recommended progression:

1. **Start with `High`**: Fix critical vulnerabilities. Get the team comfortable with scanning.
2. **Move to `Medium`**: Once High findings are resolved, tighten to catch configuration issues.
3. **Consider `Low`**: For high-security applications, tighten further to enforce defense-in-depth.

This graduated approach is more sustainable than starting strict and relaxing later (which creates a false sense of progress).

### Consistent with industry practice

DAST tools in CI/CD commonly default to failing only on High or Critical findings. This matches the expectations of development teams integrating security scanning for the first time.

## Consequences

- Baseline scans with `FAIL_ON=High` will pass for most well-maintained applications, providing a smooth onboarding experience.
- Medium and Low findings are still reported in the scan output and HTML reports. They are not hidden, only non-blocking.
- Teams that want stricter enforcement can set `FAIL_ON=Medium` or `FAIL_ON=Low` via the pipeline parameters.
- The full scan defaults to `FAIL_ON=Medium` in the example params (`examples/params-full.yml`) to demonstrate the tighter threshold for comprehensive scans.
