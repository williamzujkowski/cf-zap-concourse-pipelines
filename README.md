# cf-zap-concourse-pipelines

Concourse CI pipelines for running [OWASP ZAP](https://www.zaproxy.org/) security scans against Cloud Foundry applications using the [ZAP Automation Framework](https://www.zaproxy.org/docs/automate/automation-framework/).

## Overview

This repository provides ready-to-use Concourse pipelines that run ZAP security scans against your CF-deployed applications. It supports two scan modes:

- **Baseline scan**: Fast, passive-only scan. Spiders the application and checks for common security issues without sending attack payloads. Safe to run frequently.
- **Full scan**: Comprehensive scan including active testing. Sends attack payloads to detect vulnerabilities like XSS, SQL injection, and more. Run during off-peak hours.

Both scan types produce HTML and JSON reports as Concourse build artifacts.

## Architecture

```
+-------------------+       +-------------------+       +-------------------+
|                   |       |                   |       |                   |
|  Concourse CI     +------>+  ZAP Container    +------>+  CF Application   |
|                   |       |  (zap-stable)     |       |  (target)         |
|  - Triggers scan  |       |  - Loads plan     |       |                   |
|  - Stores reports |       |  - Spiders app    |       |                   |
|                   |       |  - Runs checks    |       |                   |
+-------------------+       |  - Produces HTML  |       +-------------------+
                            |    + JSON reports |
                            +-------------------+

Pipeline flow:
  fly set-pipeline --> baseline-scan job --> ZAP passive scan --> reports artifact
                   --> full-scan job     --> ZAP active scan  --> reports artifact
```

## Prerequisites

- [Concourse CI](https://concourse-ci.org/) instance (v7.0+)
- [`fly` CLI](https://concourse-ci.org/fly.html) authenticated to your Concourse instance
- Docker (used by Concourse workers to pull the ZAP image)
- A deployed application with a reachable URL to scan

## Quick Start

### 1. Clone this repository

```bash
git clone https://github.com/williamzujkowski/cf-zap-concourse-pipelines.git
cd cf-zap-concourse-pipelines
```

### 2. Set the pipeline

```bash
fly -t my-target set-pipeline \
  -p zap-scan \
  -c pipelines/zap-scan.yml \
  -v target_url=https://my-app.apps.example.com \
  -v scan_type=baseline \
  -v fail_on=High \
  -v zap_image_tag=stable
```

Or use a params file:

```bash
fly -t my-target set-pipeline \
  -p zap-scan \
  -c pipelines/zap-scan.yml \
  -l examples/params-baseline.yml
```

### 3. Unpause and trigger the scan

```bash
fly -t my-target unpause-pipeline -p zap-scan
fly -t my-target trigger-job -j zap-scan/baseline-scan --watch
```

### 4. Download reports

```bash
fly -t my-target get-artifact -j zap-scan/baseline-scan -o reports=./reports
```

Reports are saved as HTML (human-readable) and JSON (machine-parseable) in the `reports/` directory.

## Parameters

| Parameter       | Required | Default   | Description                                                        |
| --------------- | -------- | --------- | ------------------------------------------------------------------ |
| `target_url`    | Yes      | -         | URL of the application to scan (e.g., `https://app.example.com`)   |
| `scan_type`     | No       | `baseline`| Scan type: `baseline` (passive only) or `full` (passive + active)  |
| `fail_on`       | No       | `High`    | Minimum alert risk level that causes the job to fail: `High`, `Medium`, `Low`, or `Informational` |
| `zap_image_tag` | No       | `stable`  | Docker image tag for `zaproxy/zap-stable` (e.g., `stable`, `2.17.0`) |

## Repository Structure

```
cf-zap-concourse-pipelines/
  pipelines/
    zap-scan.yml              # Concourse pipeline definition
  tasks/
    zap-scan/
      task.yml                # Concourse task definition
      run.sh                  # Task execution script
      plans/
        baseline.yaml         # ZAP AF plan for passive scans
        full.yaml             # ZAP AF plan for active scans
  examples/
    params-baseline.yml       # Example params for baseline scan
    params-full.yml           # Example params for full scan
    local-concourse.md        # Running Concourse locally
  docs/
    tuning.md                 # Scan tuning guide
    auth.md                   # Authentication support (planned)
    troubleshooting.md        # Troubleshooting guide
    integration-workflow.md   # End-to-end workflow with kind-deployment
    adr/
      001-zap-automation-framework.md
      002-threshold-defaults.md
```

## Scan Types

### Baseline Scan

The baseline scan is designed to be safe and fast. It performs only passive analysis:

1. Spiders the target application (max 2 minutes, depth 5).
2. Waits for passive scan rules to complete (max 5 minutes).
3. Generates reports.
4. Fails the build if High-risk alerts are found (configurable).

Typical runtime: 3-8 minutes depending on application size.

### Full Scan

The full scan includes active testing and is more thorough:

1. Spiders the target application (max 5 minutes, depth 10).
2. Waits for passive scan rules to complete (max 10 minutes).
3. Runs active scan with attack payloads (Medium strength by default, High for XSS/SQLi).
4. Generates reports.
5. Fails the build if Medium-risk or higher alerts are found (configurable).

Typical runtime: 15-45 minutes depending on application size and attack surface.

**Warning**: Active scans send attack payloads to the target. Only run against applications you own and in environments where this is acceptable.

## Integration with kind-deployment

This project is designed to work alongside [kind-deployment](https://github.com/williamzujkowski/kind-deployment) for local Kubernetes + CF development. See [docs/integration-workflow.md](docs/integration-workflow.md) for the end-to-end workflow.

## Further Reading

- [OWASP ZAP Automation Framework](https://www.zaproxy.org/docs/automate/automation-framework/)
- [ZAP Docker Images](https://www.zaproxy.org/docs/docker/about/)
- [Concourse CI Documentation](https://concourse-ci.org/docs.html)
- [Scan Tuning Guide](docs/tuning.md)
- [Troubleshooting](docs/troubleshooting.md)
- [ADR: Why ZAP Automation Framework](docs/adr/001-zap-automation-framework.md)

## License

Apache License 2.0. See [LICENSE](LICENSE) for details.
