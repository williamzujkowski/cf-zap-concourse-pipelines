# Troubleshooting

Common issues and solutions when running ZAP scans in Concourse.

## ZAP Cannot Reach the Target

**Symptom**: Scan fails immediately with connection errors or "Failed to connect" messages.

**Causes and solutions**:

1. **DNS resolution**: The Concourse worker container needs to resolve the target hostname. If using a local DNS name (e.g., `*.nip.io`), ensure the worker's DNS configuration can resolve it.

2. **Network connectivity**: The Concourse worker must have network access to the target. If the target is behind a firewall or in a private network, ensure the worker is deployed in the same network or has appropriate routing.

3. **Wrong URL**: Verify the URL is correct and includes the scheme (`https://`). Test with `curl` from a similar environment first.

4. **Target is down**: Confirm the application is running and responding before triggering the scan.

## Self-Signed Certificate Errors

**Symptom**: ZAP refuses to connect to HTTPS targets with certificate validation errors.

**Solutions**:

1. **Accept all certificates** (development/testing only): ZAP's Automation Framework can be configured to accept untrusted certificates. Add to your plan's environment:

   ```yaml
   env:
     parameters:
       failOnError: true
   ```

   And run ZAP with the `-config connection.dnsTtlSuccessfulQueries=-1` option.

2. **Import the CA certificate**: If you have a custom CA, you can import it into ZAP's trust store. This requires a custom Docker image or a pre-scan setup step.

3. **Use HTTP for local development**: If the target is a local development instance, consider using HTTP instead of HTTPS to avoid certificate issues entirely.

## Scan Takes Too Long

**Symptom**: The scan runs for an extended period or times out.

**Solutions**:

1. **Reduce spider depth**: Lower `maxDepth` in the spider configuration. Most vulnerabilities are found in the first 3-5 levels.

2. **Set duration limits**: Ensure `maxDuration` is set on spider, passive scan wait, and active scan jobs.

3. **Lower active scan strength**: Use `low` strength instead of `medium` or `high`. This reduces the number of attack variants per endpoint.

4. **Disable slow rules**: Turn off specific active scan rules that are not relevant to your application (e.g., database-specific injection rules if you know your database type).

5. **Skip Ajax spider**: The Ajax spider adds significant time. Only enable it for JavaScript-heavy SPAs.

See [docs/tuning.md](tuning.md) for detailed tuning guidance.

## Too Many False Positives

**Symptom**: The scan reports many findings that are not actual vulnerabilities.

**Solutions**:

1. **Use alertFilter**: Add `alertFilter` entries to your plan to suppress known false positives. See [docs/tuning.md](tuning.md) for common CF-specific false positives.

2. **Raise the failure threshold**: Set `FAIL_ON=High` to only fail on clearly exploitable issues.

3. **Adjust rule thresholds**: Set individual rule thresholds to `medium` or `high` to reduce sensitivity.

4. **Review and triage**: Not all findings need to be fixed immediately. Use the JSON report to build a triage process.

## Concourse Task OOM (Out of Memory)

**Symptom**: The task is killed with OOM errors, or ZAP crashes during large scans.

**Solutions**:

1. **Increase task memory**: If your Concourse deployment supports resource limits, increase the memory allocation for the task. Add to your pipeline:

   ```yaml
   task: run-zap-scan
   config:
     container_limits:
       memory: 4gb
   ```

2. **Reduce scan scope**: Limit the spider depth and active scan duration to reduce ZAP's memory usage.

3. **Use `maxAlertsPerRule`**: Set this to a lower value (e.g., 5) to reduce the number of alerts ZAP keeps in memory.

## Reports Not Appearing

**Symptom**: The scan completes but no reports are found in the build artifacts.

**Causes and solutions**:

1. **Report directory mismatch**: Ensure the `reportDir` in your plan matches the directory ZAP writes to. The plans in this repo use `/zap/wrk/reports/`.

2. **ZAP errored before report generation**: Check the scan output for errors. If ZAP fails during scanning, it may not reach the report generation step.

3. **Output mapping**: Verify the Concourse task output mapping. The `reports` output in `task.yml` must match the directory where `run.sh` copies reports.

4. **Permissions**: Ensure the report directory is writable. The `run.sh` script creates the directory with `mkdir -p`.

## Exit Code Reference

| Exit Code | Meaning                                                    |
| --------- | ---------------------------------------------------------- |
| 0         | Scan completed, no alerts at or above the threshold level  |
| 1         | ZAP encountered an error (not an alert finding)            |
| 2         | Scan completed, alerts found at or above the threshold     |

- **Exit code 1**: Indicates a ZAP configuration or runtime error. Check the scan output for error messages. Common causes: invalid automation plan YAML, unreachable target, malformed URL.

- **Exit code 2**: Indicates the scan found security alerts at or above your configured `FAIL_ON` level. This is the expected failure mode. Review the generated reports for details.

## ZAP Version Issues

**Symptom**: Automation plan features do not work, or YAML syntax is rejected.

**Solutions**:

1. **Check the ZAP version**: The automation plans in this repo are tested with ZAP 2.17.0 (the `stable` tag as of February 2026). Older versions may not support all features.

2. **Pin the image tag**: Instead of using `stable` (which may change), pin to a specific version like `2.17.0` for reproducibility.

3. **Check the changelog**: Review the [ZAP release notes](https://www.zaproxy.org/docs/desktop/releases/) for changes that may affect your plans.

## Getting Help

- [OWASP ZAP User Group](https://groups.google.com/g/zaproxy-users): Community support for ZAP questions.
- [ZAP GitHub Issues](https://github.com/zaproxy/zaproxy/issues): For ZAP bugs.
- [Concourse Discussions](https://github.com/concourse/concourse/discussions): For Concourse CI questions.
