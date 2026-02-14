# Security Policy

## Supported Versions

| Version | Supported          |
| ------- | ------------------ |
| main    | Yes                |

## Reporting a Vulnerability

If you discover a security vulnerability in this project, please report it responsibly.

**Do not open a public GitHub issue for security vulnerabilities.**

Instead, please email the maintainer directly or use GitHub's private vulnerability reporting feature:

1. Go to the **Security** tab of this repository.
2. Click **Report a vulnerability**.
3. Provide a description of the vulnerability, steps to reproduce, and any potential impact.

You should receive a response within 72 hours acknowledging the report. We will work with you to understand the issue and coordinate a fix.

## Scope

This repository contains Concourse pipeline definitions and ZAP automation plans. Security concerns may include:

- **Credential exposure**: Pipeline parameters or task scripts leaking secrets.
- **Injection risks**: Unsanitized input in shell scripts or YAML templates.
- **ZAP misconfiguration**: Scan plans that could cause unintended denial-of-service against target applications.

## Best Practices

- Never commit target URLs with credentials embedded (e.g., `https://user:pass@example.com`).
- Use Concourse credential management (Vault, CredHub) for sensitive parameters.
- Review ZAP scan plans before running against production environments.
- Pin ZAP image tags to specific versions for reproducible scans.
