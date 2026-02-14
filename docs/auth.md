# Authentication Support

## Current State (v1)

Version 1 of these pipelines does **not** support authenticated scanning. All scans run as an unauthenticated user, meaning they can only test publicly accessible pages and endpoints.

This is intentional for the initial release: unauthenticated scans are simpler to configure, safer to run in CI, and still catch a significant class of vulnerabilities (missing headers, information disclosure, publicly exploitable flaws).

## Planned Authentication Methods

The following authentication methods are planned for future versions:

### Form-Based Authentication

For applications with a login form, ZAP can be configured to submit credentials and maintain a session.

```yaml
# Example ZAP AF authentication configuration (for reference)
env:
  contexts:
    - name: "authenticated-context"
      urls:
        - "https://my-app.example.com"
      authentication:
        method: "form"
        parameters:
          loginPageUrl: "https://my-app.example.com/login"
          loginRequestUrl: "https://my-app.example.com/auth"
          loginRequestBody: "username={%username%}&password={%password%}"
      sessionManagement:
        method: "cookie"
      users:
        - name: "test-user"
          credentials:
            username: "scanner@example.com"
            password: "{{ZAP_AUTH_PASSWORD}}"
```

### Bearer Token Authentication

For API-first applications that use bearer tokens (JWT, OAuth2 tokens):

```yaml
# Example ZAP AF bearer token configuration (for reference)
env:
  contexts:
    - name: "api-context"
      urls:
        - "https://api.example.com"
      authentication:
        method: "http"
        parameters:
          hostname: "api.example.com"
          port: 443
          realm: ""
      sessionManagement:
        method: "script"
        parameters:
          script: "jwt-session.js"
```

### OAuth2 / SSO

For applications using OAuth2 or SSO providers, the approach typically involves:

1. Obtaining a token outside of ZAP (via client credentials flow or pre-authentication).
2. Injecting the token into ZAP's requests via a session management script or header replacement.

This is the most complex auth scenario and will require custom scripting support.

## Security Considerations for Authenticated Scans

When authenticated scanning is implemented:

- **Credentials must never be committed** to this repository. Use Concourse credential management (Vault, CredHub) to inject secrets at runtime.
- **Use dedicated test accounts** with limited privileges. Never scan with admin credentials.
- **Scope the scan carefully** to avoid modifying data through authenticated endpoints (e.g., exclude DELETE endpoints, admin panels).
- **Monitor the application** during authenticated scans as active scanning may create, modify, or delete data.

## How to Add Auth When Ready

When authentication support is added, the changes will involve:

1. Adding authentication configuration to the ZAP automation plans (`plans/baseline.yaml`, `plans/full.yaml`).
2. Adding new parameters to `task.yml` for auth-related settings.
3. Updating `run.sh` to handle credential injection from Concourse parameters.
4. Creating a dedicated `plans/authenticated-baseline.yaml` plan.
5. Updating the pipeline to optionally pass auth parameters.

The approach will keep unauthenticated scanning as the default, with authentication as an opt-in configuration.
