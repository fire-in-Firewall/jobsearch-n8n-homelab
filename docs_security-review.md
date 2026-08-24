# Security Research Review — Job Automation Lab

## Scope

This review is based on the current n8n workflow exports:

- `jobs.finder`
- `jobs.backfill_extractor`
- `jobs.canonical_loader`
- `jobs.fit_score`
- `jobs.orchestrator`
- `jobs.tailor_resume`
- `build_manager`
- `dictionary_maintenance`

The review focuses on:

- trust boundaries,
- credential management,
- host privileges,
- data privacy,
- AI prompt injection,
- SaaS/shared-responsibility risk,
- public artifact sharing,
- reproducibility security.

---

## Executive assessment

The system is appropriate for a **single-user homelab** but should not be presented as production-hardened.

The highest-risk issues in the current exports are:

1. real candidate profile content embedded in `jobs.canonical_loader`,
2. `build_manager` granting `anyone / reader` access to generated resumes,
3. `Execute Command` invoking a shell on the host,
4. recursive filesystem deletion/writes,
5. unvalidated runtime build-path construction,
6. debug bundles containing full prompts and LLM outputs,
7. potentially sensitive data persisting in n8n execution history,
8. hard-coded personal filesystem/resource identifiers.

---

## Findings

| ID | Severity | Finding |
|---|---|---|
| SEC-01 | Critical | Canonical candidate profile embedded in workflow |
| SEC-02 | Critical | Generated PDF shared to anyone with link |
| SEC-03 | High | Host shell execution in `build_manager` |
| SEC-04 | High | Runtime-derived build path used for filesystem and shell operations |
| SEC-05 | High | Personal absolute filesystem paths |
| SEC-06 | High | Debug bundler captures prompt/output content |
| SEC-07 | High | Sensitive data can persist in execution history |
| SEC-08 | Medium/High | Incomplete resume LLM schema validation |
| SEC-09 | Medium | Provider date configuration drift |
| SEC-10 | Medium | Resource IDs embedded in exported workflows |

---

## SEC-01 — Candidate profile in workflow source

`jobs.canonical_loader` contains the canonical candidate profile directly inside the Code node.

This includes:

- experience,
- certification,
- skills,
- tools,
- frameworks,
- capability descriptions,
- bullet-bank content.

### Risk

The workflow export becomes a data export.

### Recommendation

Public repository:

```text
jobs.canonical_loader.example.json
```

Private/local:

```text
jobs.canonical_loader.json
```

Or inject the candidate profile from a private Notion/database/configuration layer.

---

## SEC-02 — Public Drive sharing

`build_manager.Node5_share_link` creates:

```text
type = anyone
role = reader
```

### Risk

Every generated resume becomes publicly accessible through the file link.

### Recommendation

Default to:

```text
private
```

or:

```text
user / reader
domain / reader
```

Use `anyone / reader` only as an explicit opt-in mode.

---

## SEC-03 — Host shell execution

The builder runs:

```bash
cd {{$json.build_path}}
latexmk -pdf -interaction=nonstopmode -f main.tex
```

### Risk

A workflow with shell access has host-level code execution authority.

n8n explicitly categorizes risky host-capable nodes in its security audit. Run `n8n audit` and review the filesystem/risky-node report before deployment.

### Recommendation

Long-term:

```text
n8n
  ↓
build worker
```

rather than direct shell execution in the orchestration engine.

---

## SEC-04 — Build-path validation

The build path is assembled from:

```text
notionPageId
company
timestamp
```

The company string is normalized, but the page ID is not subject to an equivalent allow-list validation.

The path is then used for:

```text
recursive delete
directory creation
template copy
file writing
shell cwd
```

### Recommendation

Use:

```javascript
const ROOT = "/safe/build/root";
const safeId = validatePageId(pageId);
const candidate = path.resolve(ROOT, safeId, buildId);

if (!candidate.startsWith(ROOT + path.sep)) {
  throw new Error("Invalid build path");
}
```

---

## SEC-05 — Absolute personal paths

Current exports contain paths such as:

```text
${USER_HOME}/...
/Library/TeX/texbin
```

### Risks

- leaks local account information,
- prevents portability,
- increases repository fingerprinting,
- complicates reproduction.

### Recommendation

Use local configuration:

```text
RESUME_TEMPLATE_PATH
RESUME_BUILD_ROOT
LATEX_BIN_PATH
```

---

## SEC-06 — Debug bundle

`jobs.tailor_resume.Node10_debug_bundler` stores:

- job context,
- structured requirements,
- selected bullets,
- matched requirements,
- projected skills,
- LLM system prompt,
- LLM user prompt,
- LLM output.

### Risk

A debugging convenience becomes an unbounded sensitive-data copy.

### Recommendation

Support:

```text
DEBUG_MODE=true
```

with the secure default:

```text
DEBUG_MODE=false
```

When debug mode is enabled:

- redact candidate data,
- truncate prompts,
- limit retention,
- do not persist to Git.

---

## SEC-07 — Execution-data retention

The system passes candidate evidence and job data through many nodes.

Depending on n8n execution-data settings, these values may remain available in execution history.

### Recommendation

Define:

```text
successful execution retention
failed execution retention
binary-data retention
debug retention
```

and review backups separately.

---

## SEC-08 — Resume-output validation

The current parser explicitly checks:

```text
optimized_professional_bullets is an array
```

but does not perform equivalent cardinality/type validation for every output property.

The prompt requests:

```text
6 professional
3 training
```

The implementation should enforce that contract after the model responds.

Recommended checks:

```text
Array.isArray(professional)
Array.isArray(training)

professional.length === 6
training.length === 3

every item is string
every item <= 30 words
```

---

## SEC-09 — Configuration drift

The finder contains centralized configuration objects but some API parameters are hard-coded in the actual request node.

This creates:

```text
configuration says X
request executes Y
```

### Recommendation

All runtime parameters should be sourced from one configuration object.

---

## SEC-10 — Resource identifiers

Exports include:

- Notion database IDs,
- Google Drive folder ID,
- n8n workflow IDs,
- credential IDs.

These are not secrets by themselves, but they are unnecessary public metadata and make the export less portable.

### Recommendation

Replace with placeholders in public examples.

---

# Threat model

## Assets

- candidate profile,
- resume content,
- job-search history,
- generated resumes,
- API credentials,
- OAuth credentials,
- Notion data,
- Google Drive artifacts,
- local filesystem.

## Adversaries

- malicious job-description author,
- compromised API provider,
- compromised n8n account,
- leaked repository,
- local user/process with filesystem access,
- accidental data-sharing operator.

---

## Attack path: prompt injection

```text
malicious JD
    ↓
jobs.finder
    ↓
LLM extraction
    ↓
structured requirements
```

Control:

- treat JD as data,
- never allow JD instructions to override system prompt,
- validate output schema.

---

## Attack path: public resume disclosure

```text
generated PDF
    ↓
Google Drive upload
    ↓
anyone / reader
    ↓
public link
```

Control:

- change default permission,
- make sharing explicit,
- optionally use expiring permissions.

---

## Attack path: host command execution

```text
runtime input
    ↓
build_path
    ↓
shell command
    ↓
latexmk
    ↓
host
```

Control:

- validate build path,
- avoid shell interpolation,
- isolate build worker.

---

# Shared responsibility

| Layer | Owner | Security responsibility |
|---|---|---|
| Host OS | project owner | patching, disk encryption, access control |
| n8n | project owner | authentication, encryption key, workflow permissions, audit |
| Ollama | project owner | local binding, model updates, host access |
| Notion | shared | integration scope + Notion-side account security |
| Google Cloud | shared | OAuth configuration + account security |
| Google Drive | shared | folder/permission policy |
| RapidAPI providers | third party | provider security / availability |
| GitHub | shared | repository settings + secret scanning |
| Workflow code | project owner | input validation + safe execution |

---

# Remediation roadmap

## P0

- remove real canonical profile from public workflow,
- make Drive sharing private by default,
- sanitize filesystem paths,
- add build-path validation,
- remove debug bundles from normal operation,
- secret scan repository and Git history.

## P1

- schema-validate every LLM response,
- move build execution into isolated worker,
- implement execution-data retention,
- introduce explicit prompt-injection controls,
- parameterize API date filters and all resource IDs.

## P2

- CI SAST,
- dependency scanning,
- regression fixtures,
- source-controlled environment promotion,
- separate operational datastore.

---

# Security verification checklist

Before publishing:

```text
[ ] gitleaks / secret scanner
[ ] no candidate profile in exported JSON
[ ] no Drive folder ID
[ ] no personal filesystem paths
[ ] no credential IDs required for public reproduction
[ ] Drive sharing default is not `anyone`
[ ] `build_path` cannot escape fixed root
[ ] LLM output schema is enforced
[ ] debug mode is opt-in
[ ] n8n execution retention is defined
[ ] `n8n audit` reviewed
```
