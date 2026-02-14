# Tuning ZAP Scans

This guide covers how to adjust ZAP scan behavior for your specific application and CI requirements.

## Understanding Alert Risk Levels

ZAP categorizes findings by risk level:

| Risk Level      | Meaning                                           | Examples                                      |
| --------------- | ------------------------------------------------- | --------------------------------------------- |
| **High**        | Likely exploitable, significant impact             | SQL injection, XSS, remote code execution     |
| **Medium**      | Potentially exploitable or notable configuration   | Missing security headers, CSRF, open redirect |
| **Low**         | Minor issues, defense-in-depth concerns            | Cookie without Secure flag, information leak  |
| **Informational** | Not a vulnerability, but useful context          | Server version disclosed, unusual response    |

## Choosing a Failure Threshold

The `FAIL_ON` parameter controls which alert level causes the pipeline to fail.

- **`High`** (recommended default): Only fail on clearly exploitable vulnerabilities. Best for initial adoption to avoid alert fatigue.
- **`Medium`**: Fail on exploitable and configuration issues. Good once your team has addressed High findings.
- **`Low`**: Strict mode. Useful for high-security applications.
- **`Informational`**: Maximum strictness. Not recommended for CI as it generates noise.

See [ADR 002](adr/002-threshold-defaults.md) for the reasoning behind defaulting to High.

## Suppressing False Positives

Use the `alertFilter` job type in your automation plan to suppress known false positives. The filter MUST appear before scanning jobs in the YAML.

```yaml
- type: alertFilter
  parameters:
    alertFilters:
      # Suppress a specific rule entirely
      - ruleId: 10021
        newRisk: "False Positive"
        enabled: true

      # Suppress for a specific URL pattern only
      - ruleId: 10036
        newRisk: "False Positive"
        urlRegex: ".*/api/health.*"
        enabled: true
```

### Common CF-Specific False Positives

| Rule ID | Alert Name                      | Why it fires on CF apps                                     |
| ------- | ------------------------------- | ----------------------------------------------------------- |
| 10021   | X-Content-Type-Options Missing  | CF system endpoints (health checks) often omit this header  |
| 10036   | Server Header Version Disclosed | Gorouter adds a Server header with version information      |
| 10038   | Content Security Policy Missing | Not all CF apps set CSP, especially APIs                    |
| 10098   | Cross-Domain Misconfiguration   | CF routing can trigger cross-domain checks                  |

## Adjusting Spider Behavior

### Depth and Duration

For **small applications** (< 50 pages):
```yaml
- type: spider
  parameters:
    maxDuration: 1
    maxDepth: 3
    maxChildren: 10
```

For **large applications** (hundreds of pages):
```yaml
- type: spider
  parameters:
    maxDuration: 10
    maxDepth: 15
    maxChildren: 50
```

### Ajax Spider

Enable the Ajax spider for JavaScript-heavy single-page applications (SPAs) that render content dynamically. This is commented out by default in `full.yaml` because it requires a headless browser and adds significant scan time.

```yaml
- type: spiderAjax
  parameters:
    context: "full-scan-context"
    maxDuration: 5
    maxCrawlDepth: 5
    browserId: "firefox-headless"
```

Note: The ZAP Docker image includes Firefox. The Ajax spider can add 5-15 minutes to scan time.

## Active Scan Policy Tuning

### Strength and Threshold

Each active scan rule has two settings:

- **Strength**: How many attack variants to try (`low`, `medium`, `high`, `insane`). Higher = slower but more thorough.
- **Threshold**: How sensitive the detection is (`off`, `low`, `medium`, `high`). Lower = more findings but more false positives.

### Per-Rule Overrides

Override specific rules in the `activeScan` job:

```yaml
- type: activeScan
  parameters:
    defaultPolicy:
      strength: "medium"
      threshold: "medium"
    rules:
      # Disable a noisy rule
      - id: 40025
        threshold: "off"

      # Make XSS detection more aggressive
      - id: 40012
        strength: "high"
        threshold: "low"
```

### Common Rule IDs

| ID    | Name                        | Notes                                |
| ----- | --------------------------- | ------------------------------------ |
| 40012 | Reflected XSS               | High priority, worth running at high |
| 40014 | Persistent XSS              | High priority                        |
| 40018 | SQL Injection               | High priority                        |
| 40019 | SQL Injection (MySQL)       | Database-specific                    |
| 40024 | SQL Injection (PostgreSQL)  | Database-specific                    |
| 40025 | Expression Language Injection | Noisy on some frameworks            |
| 90019 | Server Side Code Injection  | Can cause errors on fragile apps     |
| 90020 | Remote OS Command Injection | Can cause errors on fragile apps     |

## Performance Tuning for CI

To reduce scan time in CI pipelines:

1. **Reduce spider depth**: Most vulnerabilities are found in the first 3-5 levels.
2. **Lower active scan strength**: Use `low` strength for faster scans with fewer variants.
3. **Disable slow rules**: Turn off rules that are unlikely to apply (e.g., database-specific rules if you know your stack).
4. **Limit scan duration**: Set `maxScanDurationInMins` on the active scan.
5. **Skip Ajax spider**: Unless your app is a JavaScript SPA, the standard spider is sufficient.
6. **Reduce `maxAlertsPerRule`**: Lower from 10 to 5 if you only need to know a rule triggered, not every instance.

### Example: Fast CI Scan (Under 5 Minutes)

```yaml
jobs:
  - type: passiveScan-config
    parameters:
      maxAlertsPerRule: 5
      scanOnlyInScope: true

  - type: spider
    parameters:
      maxDuration: 1
      maxDepth: 3
      maxChildren: 10

  - type: passiveScan-wait
    parameters:
      maxDuration: 2

  - type: activeScan
    parameters:
      maxScanDurationInMins: 5
      defaultPolicy:
        strength: "low"
        threshold: "medium"

  - type: exitStatus
    parameters:
      errorLevel: "High"
```
