# End-to-End Integration Workflow

This guide walks through the complete workflow of deploying an application with [kind-deployment](https://github.com/williamzujkowski/kind-deployment) and scanning it with ZAP via these Concourse pipelines.

## Prerequisites

- Docker installed and running
- `kubectl` installed
- `fly` CLI installed and authenticated to a Concourse instance
- `cf` CLI installed (Cloud Foundry CLI)
- Git

## Step 1: Set Up the Local Cluster

Clone the kind-deployment fork and bring up the cluster:

```bash
git clone https://github.com/williamzujkowski/kind-deployment.git
cd kind-deployment
make up
```

This creates a local Kubernetes cluster with Kind and deploys the necessary platform components.

## Step 2: Bootstrap Cloud Foundry

Deploy Cloud Foundry (Korifi) on the cluster:

```bash
make bootstrap
```

Wait for all pods to reach Running status:

```bash
kubectl get pods -A --watch
```

## Step 3: Deploy a Sample Application

Push a sample application to get a route URL:

```bash
cf push hello-js -p samples/hello-js
```

Note the route URL from the output, for example:

```
routes:          hello-js.apps.127-0-0-1.nip.io
```

Verify the application is accessible:

```bash
curl https://hello-js.apps.127-0-0-1.nip.io
```

## Step 4: Set the ZAP Scan Pipeline

Clone this repository and set the pipeline with the application URL:

```bash
cd ..
git clone https://github.com/williamzujkowski/cf-zap-concourse-pipelines.git
cd cf-zap-concourse-pipelines

fly -t local set-pipeline \
  -p zap-scan \
  -c pipelines/zap-scan.yml \
  -v target_url=https://hello-js.apps.127-0-0-1.nip.io \
  -v scan_type=baseline \
  -v fail_on=High \
  -v zap_image_tag=stable
```

## Step 5: Run the Baseline Scan

Unpause and trigger the baseline scan:

```bash
fly -t local unpause-pipeline -p zap-scan
fly -t local trigger-job -j zap-scan/baseline-scan --watch
```

The baseline scan takes approximately 3-8 minutes. It performs passive-only checks (no attack payloads).

## Step 6: Review Reports

Download the scan reports:

```bash
mkdir -p reports
fly -t local get-artifact -j zap-scan/baseline-scan -o reports=./reports
```

Open the HTML report in your browser:

```bash
open reports/zap-baseline-report.html    # macOS
xdg-open reports/zap-baseline-report.html # Linux
```

The report shows:

- Summary of alerts by risk level
- Detailed findings with descriptions, solutions, and references
- List of URLs tested

## Step 7: (Optional) Run a Full Scan

For a comprehensive scan including active testing:

```bash
fly -t local set-pipeline \
  -p zap-scan-full \
  -c pipelines/zap-scan.yml \
  -v target_url=https://hello-js.apps.127-0-0-1.nip.io \
  -v scan_type=full \
  -v fail_on=Medium \
  -v zap_image_tag=stable

fly -t local unpause-pipeline -p zap-scan-full
fly -t local trigger-job -j zap-scan-full/full-scan --watch
```

The full scan takes 15-45 minutes and sends attack payloads. Only run against applications you own.

## Step 8: Set Up Nightly Scans (Optional)

The pipeline includes a `daily-timer` resource. To enable automatic daily baseline scans, modify the baseline-scan job to trigger on the timer:

In `pipelines/zap-scan.yml`, change:

```yaml
- get: daily-timer
  trigger: false
```

to:

```yaml
- get: daily-timer
  trigger: true
```

Then update the pipeline:

```bash
fly -t local set-pipeline \
  -p zap-scan \
  -c pipelines/zap-scan.yml \
  -v target_url=https://hello-js.apps.127-0-0-1.nip.io \
  -v scan_type=baseline \
  -v fail_on=High \
  -v zap_image_tag=stable
```

The scan will now trigger automatically between 2:00-3:00 AM ET daily.

## Step 9: Teardown

When you are done:

```bash
# Destroy the Concourse pipeline
fly -t local destroy-pipeline -p zap-scan

# Tear down the kind cluster
cd ../kind-deployment
make down
```

## Workflow Summary

```
kind-deployment                    cf-zap-concourse-pipelines
===============                    ==========================
1. make up                         4. fly set-pipeline
2. make bootstrap                  5. fly trigger-job (baseline)
3. cf push hello-js                6. Review HTML report
                                   7. fly trigger-job (full) [optional]
                                   8. Enable nightly timer [optional]
                                   9. fly destroy-pipeline
make down
```

## Next Steps

- Customize the scan by editing the plans in `tasks/zap-scan/plans/`. See [docs/tuning.md](tuning.md).
- Add alert filters for known false positives. See the tuning guide.
- Integrate reports into your team's review process (e.g., post SARIF to GitHub Security tab).
