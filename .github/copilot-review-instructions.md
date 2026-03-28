# Copilot Code Review Instructions — GranitLab

These instructions apply to all repositories in the GranitLab organization unless overridden by a repo-level `.github/copilot-review-instructions.md`.

## Critical: Helm Values ↔ Application Config Sync

All Granit microservices use Helm charts for Kubernetes deployment. Configuration is passed via `env:` entries in `values-dev.yaml` / `values-test.yaml`.

### Rule 1: New required config → must exist in Helm values

If the PR introduces any of the following patterns:
- New `configuration.GetSection("X")` or `configuration.GetValue<T>("X")` calls
- New `IOptions<T>` / `IOptionsSnapshot<T>` bindings
- New `throw new InvalidOperationException(...)` for missing config
- New `GetConnectionString("X")` calls
- New properties on existing options/settings classes that are validated at startup

Then check that matching `env:` entries exist in **both**:
- `helm/*/values-dev.yaml`
- `helm/*/values-test.yaml`

.NET configuration maps environment variables using `__` as section separator. For example:
- `Services:Conversation:BaseUrl` in appsettings → `Services__Conversation__BaseUrl` in Helm values env

**If matching env vars are missing, this MUST be flagged as a blocking issue.** Missing config causes CrashLoopBackOff on deploy.

### Rule 2: New service dependency → must have K8s DNS URL

If the PR adds a new HTTP client, service client, or `BaseUrl` configuration for an inter-service call, verify:
- The URL uses Kubernetes internal DNS format: `http://{service}.app-services.svc.cluster.local:{port}`
- The port matches the K8s Service port (80 for most services, 8080 for CoreService)

### Rule 3: New secrets → must be in deployment template

If the PR adds new `secretKeyRef` usage or new sensitive config (API keys, passwords, tokens), verify:
- The secret key exists in `helm/*/templates/secret.yaml`
- The `values-dev.yaml` and `values-test.yaml` have corresponding `secrets.*` entries

### Rule 4: Database schema changes → check migration impact

If the PR modifies:
- Entity classes (new properties, removed properties, type changes)
- DbContext configuration (new entity mappings, index changes)
- New migration files

Then verify:
- Migration file is included in the PR
- If tables are altered, the service user must have ownership of those tables

## Code Quality Rules

### Startup validation completeness
If the PR adds validation for one config section, check that ALL sub-properties used in the code are also validated at startup. Partial validation leads to runtime errors instead of fast startup failures.

### appsettings.json consistency
If a new config section is added to code, verify it also has a default/empty entry in `appsettings.json` for documentation purposes.

### File encoding
All files MUST be UTF-8 without BOM. Flag any file with BOM character (U+FEFF).
