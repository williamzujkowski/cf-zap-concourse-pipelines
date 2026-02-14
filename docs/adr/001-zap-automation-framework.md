# ADR 001: Use ZAP Automation Framework Over Legacy Scripts

## Status

Accepted

## Date

2026-02-14

## Context

OWASP ZAP provides two approaches for automated scanning:

1. **Legacy Python scripts**: `zap-baseline.py`, `zap-full-scan.py`, and `zap-api-scan.py`. These were the traditional method for running ZAP in CI/CD pipelines. They use command-line arguments to configure scan behavior.

2. **Automation Framework**: Introduced in ZAP 2.11 and now the officially recommended approach. Uses declarative YAML plans to define scan behavior, with a composable job system for spiders, scans, reports, and alert filtering.

The legacy scripts are now deprecated by the ZAP project and receive limited maintenance. The Automation Framework is actively developed and is the focus of new features.

## Decision

We will use the ZAP Automation Framework (YAML plans) for all scan configurations.

## Reasons

### Declarative and version-controlled

Automation plans are YAML files that live in the repository alongside the pipeline definitions. Changes to scan configuration are reviewed and tracked through the normal git workflow, providing full audit history.

### Composable

The job-based structure allows mixing and matching scan components. A plan can include any combination of spiders, passive scans, active scans, alert filters, and report generators in any order. This is more flexible than the fixed behavior of the legacy scripts.

### Officially recommended

The ZAP project recommends the Automation Framework for all new integrations. From the ZAP documentation: the Automation Framework is the recommended way to automate ZAP. The legacy scripts are maintained for backward compatibility but do not receive new features.

### Better alert management

The `alertFilter` job type provides fine-grained control over false positive suppression, including per-URL and per-parameter filtering. The legacy scripts only supported global alert overrides.

### Consistent output

The `report` job type supports multiple output formats (HTML, SARIF JSON, Markdown) with consistent configuration. The legacy scripts had varying report capabilities.

## ZAP Version Pinning

The `stable` Docker tag points to ZAP 2.17.0 as of February 2026. For reproducible scans:

- Development/testing: Use `stable` for convenience.
- Production CI: Pin to a specific version (e.g., `2.17.0`) and update deliberately.

## Consequences

- All scan configuration is managed through YAML plans in `tasks/zap-scan/plans/`.
- Teams familiar with the legacy scripts (`zap-baseline.py`) will need to learn the Automation Framework YAML syntax.
- The ZAP documentation for the Automation Framework is the primary reference: https://www.zaproxy.org/docs/automate/automation-framework/

## References

- [ZAP Automation Framework Documentation](https://www.zaproxy.org/docs/automate/automation-framework/)
- [ZAP Docker Documentation](https://www.zaproxy.org/docs/docker/about/)
- [ZAP 2.17.0 Release Notes](https://www.zaproxy.org/docs/desktop/releases/2.17.0/)
