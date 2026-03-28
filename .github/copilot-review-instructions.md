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

## Helm Chart Completeness

### Rule 5: New Helm chart must include service.yaml

If the PR creates or modifies a Helm chart (`helm/*/`), verify that `templates/service.yaml` exists. Without it, the pod runs but has no Kubernetes DNS routing — other services cannot reach it.

Required templates for any service:
- `templates/deployment.yaml`
- `templates/service.yaml`
- `templates/secret.yaml`

### Rule 6: Helm values secrets must never contain real credentials

If the PR modifies `values-dev.yaml` or `values-test.yaml`, verify that `secrets.*` entries contain only placeholders like `LOCAL_DEV_ONLY` or `PLACEHOLDER` — never real passwords. Real secrets are injected by CI/CD via `HELM_SECRETS`.

## Infrastructure Package Rules

### Rule 7: No duplicate authentication registration

If the service uses `AddGranitInfrastructure(configuration)` or `AddCommonInfrastructure(configuration)`, it MUST NOT also manually register JWT Bearer authentication (`AddAuthentication().AddJwtBearer()`). This causes "Scheme already exists: Bearer" crash at startup.

### Rule 8: Authentication config section naming

`AddGranitInfrastructure` expects config under `Authentication:AuthService:*` (Issuer, Audience, SigningKey). If the PR uses `Authentication:Jwt:*` instead, flag it — the service will fail to register `ICurrentUserService`.

## Resource & Performance Rules

### Rule 9: Memory requests vs actual usage

If the PR modifies Kubernetes resource requests/limits in `values-*.yaml`:
- Memory `requests` MUST be higher than the service's actual idle memory footprint
- .NET services typically use 150-300Mi at idle — setting `requests: 128Mi` causes HPA to over-scale
- CPU-based HPA is preferred for .NET services (not memory-based) because .NET has high baseline memory usage

### Rule 10: HPA configuration sanity

If HPA is enabled in values:
- `targetCPUUtilizationPercentage` should be 60-80% (not 50% — too aggressive for .NET)
- Memory-based HPA (`targetMemoryUtilizationPercentage`) should generally NOT be used for .NET services
- `minReplicas` should be at least 2 for TEST environment

## Code Quality Rules

### Rule 11: Startup validation completeness
If the PR adds validation for one config section (e.g., `Services:Conversation`), check that ALL sub-properties used in the code are also validated at startup. Partial validation leads to runtime errors instead of fast startup failures.

### Rule 12: appsettings.json consistency
If a new config section is added to code, verify it also has a default/empty entry in `appsettings.json` for documentation purposes.

### Rule 13: File encoding
All files MUST be UTF-8 without BOM. Flag any file with BOM character (U+FEFF). BOM breaks YAML parsers, Git diffs, and CI/CD workflows.

### Rule 14: Database naming convention
All Granit services use `UseSnakeCaseNamingConvention()`. If the PR adds new entities or migrations:
- Table names must be snake_case (e.g., `user_personal_data`, not `UserPersonalData`)
- Column names must be snake_case
- `__EFMigrationsHistory` columns are `migration_id`, `product_version` (snake_case)
