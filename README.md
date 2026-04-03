# Dynatrace IAM Policies — Reference Guide

> **Source:** [IAM policy reference](https://docs.dynatrace.com/docs/manage/identity-access-management/permission-management/manage-user-permissions-policies/advanced/iam-policystatements) · [Policy syntax & examples](https://docs.dynatrace.com/docs/manage/identity-access-management/permission-management/manage-user-permissions-policies/iam-policystatement-syntax) · [Manage IAM policies](https://docs.dynatrace.com/docs/manage/identity-access-management/permission-management/manage-user-permissions-policies/iam-policy-mgt)
> **Audience:** Customer Success Engineers, Services Consultants, Solution Architects

---

## What are IAM Policies?

Dynatrace IAM policies replace the legacy role-based access control system with a fine-grained, attribute-based model. Instead of assigning coarse platform roles, you write **policy statements** that explicitly allow or deny specific actions on specific services — optionally scoped by conditions like schema ID, management zone, or app ID.

Policies are assigned to **users**, **user groups**, or **service accounts**, and can be bound at the **account level** (across all environments) or at the **environment level** (scoped to one environment).

---

## Policy Statement Syntax

```
ALLOW <service:resource:action> [, <service:resource:action>]
  [WHERE <service:attribute> <operator> 'value'
   [AND <service:attribute> <operator> 'value']];
```

```
DENY <service:resource:action>;
```

### Key Rules

- Every statement ends with a semicolon `;`
- Multiple permissions can be comma-separated in a single statement
- All `WHERE` conditions are combined with `AND` (all must be true)
- To express `OR` logic, write **separate statements**
- String values can use single `'value'` or double `"value"` quotes

### Full Example

```
ALLOW settings:objects:read, settings:objects:write
  WHERE settings:schemaId = "builtin:container.monitoring-rule";

ALLOW environment:roles:viewer
  WHERE environment:management-zone = "Production";

DENY storage:logs:read;
```

---

## Policy Evaluation Order

When a request is evaluated, Dynatrace checks statements in this strict order:

| Priority | Type | Result |
|----------|------|--------|
| 1 | Unconditional `DENY` | Access blocked immediately |
| 2 | Conditional `DENY` | Access blocked if condition matches |
| 3 | Unconditional `ALLOW` | Access granted |
| 4 | Conditional `ALLOW` | Access granted if condition matches |
| 5 | No match | **Implicit DENY** — access denied |

> **Bottom line:** `DENY` always wins regardless of specificity. If no statement matches at all, access is denied by default.

---

## Account-Level vs Environment-Level Policies

| Aspect | Account-Level | Environment-Level |
|--------|--------------|-------------------|
| Scope | All environments under the account | Single environment only |
| Use case | Company-wide baseline access | Per-team or per-project scoping |
| Binding | Bound to account | Bound to a specific environment ID |
| Inheritance | Inherited by all environments | Does not propagate upward |
| Conditions | Cannot reference environment-specific attributes | Can use `environment:management-zone` etc. |

> **Gotcha:** You **cannot mix account-level and environment-level conditions in a single policy statement**. If a policy is bound at the environment level, conditions must be environment-scoped and vice versa.

---

## Condition Operators

| Operator | Description | Example |
|----------|-------------|---------|
| `=` | Exact match | `settings:schemaId = "builtin:alerting.profile"` |
| `!=` | Not equal | `automation:workflow-type != "SIMPLE"` |
| `IN` | Value in a list | `settings:schemaId IN ("schema1", "schema2")` |
| `NOT IN` | Value not in a list | `settings:schemaId NOT IN ("builtin:secret.schema")` |
| `startsWith` | Prefix match | `extensions:extension-name startsWith "custom."` |
| `NOT startsWith` | Does not start with prefix | `extensions:extension-name NOT startsWith "builtin."` |

---

## Services Reference

### `settings`
Manages configuration objects and schema definitions.

| Permission | Description |
|------------|-------------|
| `settings:objects:read` | Read settings objects |
| `settings:objects:write` | Create/update settings objects |
| `settings:schemas:read` | Read schema definitions |

**Condition attributes:**
- `settings:schemaId` — matches a specific schema identifier (e.g. `builtin:alerting.profile`)

> **Gotcha:** `settings:objects:read` and `settings:schemas:read` are **separate permissions**. If a user needs to view settings in the UI or via API, they typically need both. Granting only one will silently break parts of the experience.

---

### `storage`
Controls access to Grail data — logs, events, spans, metrics, business events.

| Permission | Description |
|------------|-------------|
| `storage:logs:read` | Read log data |
| `storage:events:read` | Read event data |
| `storage:spans:read` | Read trace/span data |
| `storage:metrics:read` | Read metric data |
| `storage:bizevents:read` | Read business events |
| `storage:buckets:read` | Read bucket configuration |
| `storage:buckets:write` | Write bucket configuration |
| `storage:filter-segments:read` | Read filter segments |
| `storage:filter-segments:write` | Write filter segments |

**Condition attributes:**
- `storage:table-name` — scope access to a specific Grail table

> **Gotcha:** Conditional `DENY` on Grail table permissions is **not supported**. If you write `DENY storage:logs:read WHERE storage:table-name = "logs"`, it is treated as an **unconditional DENY** — blocking all log access, not just that table.

---

### `environment`
Legacy-style role grants scoped by management zone.

| Permission | Description |
|------------|-------------|
| `environment:roles:viewer` | View all entities and data |
| `environment:roles:manage-settings` | Manage environment settings |
| `environment:roles:agent-install` | Install OneAgent |
| `environment:roles:configure-request-capture-data` | Configure request capture |
| `environment:roles:manage-security-problems` | Manage security findings |
| `environment:roles:view-security-problems` | View security findings |
| `environment:roles:logviewer` | View logs in classic Log Viewer |
| `environment:roles:replay-sessions` | Replay session recordings |

**Condition attributes:**
- `environment:management-zone` — scope the role to a specific management zone display name

> **Gotcha:** `environment:roles:agent-install` and `environment:roles:configure-request-capture-data` **cannot be conditioned**. They must be granted unconditionally or not at all.

---

### `automation`
Workflow and scheduling engine permissions.

| Permission | Description |
|------------|-------------|
| `automation:workflows:read` | Read workflows |
| `automation:workflows:write` | Create/update workflows |
| `automation:workflows:run` | Execute workflows |
| `automation:workflows:admin` | Admin access to all workflows |
| `automation:rules:read` | Read scheduling rules |
| `automation:rules:write` | Write scheduling rules |
| `automation:calendars:read` | Read business calendars |
| `automation:calendars:write` | Write business calendars |

**Condition attributes:**
- `automation:workflow-type` — `SIMPLE` or `STANDARD`

---

### `app-engine`
Controls Dynatrace App installation and execution.

| Permission | Description |
|------------|-------------|
| `app-engine:apps:install` | Install and update apps |
| `app-engine:apps:run` | Run apps, access the Launcher |
| `app-engine:apps:delete` | Uninstall apps |
| `app-engine:functions:run` | Use the function executor |
| `app-engine:edge-connects:read` | Read EdgeConnect configurations |
| `app-engine:edge-connects:write` | Write EdgeConnect configurations |
| `app-engine:edge-connects:delete` | Delete EdgeConnects |
| `app-engine:certificates:create` | Create short-lived app certificates |

**Condition attributes:**
- `shared:app-id` — scope to a specific app identifier
- `app-engine:app-installer` — match by the user ID who installed the app

---

### `app-settings`
Per-app settings objects (separate from platform `settings`).

| Permission | Description |
|------------|-------------|
| `app-settings:objects:read` | Read app settings objects |
| `app-settings:objects:write` | Write app settings objects |
| `app-settings:objects:admin` | Admin-mode access to app settings |

**Condition attributes:**
- `settings:schemaId` — match app-specific schema
- `shared:app-id` — scope to a specific app

---

### `document`
Notebooks, dashboards, and shared document management.

| Permission | Description |
|------------|-------------|
| `document:documents:read` | Read documents |
| `document:documents:write` | Create/update documents |
| `document:documents:delete` | Delete documents |
| `document:documents:restore` | Restore deleted documents |
| `document:shares:read` | Read document shares |
| `document:shares:write` | Manage document shares |
| `document:shares:delete` | Delete document shares |

---

### `davis-copilot`
Controls access to AI-assisted features.

| Permission | Description |
|------------|-------------|
| `davis-copilot:conversations:execute` | Chat with Davis Copilot |
| `davis-copilot:nl2dql:execute` | Convert natural language to DQL |
| `davis-copilot:dql2nl:execute` | Summarize DQL queries in plain language |
| `davis-copilot:document-search:execute` | AI document search |

---

### `data-acquisition`
Ingest permissions — primarily for service accounts and tokens.

| Permission | Description |
|------------|-------------|
| `data-acquisition:logs:ingest` | Ingest log data |
| `data-acquisition:metrics:ingest` | Ingest metric data |
| `data-acquisition:events:ingest` | Ingest event data |

---

### `davis`
Davis AI analyzer access.

| Permission | Description |
|------------|-------------|
| `davis:analyzers:read` | View Davis analyzers |
| `davis:analyzers:execute` | Run Davis analyzers |

---

### `ai`

| Permission | Description |
|------------|-------------|
| `ai:operator:execute` | Interact with the AI conversational interface |

---

### `extensions`
EF2 extension definition and monitoring configuration.

| Permission | Description |
|------------|-------------|
| `extensions:definitions:read` | Read extension definitions |
| `extensions:definitions:write` | Write extension definitions |
| `extensions:monitoringconfigurations:read` | Read monitoring configs |
| `extensions:monitoringconfigurations:write` | Write monitoring configs |

**Condition attributes:**
- `extensions:extension-name` — match by extension name
- `extensions:host` — scope to a specific host
- `extensions:host-group` — scope to a host group
- `extensions:ag-group` — scope to an ActiveGate group
- `extensions:management-zone` — scope to a management zone

---

### `email`

| Permission | Description |
|------------|-------------|
| `email:emails:send` | Send emails via the Dynatrace API |

---

### `business-analytics`

| Permission | Description |
|------------|-------------|
| `business-analytics:business-flows:read` | Read business flows |
| `business-analytics:business-flows:write` | Write business flows |

---

## Policy Binding

Policies are attached to an **identity** at a specific **scope**:

| Identity Type | When to Use |
|---------------|-------------|
| **User group** | Standard access control for human users — bind policies to groups, add users to groups |
| **Individual user** | Exception-based access, rarely recommended |
| **Service account** | Automation, integrations, API tokens — always prefer service accounts over user credentials for programmatic access |

**Binding scope:**
- **Account** — policy applies across all environments
- **Environment** — policy applies only to one environment (specify environment ID)

---

## Common Policy Examples

### Read-only access to a specific Settings schema
```
ALLOW settings:objects:read, settings:schemas:read
  WHERE settings:schemaId = "builtin:alerting.profile";
```

### Full access to automation workflows, standard type only
```
ALLOW automation:workflows:read, automation:workflows:write, automation:workflows:run
  WHERE automation:workflow-type = "STANDARD";
```

### Viewer role scoped to a management zone
```
ALLOW environment:roles:viewer
  WHERE environment:management-zone = "Production";
```

### Read access to multiple Grail tables
```
ALLOW storage:logs:read, storage:events:read, storage:spans:read;
```

### Service account — log ingestion only
```
ALLOW data-acquisition:logs:ingest;
```

### App access scoped to a specific app
```
ALLOW app-engine:apps:run
  WHERE shared:app-id = "dynatrace.site-reliability-guardian";
```

### OR logic — two separate statements
```
ALLOW settings:objects:read
  WHERE settings:schemaId = "builtin:alerting.profile";

ALLOW settings:objects:read
  WHERE settings:schemaId = "builtin:alerting.maintenance-window";
```

---

## Gotchas & Common Pitfalls

### 1. DENY always wins — no exceptions
A `DENY` statement overrides **all** `ALLOW` statements, regardless of how specific the `ALLOW` is. If a user is in two groups and one group has a `DENY` statement that matches, access is blocked even if the other group has an explicit `ALLOW`.

**Avoid broad `DENY` statements unless you are certain of the blast radius.**

---

### 2. Conditional DENY on Grail tables is silently unconditional
Writing this:
```
DENY storage:logs:read WHERE storage:table-name = "logs";
```
Does **not** scope the deny to that table. Dynatrace treats conditional `DENY` on Grail permissions as **unconditional**, blocking all log reads entirely.

**Rule:** Do not use `DENY` with `storage:*` permissions unless you mean to deny access to that data type globally.

---

### 3. `settings:objects:read` ≠ `settings:schemas:read`
These are two separate permission checks:
- `settings:schemas:read` — lets the user see the schema definition (what fields exist)
- `settings:objects:read` — lets the user read the actual configured values

Most UI interactions require **both**. Granting only one causes confusing partial failures — the page may load but data won't appear, or vice versa.

---

### 4. No match = implicit DENY
There is no default-allow fallback. If no policy statement matches a user's request, access is **silently denied**. This is especially surprising when migrating from legacy roles, which often had implicit broad access.

**When troubleshooting access issues, check for missing statements before looking for conflicting `DENY`s.**

---

### 5. Cannot mix account-level and environment-level conditions
You cannot write a single policy statement that conditions on both account-scope and environment-scope attributes. Policies must be consistent with the level at which they are bound.

---

### 6. Environment roles cannot be conditioned by schema or resource
`environment:roles:*` permissions only support `environment:management-zone` as a condition. You cannot scope an environment role by schema, app, or any other attribute.

---

### 7. Some environment roles cannot be conditioned at all
The following **must be granted unconditionally**:
- `environment:roles:agent-install`
- `environment:roles:configure-request-capture-data`

Adding a `WHERE` clause to these has no effect or may cause unexpected behavior.

---

### 8. OR logic requires multiple statements
The `WHERE` clause only supports `AND` between conditions. There is no `OR` keyword. To express OR, write a separate `ALLOW` statement for each case. This increases statement count — be aware of the **100 statement limit per policy**.

---

### 9. Template variables are not supported in policy conditions
You **cannot** use dynamic placeholders like `{environment}` or `{user}` in condition values. All values must be hardcoded strings. This means environment-specific policies must be duplicated or managed separately per environment.

---

### 10. Legacy roles have implicit permissions IAM cannot replicate 1:1
Some platform roles (like "Administrator") granted access to features that don't map cleanly to individual IAM permissions — especially for older UI pages or classic features. When transitioning users from legacy roles to IAM policies, audit access requirements explicitly rather than assuming a 1:1 mapping exists.

---

### 11. Management zone conditions use the display name
The `environment:management-zone` condition uses the **management zone display name** (e.g. `"Production"`), not the numeric ID. Using the numeric ID will silently fail — the condition won't match and access will be denied.

```
ALLOW environment:roles:viewer
  WHERE environment:management-zone = "Production";
```

**To find the display name:** Settings → Management Zones → use the name exactly as shown, including capitalization.

---

### 12. Policy limit: 100 statements per policy
A single policy document can contain up to 100 statements. If OR logic across many schemas or zones is needed, consider using `IN` lists rather than separate statements where possible.

```
ALLOW settings:objects:read
  WHERE settings:schemaId IN (
    "builtin:alerting.profile",
    "builtin:alerting.maintenance-window"
  );
```

---

## Quick Reference — Evaluation Cheat Sheet

```
Request comes in
       │
       ▼
Unconditional DENY? ──► YES → DENY (blocked)
       │ NO
       ▼
Conditional DENY matches? ──► YES → DENY (blocked)
       │ NO
       ▼
Unconditional ALLOW? ──► YES → ALLOW (granted)
       │ NO
       ▼
Conditional ALLOW matches? ──► YES → ALLOW (granted)
       │ NO
       ▼
No match → Implicit DENY (blocked, no error shown)
```

---

> **Disclaimer:** This guide is AI-assisted and intended for reference and learning purposes only. It may contain inaccuracies, incomplete information, or content that has drifted from current product behavior — always consult the [official Dynatrace documentation](https://docs.dynatrace.com) for authoritative guidance. This is not an official Dynatrace resource.
