# Artefact naming conventions

This document defines the naming and folder conventions for AIS integration artefacts across workflows, APIM, configuration, messaging, secrets, tests, and delivery branches to ensure consistent implementation and domain-scoped deployments.

---

## Table of contents

- [1. Two-audience principle](#two-audience-principle)
- [2. Identity tokens](#identity-tokens)
- [3. Logic App + schema artefacts (co-located)](#logic-app--schema-artefacts-co-located)
- [4. APIM artefacts (policy co-located with spec)](#apim-artefacts-policy-co-located-with-spec)
- [5. App Configuration keys](#app-configuration-keys)
- [6. Service Bus queues & topics](#service-bus-queues--topics)
- [7. Key Vault secrets](#key-vault-secrets)
- [8. Test fixtures](#test-fixtures)
- [9. P4 file names (SFTP/Blob)](#p4-file-names-sftpblob)
- [10. Git flow branching strategy](#git-flow-branching-strategy)
- [11. Worked example](#worked-example)
- [12. Anti-patterns (never do this)](#anti-patterns-never-do-this)
- [13. Related documents](#related-documents)

---

<a id="two-audience-principle"></a>
## 1. Two-audience principle

Every artefact name serves one of two audiences. Never mix concerns:

| Audience | Consumers | Pattern | Exposes |
|----------|-----------|---------|---------|
| **Public / consumer** | External callers, APIM subscribers | `{domain}-{function}-v{n}` | Business capability only |
| **Internal / ops** | Developers, support, pipelines | `{source}-{target}-{function}[-{direction}]` | Full routing identity |

**Rule: no `integrationId` (e.g. `INT0033_7`) in any artefact name.** Integration IDs belong only in:
- Logs and run summaries
- P4 file names (see §9)

---

<a id="identity-tokens"></a>
## 2. Identity tokens

All tokens: **lowercase kebab-case only**. No underscores, no dots.

| Token | Definition | Example |
|-------|-----------|---------|
| `{domain}` | External-system grouping (HLD isolation boundary) | `otmb`, `hr`, `fin` |
| `{function}` | Business function slug — **immutable**, registry-governed | `order-release-split`, `hr-sync` |
| `{version}` | API contract version | `v1`, `v2` |
| `{source}` | Source system slug | `otm`, `qone`, `d365` |
| `{target}` | Target system slug | `d365`, `otm` |
| `{direction}` | Platform-relative flow: `inbound` (external→platform) / `outbound` (platform→external) | `inbound` |
| `{wave}` | Delivery wave number | `1`, `2` |
| `{env}` | Environment — **always 3-char** | `sbx`, `dev`, `test`, `prd` |

> **`{env}` values:** `sbx` · `dev` · `test` · `prd`. Use `prd` everywhere — not `prod`. This applies to resource names, branch names, and config keys without exception.

---

<a id="logic-app--schema-artefacts-co-located"></a>
## 3. Logic App + schema artefacts (co-located)

Schemas live **inside** the workflow folder — no separate top-level `schemas/` directory. Related artefacts (workflow, schema, liquid) are always found together.

### Folder structure

```
workflows/
  wave-{wave}/
    {domain}/
      {function}/
        workflow.json                                  ← Logic App workflow definition
        connections.json                               ← SB service provider connection
        {source}-{target}-{function}.liquid            ← Liquid transform
        {domain}-{function}-request.schema.json        ← Inbound request schema
        {domain}-{entity}-canonical.schema.json        ← Canonical/target schema
```

**Concrete example (INT0033_7):**

```
workflows/wave-1/otmb/order-release-split/
  workflow.json
  connections.json
  otm-d365-order-release-split.liquid
  otmb-order-release-split-request.schema.json
  otmb-release-canonical.schema.json
```

### Artefact name patterns

| Artefact | Pattern | Example |
|----------|---------|---------|
| Workflow name (Logic App runtime) | `wf-{source}-{target}-{function}[-{direction}]` | `wf-otm-d365-order-release-split-inbound` |
| Workflow folder | `workflows/wave-{wave}/{domain}/{function}/` | `workflows/wave-1/otmb/order-release-split/` |
| Liquid transform | `{source}-{target}-{function}.liquid` | `otm-d365-order-release-split.liquid` |
| Request schema | `{domain}-{function}-request.schema.json` | `otmb-order-release-split-request.schema.json` |
| Canonical schema | `{domain}-{entity}-canonical.schema.json` | `otmb-release-canonical.schema.json` |

> **Two-audience split:** The folder path uses public identity (`{domain}/{function}`); the workflow runtime name and liquid filename use internal identity (`{source}-{target}-{function}`).

### Logic App workflow name constraints

Logic App Standard workflow names must use **alphanumeric and hyphens only** — no underscores (the runtime silently refuses to register the workflow) and no spaces. There is no documented character maximum beyond the NTFS path limit (~255 chars).

If a name would be ambiguous due to very long system slugs, register a short alias at that point — short aliases are not pre-defined, they are added on demand when a real naming conflict arises.

> **Microsoft best practice alignment:** Co-location of tightly coupled artefacts is the recommended pattern in Azure DevOps/GitHub repos. Logic Apps Standard's own runtime structure (`{workflow}/workflow.json`) co-locates all workflow files. Separating schemas into a remote folder adds navigation overhead with no access-control benefit since all files deploy together.

---

<a id="apim-artefacts-policy-co-located-with-spec"></a>
## 4. APIM artefacts (policy co-located with spec)

Policy lives **in the same folder as the API spec** — no separate `policies/` subfolder. API spec and policy always deploy as a pair and are always found together.

### Folder structure

```
apim/
  apis/
    {domain}/
      {domain}-{function}-v{n}-api.yaml        ← OpenAPI spec (REST) or .wsdl (SOAP)
      {domain}-{function}-v{n}-policy.xml      ← Inbound/backend policy
  products/
    {domain}/
      {domain}-product.json                     ← Product definition (domain scoped)
  fragments/
    {domain}/
      {domain}-inbound-common.xml              ← Shared inbound fragment (domain scoped)
  subscriptions/
    {domain}/
      {domain}-subscription.json               ← Subscription definition (domain scoped)
  named-values/
    shared/
      shared-named-values.json                 ← Instance-wide non-secret values
    {domain}/
      {domain}-named-values.json               ← Domain-specific non-secret values
```

**Concrete example (SOAP):**

```
apim/apis/otmb/
  otmb-order-release-split-v1-api.wsdl
  otmb-order-release-split-v1-policy.xml

apim/products/otmb/
  otmb-product.json

apim/fragments/otmb/
  otmb-inbound-common.xml

apim/subscriptions/otmb/
  otmb-subscription.json

apim/named-values/shared/
  shared-named-values.json

apim/named-values/otmb/
  otmb-named-values.json
```

### Artefact name patterns

| Artefact | Pattern | Example |
|----------|---------|---------|
| API spec (REST) | `{domain}-{function}-v{n}-api.yaml` | `otmb-order-release-split-v1-api.yaml` |
| API spec (SOAP) | `{domain}-{function}-v{n}-api.wsdl` | `otmb-order-release-split-v1-api.wsdl` |
| Policy | `{domain}-{function}-v{n}-policy.xml` | `otmb-order-release-split-v1-policy.xml` |
| Product file | `{domain}-product.json` (in `apim/products/{domain}/`) | `otmb-product.json` |
| Subscription file | `{domain}-subscription.json` (in `apim/subscriptions/{domain}/`) | `otmb-subscription.json` |
| Subscription name | `ves-sub-{domain}-{env}` | `ves-sub-otmb-prd` |
| Inbound fragment | `{domain}-inbound-common.xml` (in `apim/fragments/{domain}/`) | `otmb-inbound-common.xml` |

### APIM Named Values naming patterns

| Artefact | Pattern | Example |
|----------|---------|---------|
| Shared named values file | `shared-named-values.json` (in `apim/named-values/shared/`) | `shared-named-values.json` |
| Shared named value key | `shared-{purpose}` | `shared-tenant-id` |
| Domain named values file | `{domain}-named-values.json` (in `apim/named-values/{domain}/`) | `otmb-named-values.json` |
| Domain named value key | `{domain}-{purpose}` | `otmb-backend-base-url` |
| Key Vault-backed named value key | `{domain}-{purpose}-kvref` | `otmb-backend-client-secret-kvref` |

**Rules:**
- Keep named value keys lowercase kebab-case.
- Keep shared keys under `shared-*`; keep integration-specific keys under `{domain}-*`.
- Do not store secret values directly; use Key Vault-backed named values for secrets.

**Named values rule:** Store only non-secret values in APIM named-values files. Secrets must be referenced from Azure Key Vault-backed named values, not committed in repository files.

> **Microsoft best practice alignment:** Microsoft's APIM DevOps Resource Kit and APIOps tooling co-locate policy with the API definition (single folder per API). Separating policies breaks the "single source of truth" for one API's contract.

---

<a id="app-configuration-keys"></a>
## 5. App Configuration keys

Use one of these three formats based on scope:

- **Global shared** (used across all domains): `ais:shared:{category}:{resource}:{setting}`
- **Domain shared** (used across workflows in one domain): `ais:{domain}:shared:{category}:{resource}:{setting}`
- **Function specific** (used by one integration/function): `ais:{domain}:{function}:{category}:{key}`

### Format tokens

| Token | Meaning | Example |
|-------|---------|---------|
| `{domain}` | Domain slug | `otmb` |
| `{function}` | Function slug | `order-release-split` |
| `{category}` | Concern area | `source`, `transform`, `observability`, `deadletter`, `oauth` |
| `{resource}` | Logical resource grouping | `appinsights`, `servicebus`, `keyvault`, `dlq-replay` |
| `{setting}` / `{key}` | Specific setting name (kebab-case) | `ikey`, `queue-name`, `liquid-maps`, `status-codes` |

### Example types

| Type | Format | Example |
|------|--------|---------|
| Global observability setting | `ais:shared:{category}:{resource}:{setting}` | `ais:shared:observability:appinsights:ikey` |
| Global dead-letter reasons | `ais:shared:{category}:{setting}` | `ais:shared:deadletter:reasons` |
| Domain transform mapping catalog | `ais:{domain}:shared:{category}:{setting}` | `ais:otmb:shared:transform:liquid-maps` |
| Domain dead-letter reasons | `ais:{domain}:shared:{category}:{setting}` | `ais:otmb:shared:deadletter:reasons` |
| Domain replay queue list | `ais:{domain}:shared:{category}:{resource}` | `ais:otmb:shared:dlq-replay:queues` |
| Domain replay reason list | `ais:{domain}:shared:{category}:{resource}` | `ais:otmb:shared:dlq-replay:reasons` |
| Domain integration name map | `ais:{domain}:shared:{category}:{setting}` | `ais:otmb:shared:integration:names` |
| Domain Key Vault URL | `ais:{domain}:shared:{category}:{setting}` | `ais:otmb:shared:keyvault:url` |
| Domain Key Vault secret-name map | `ais:{domain}:shared:{category}:{setting}` | `ais:otmb:shared:keyvault:secret-names` |
| Domain OAuth endpoint map | `ais:{domain}:shared:{category}:{setting}` | `ais:otmb:shared:oauth:endpoints` |
| Domain OAuth grant type | `ais:{domain}:shared:{category}:{setting}` | `ais:otmb:shared:oauth:grant-type` |
| Domain schema map | `ais:{domain}:shared:{category}:{setting}` | `ais:otmb:shared:validation:schemas` |
| Domain Service Bus queue-name map | `ais:{domain}:shared:{category}:{setting}` | `ais:otmb:shared:servicebus:queue-names` |
| Domain API response code map | `ais:{domain}:shared:{category}:{setting}` | `ais:otmb:shared:api-response:status-codes` |
| Domain log message catalog | `ais:{domain}:shared:{category}:{setting}` | `ais:otmb:shared:observability:log-messages` |
| Function source queue name | `ais:{domain}:{function}:{category}:{key}` | `ais:otmb:order-release-split:source:queue-name` |
| Function transform default values | `ais:{domain}:{function}:{category}:{key}` | `ais:otmb:nota-fiscal:transform:default-values` |

**Rules:**
- All segments lowercase; use kebab-case for multi-word segments.
- Use `ais:shared:*` only for truly global values.
- Use `ais:{domain}:shared:*` for domain-wide values.
- Use `ais:{domain}:{function}:*` for function-specific values.
- Never use `integrationId` as a segment.
- Do not mix `:` and `/` separators in a key.
- **C# auto-binding:** kebab-case leaf segments (e.g. `queue-name`) do not map to C# properties by default. When consuming App Configuration values in a Function App or .NET host, use `IConfiguration.GetValue<T>("ais:otmb:order-release-split:source:queue-name")` with the full key string, or register a key-name transform (e.g. `TrimKeyPrefix` + `Replace("-", "")` / `ConfigurationKeyNameAttribute`) to bind to typed options classes. Do **not** rename keys to PascalCase — that breaks the colon-delimited filtering convention used by all SDK consumers.

---

<a id="service-bus-queues--topics"></a>
## 6. Service Bus queues & topics

Format: `{source}-{target}-{function}-{direction}`

No `sb-` prefix — redundant inside a Service Bus namespace (per CAF). Existing `sb-` prefixed queues are renamed during wave migration.

| Type | Pattern | Example |
|------|---------|---------|
| Session queue (P2 inbound) | `{source}-{target}-{function}-inbound` | `otm-d365-order-release-split-inbound` |
| Polling queue (P2 polling) | `{source}-{target}-{function}-inbound` | `qualityone-d365-plm-inbound` |
| Outbound queue (P3) | `{source}-{target}-{function}-outbound` | `d365-otm-order-modify-outbound` |
| Request queue (P10) | `{source}-{target}-{function}-request` | `d365-longview-tax-request` |
| Reply queue (P10) | `{source}-{target}-{function}-reply` | `d365-longview-tax-reply` |
| Topic (P9 pub-sub) | `{source}-{target}-{function}-topic` | `d365-bottomline-payment-topic` |
| Subscription | `{consumer}` | `d365` |
| Dead-letter queue | Auto-appended by Azure: `…/$DeadLetterQueue` | — |

**Character limits:**
- Queue and topic names: max **260 characters** — pattern is well within this
- **Subscription names: max 50 characters** — keep `{consumer}` aliases short

> **HLD §3.2 abstract notation:** `{namespace}/{domain}-{direction}` is the logical representation used in architecture diagrams. The Azure queue name above is the concrete deployment form, adding `{source}` and `{target}` for full self-documentation in a shared namespace with 41+ integrations.

---

<a id="key-vault-secrets"></a>
## 7. Key Vault secrets

Format: `{source}-{target}-{purpose}`

| Example | Meaning |
|---------|---------|
| `ais-otmb-oauth-client-id` | AIS to OTM Brazil OAuth client id |
| `ais-otmb-oauth-client-secret` | AIS to OTM Brazil OAuth client secret |
| `ais-bottomline-sftp-key` | AIS to Bottomline SFTP private key |
| `ais-bottomline-pgp-private-key` | AIS to Bottomline PGP private key for P4 file encryption |

**Rules:**
- **Alphanumeric and hyphens only** — no dots, no underscores (Azure Key Vault constraint: `^[0-9a-zA-Z-]+$`)
- Max 127 characters
- Lowercase throughout
- Use directional source-target prefixes to avoid ambiguity between opposite flows (for example, `ais-otmb-*` vs `otmb-ais-*`)
- Env isolation is provided by the vault itself (`kvdevaisweu001`, `kvprdaisweu001`) — do not add an env suffix to the secret name

## Azure Key Vault Reference Pattern

### Overview

To ensure secure management of sensitive configuration values, this solution uses **Azure Key Vault References** instead of storing secrets directly within Azure Logic App application settings.

With this approach, application settings contain a reference to a secret stored in **Azure Key Vault**, while the actual secret value remains securely managed within the Key Vault.

This pattern helps protect sensitive information such as:

- Connection strings
- API keys
- Application Insights Instrumentation Keys
- Client secrets
- Authentication credentials
- Other environment-specific secrets

---

### Configuration Format

```text
@Microsoft.KeyVault(SecretUri=https://pes-kv-s-sit-ais-weu-001.vault.azure.net/secrets/ais-appInsights-iKey)
```

---

### Example

| Application Setting | Value |
|---------------------|---------|
| APPINSIGHTS_INSTRUMENTATIONKEY | `@Microsoft.KeyVault(SecretUri=https://pes-kv-s-sit-ais-weu-001.vault.azure.net/secrets/ais-appInsights-iKey)` |

---

### How It Works

1. Sensitive values are stored securely as secrets in Azure Key Vault.
2. The Logic App application setting contains a Key Vault Reference instead of the actual secret value.
3. During runtime, Azure resolves the Key Vault Reference and retrieves the corresponding secret value.
4. The Logic App accesses the resolved value through the configured application setting.
5. The secret value is never exposed in the workflow definition, ARM/Bicep templates, source code repository, or deployment pipelines.

---

<a id="test-fixtures"></a>
## 8. Test fixtures

| Artefact | Pattern | Example |
|----------|---------|---------|
| Happy path fixture | `{domain}-{function}-happy.json` | `otmb-order-release-split-happy.json` |
| Failure fixture | `{domain}-{function}-failure.json` | `otmb-order-release-split-failure.json` |
| Duplicate fixture | `{domain}-{function}-duplicate.json` | `otmb-order-release-split-duplicate.json` |
| Fixture folder | `tests/contract/{domain}/{function}/Fixtures/` | `tests/contract/otmb/order-release-split/Fixtures/` |

---

<a id="p4-file-names-sftpblob"></a>
## 9. P4 file names (SFTP/Blob)

> P4 is the **one pattern where `integrationId` appears in the runtime file name** — required for traceability with partner EDI/SFTP systems that use the sender ID for routing and acknowledgement.

Format: `{IntId}_{Entity}_{DIR}_{YYYYMMDD}_{HHmmss}_{Seq}.{ext}`

Example: `INT0015_PAYMENT_OUT_20260618_143022_001.pgp`

---

<a id="git-flow-branching-strategy"></a>
## 10. Git flow branching strategy

### Branch model

```
main          ← production-ready; protected; requires 2 approvals + change manager
  ↑ PR + approval gate (test sign-off)
release/wave-{n}-{YYYY-QN}   ← wave release candidate; deploys to test with approval gate
  ↑ PR + 1 approval
develop       ← integration branch; auto-deploys to dev; protected (no direct push)
  ↑ PR + 1 approval
feature/{description}         ← per-integration or per-feature work (branched from develop)
hotfix/{description}          ← emergency prod fix (branched from main, merged to main + develop)
docs/{description}            ← documentation-only changes (skips artefact lint + deploy stages)
config/{description}          ← config-only changes (skips artefact lint + deploy stages)
```

### Branch naming

No integration IDs in branch names. Use domain/function identity for integration work; use area/description for engine and other work.

| Branch | Pattern | Example |
|--------|---------|---------|
| Feature (new integration) | `feature/{domain}-{function}` | `feature/otmb-order-release-split` |
| Feature (engine/factory) | `feature/{area}-{description}` | `feature/engine-liquid-key-fix` |
| Feature (pattern scaffold) | `feature/pattern-{patternId}` | `feature/pattern-p3-sb-outbound` |
| Release | `release/wave-{n}-{YYYY-QN}` | `release/wave-1-2026-Q3` |
| Hotfix | `hotfix/{description}` | `hotfix/dlq-param-strip` |
| Documentation | `docs/{description}` | `docs/artefact-naming-conventions` |
| Config change | `config/{description}` | `config/otmb-app-settings` |

> Microsoft GitHub Flow / Azure DevOps guidance: branch names should describe the **work**, not internal tracking references. Integration IDs are tracking artefacts (for logs and tickets), not part of the public identity of the work.

### Environment → branch mapping

| Environment | Branch | Deploy trigger | Approval required |
|-------------|--------|---------------|-------------------|
| `sbx` | any `feature/*` | On push (isolated per-developer sbx) | None — developer self-service |
| `dev` | `develop` | PR merge | 1 peer reviewer |
| `test` | `release/*` | PR merge | 2 reviewers (tech lead + integration lead) |
| `prd` | `main` | PR merge | 2 reviewers + **change manager approval** + change ticket |

> **Sandbox deploy model:** push-deploy to `sbx` is safe only when each developer has an **isolated sandbox environment**. If `sbx` is a shared environment, switch to a manual deploy trigger to prevent concurrent feature branches overwriting each other.

`docs/*` and `config/*` branches skip the artefact lint and deploy stages in CI — they run only documentation validation and config schema checks.

### Branch protection rules

| Branch | Required reviewers | Dismiss stale reviews | Block direct push | Status checks |
|--------|-------------------|-----------------------|-------------------|---------------|
| `main` | 2 (incl. change manager) | Yes | Yes | All CI + security scan |
| `release/*` | 2 | Yes | Yes | All CI |
| `develop` | 1 | No | Yes | Build + unit tests |
| `feature/*` | 0 | No | No | Build |
| `docs/*` | 1 | No | Yes | Docs lint |
| `config/*` | 1 | No | Yes | Config schema validation |

### Deployment pipeline per environment

```
feature/* → sbx:   CI build → factory unit tests → artefact lint → deploy to sbx Logic App
develop   → dev:   CI build → all tests → deploy to dev → smoke test
release/* → test:  CI build → all tests → deploy to test (MANUAL APPROVAL gate) → integration tests
main      → prd:   CI build → deploy to prd (CHANGE TICKET + APPROVAL gate) → health check

docs/*    → (no deploy) → docs lint → PR to develop
config/*  → (no deploy) → config schema check → PR to develop
```

### Factory repo vs artefacts repo

Both repos follow the same Git Flow model. The factory repo (`Vestacy Non ERP`) generates artefacts committed to the artefacts repo (`vestacy-ais`). Artefact commits are made on the same feature branch name in both repos, keeping them in sync by branch.

---

<a id="worked-example"></a>
## 11. Worked example

Identity: domain `otmb`, function `order-release-split`, source `otm`, target `d365`, direction `inbound`, version `v1`

### Repo layout (`vestacy-ais`)

```
workflows/wave-1/otmb/order-release-split/
  workflow.json
  connections.json
  otm-d365-order-release-split.liquid
  otmb-order-release-split-request.schema.json
  otmb-release-canonical.schema.json

apim/apis/otmb/
  otmb-order-release-split-v1-api.wsdl
  otmb-order-release-split-v1-policy.xml

apim/products/otmb/
  otmb-product.json

apim/fragments/otmb/
  otmb-inbound-common.xml

apim/subscriptions/otmb/
  otmb-subscription.json

apim/named-values/shared/
  shared-named-values.json

apim/named-values/otmb/
  otmb-named-values.json

config/otmb/order-release-split/
  config.json
  keyvault.json

infrastructure/modules/otmb/
  main.tf
  servicebus.tf
  variables.tf

tests/contract/otmb/order-release-split/Fixtures/
  otmb-order-release-split-happy.json
  otmb-order-release-split-failure.json
  otmb-order-release-split-duplicate.json
```

### Name reference

| Artefact | Name |
|----------|------|
| Logic App workflow name (runtime) | `wf-otm-d365-order-release-split-inbound` |
| Liquid file | `otm-d365-order-release-split.liquid` |
| Request schema | `otmb-order-release-split-request.schema.json` |
| Canonical schema | `otmb-release-canonical.schema.json` |
| APIM API spec | `otmb-order-release-split-v1-api.wsdl` |
| APIM Policy | `otmb-order-release-split-v1-policy.xml` |
| APIM base path | `/otmb/v1/order-release-split` |
| Service Bus queue | `otm-d365-order-release-split-inbound` |
| App Config prefix | `ais:otmb:order-release-split:*` |
| Feature branch | `feature/otmb-order-release-split` |

---

<a id="anti-patterns-never-do-this"></a>
## 12. Anti-patterns (never do this)

| ❌ Wrong | ✅ Correct | Reason |
|---------|----------|--------|
| `int0033_7-otm-api.yaml` | `otmb-order-release-split-v1-api.yaml` | IntId in artefact name |
| `int0007-qualityone-to-d365.liquid` | `qualityone-d365-<function>.liquid` | IntId in artefact name |
| `INT0007-request.schema.json` | `{domain}-{function}-request.schema.json` | IntId in schema name |
| `config/INT0007/` | `config/{domain}/{function}/` | IntId in folder |
| Named Value `int0033_7-target-baseUrl` | App Setting `ais:otmb:order-release-split:target:baseUrl` | IntId in APIM Named Value |
| Workflow name `int0033_7-otm-inbound` | `wf-otm-d365-order-release-split-inbound` | Underscore + IntId (runtime rejects) |
| Secret name `secret-otm.basic.auth-prd` | `ais-otm-basic-auth` | Dots, `secret-` prefix, env suffix, and missing source-target prefix |
| Branch `feature/INT0033-otmb` | `feature/otmb-order-release-split` | IntId in branch name |
| Env suffix `prod` | `prd` | Non-standard — use 3-char `prd` consistently |

---

<a id="related-documents"></a>
## 13. Related documents

| Document | Purpose |
|----------|---------|
| `doc/standards.md §9` | Canonical authority for naming principles |
| `doc/apim-conventions.md` | Full APIM naming and policy detail |
