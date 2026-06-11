---
name: pr-review
description: >
  Perform an AI-assisted pull-request review that combines a security
  perspective (OWASP Top 10, secrets, PII, PHI) and an Infrastructure-as-Code
  compliance perspective (CMS ARS 5.1 / NIST SP 800-53 Rev 5) into a single
  unified review of a set of changes. Works well as a second-opinion layer on
  top of pre-commit or per-commit checks, assessing the full set of changes
  together at the PR/branch scope. Produces a readable findings report with
  per-file, per-severity findings and inline-style suggested fixes. Use this
  skill whenever a developer asks for a PR or diff review, wants an AI-assisted
  review alongside or instead of other reviewers, or needs combined security and
  IaC-compliance feedback on a change.
---

# PR Review Skill

A composed, multi-perspective review of an entire set of changes (a pull-request
diff, a branch, or a range of commits) against the base it is being compared to.
This skill works well **after** per-commit checks have run, providing a second
layer of review at the PR scope where the full set of changes can be assessed
together — some findings only emerge when changes are composed (e.g., a refactor
in one commit plus a new caller in another reveals an access-control gap).

It composes **two perspectives** — security and IaC compliance — and presents
its findings directly to you as a readable report with inline-style suggested
fixes. It does not post to GitHub or any other system; the human reviewer and
author decide what to act on.

---

## When to Use

- A developer asks for a review of a PR, a branch, or a diff.
- You want an AI-assisted review in addition to (or instead of) another reviewer.
- A change touches application code, infrastructure code, or both, and you want
  combined security + compliance feedback before merge.

This is an **advisory** review layer. It surfaces findings and suggestions; it
does not block. Per-commit hooks (if present) are the blocking layer; PR review
is the second-opinion layer that looks at the whole change set.

---

## Overview of the Process

1. **Collect the changes** — the full diff between the base ref and HEAD.
2. **Identify which perspectives apply** — security always; compliance if IaC
   files are present.
3. **Apply both perspectives' checks** — security + compliance, unified findings.
4. **Load targeted context** — pull in the minimum files needed for accuracy.
5. **Present a readable report** — findings grouped by file, each with severity,
   perspective, a clear explanation, and an inline-style suggested fix.

---

## Step 1 — Collect the Changes

Determine the **base ref** you are comparing against from context, or ask the
user if it is ambiguous (common choices: the PR's base branch, `main`, or an
explicit commit range).

```bash
git diff "<base-ref>" HEAD --unified=5      # full content
git diff "<base-ref>" HEAD --name-only      # list of changed paths
```

If the diff is empty, report that there is nothing to review and stop.

---

## Step 2 — Identify Applicable Perspectives

| Perspective | When it applies |
|---|---|
| **Security** | Always |
| **IaC Compliance** | When at least one IaC file is in the diff |

**IaC file detection** uses these patterns: `.tf`, `.tfvars`, `.tf.json`,
`.bicep`, `.bicepparam`, `.hcl`, `*.template.json/yaml`, `Pulumi.yaml`,
`Chart.yaml`, `values.yaml`, `cdk.json`, and any YAML containing both
`apiVersion:` and `kind:`.

If no IaC files are present, skip the compliance perspective entirely and note
this at the top of the report.

---

## Step 3 — Apply Both Perspectives

If the **code-security** and **iac-compliance** skills are available alongside
this one, apply their full check lists. This skill is self-contained: the
essential checks for each perspective are summarized below so the review is
useful even when those sibling skills are not loaded.

The two perspectives are **complementary, not overlapping** — secrets in a
Terraform file are a security finding (Critical, secrets); a Terraform RDS
instance without encryption-at-rest is a compliance finding (High, SC-12/SC-28).
The same line can produce both kinds of findings, which is fine — report one
finding for each.

### Security perspective (apply the code-security skill's checks)

Review the changed code for:

- **Secrets / credentials** — hardcoded API keys, access keys, tokens,
  passwords, private keys, connection strings checked into source.
- **PII** — personally identifiable information (names, emails, SSNs, addresses,
  phone numbers) in source, logs, fixtures, or config.
- **PHI** — protected health information, treated as Critical when real.
- **OWASP Top 10 (2021)** categories, especially:
  - A01 Broken Access Control
  - A02 Cryptographic Failures
  - A03 Injection (SQL/command/template injection, etc.)
  - A05 Security Misconfiguration
  - A07 Identification and Authentication Failures
  - A08 Software and Data Integrity Failures
- **General hardening** — missing input validation, unsafe deserialization,
  insecure defaults, error handling that leaks sensitive detail.

Cite the OWASP category where applicable (e.g., "OWASP A03:2021 – Injection").

### IaC compliance perspective (apply the iac-compliance skill's checks)

When IaC files are present, review against the NIST SP 800-53 Rev 5 control
families (with their corresponding CMS ARS 5.1 controls):

- **AC (Access Control)** — IAM wildcard / admin policies, overly broad
  permissions, missing conditions.
- **AU (Audit & Accountability)** — CloudTrail off, log retention unset,
  insufficient audit logging.
- **CM (Configuration Management)** — missing required tags, unpinned module
  versions, images tagged `latest`, missing Name/description.
- **CP (Contingency Planning)** — deletion protection off in prod, missing
  backups.
- **IA (Identification & Authentication)** — hardcoded passwords, weak auth
  configuration.
- **RA (Risk Assessment)** — GuardDuty absent, missing vulnerability scanning.
- **SC (System & Communications Protection)** — SSH/RDP open to `0.0.0.0/0`,
  DB ports open to the internet, encryption at rest/in transit off (SC-12 /
  SC-28), public S3 / public RDS, missing WAF on public endpoints, KMS default
  key, missing VPC endpoints.
- **SI (System & Information Integrity)** — monitoring gaps (SI-4), deprecated
  runtimes, missing X-Ray.

Cite the NIST 800-53 Rev 5 control ID, and the CMS ARS 5.1 control ID where they
differ (e.g., "NIST AC-3, CMS ARS AC-3(HIGH)").

---

## Step 4 — Load Targeted Context

Pull in only the minimum files needed for an accurate assessment, with a ceiling
of **≤ 15 additional files per perspective** (up to 30 total). Do **not** load:

- The full source tree
- Lock files, generated artifacts, vendor directories
- Test fixtures unless they directly inform a finding

If you would exceed the ceiling, note the limitation in the report and review
what you have.

---

## Step 5 — Severity Ladders

Maintain a consistent severity ladder: **Critical / High / Medium / Low**.

**Security findings:**
| Severity | Examples |
|---|---|
| Critical | Hardcoded secrets/credentials; real PHI; RCE; auth bypass |
| High     | Real PII; significant injection risk; broken access control; crypto failure |
| Medium   | Injection with partial mitigation; suspicious PII; missing input validation on internal surface |
| Low      | Minor hygiene; placeholder-like PII; informational hardening |

**Compliance findings:**
| Severity | Examples |
|---|---|
| Critical | SSH/RDP open to 0.0.0.0/0; IAM wildcard with no conditions; unencrypted PHI/PII stores; S3 public access blocks disabled; publicly accessible RDS |
| High     | IAM admin policies; DB ports open to internet; encryption at rest off; CloudTrail off; hardcoded passwords; deprecated runtimes; deletion protection off in prod |
| Medium   | Missing VPC endpoints; WAF absent on public endpoints; 2+ required tags missing; KMS default key; log retention unset; GuardDuty absent |
| Low      | 1 required tag missing; image tagged `latest`; X-Ray off; module not version-pinned; missing Name/description |

When in doubt between two severities, choose the **lower** one and note the
uncertainty in the description.

---

## Step 6 — Present the Report

Present findings directly to the user as a readable report, with both
perspectives' findings interleaved and grouped by file. Each finding carries a
severity, a perspective, a clear explanation (with the OWASP category or NIST/CMS
control reference), and an inline-style suggested fix.

```
## PR Review Report
**Scope:** Diff against <base-ref> (N files, M lines added, P lines removed)
**Perspectives applied:** Security; Compliance (or: Security only — no IaC files)
**Files reviewed:** <list>
**Context files loaded:** <list, or "None">

---

### Findings by file

#### `path/to/file.py`

- 🔴 **CRITICAL** | **security** | Secrets — Hardcoded API key on line 42
  AWS access key checked into source. Rotate immediately; move to env var.

  ```suggestion
  api_key = os.environ["AWS_ACCESS_KEY_ID"]
  ```

- 🟡 **MEDIUM** | **security** | A03 Injection — Possible SQL injection on line 87
  `f"SELECT * FROM users WHERE id = {user_id}"` — use a parameterized query.

#### `infra/rds.tf`

- 🟠 **HIGH** | **compliance** | SC-12/SC-28 — RDS without encryption at rest (line 12)
  Set `storage_encrypted = true` and specify a `kms_key_id`. NIST SC-12,
  SC-28; CMS ARS 5.1 corresponding controls.

  ```suggestion
    storage_encrypted = true
    kms_key_id        = aws_kms_key.rds.arn
  ```

#### `infra/monitoring.tf`

- 🟡 **MEDIUM** | **compliance** | SI-4 — GuardDuty not enabled
  Significant infrastructure is being added without an
  `aws_guardduty_detector` resource. NIST SI-4 (System Monitoring) recommends
  GuardDuty for threat detection across EC2, S3, and EKS workloads. Reference
  fix (new resource, not an in-place edit):

  ```hcl
  resource "aws_guardduty_detector" "main" {
    enable = true
  }
  ```

---

### Summary
| Severity | Security | Compliance | Total |
|---|---|---|---|
| Critical | 1 | 0 | 1 |
| High     | 0 | 1 | 1 |
| Medium   | 1 | 1 | 2 |
| Low      | 0 | 0 | 0 |

**Overall:** Advisory review — findings above are for the author and human
reviewer to weigh. No finding here blocks merge on its own.
```

---

## Suggested-Fix Guidance

Distinguish two kinds of suggestion, and choose carefully:

- **Applicable fix** — the change can replace the exact line(s) at the cited
  location as-is. Present it in a `` ```suggestion `` block. Use this **only**
  when you are confident the suggestion correctly replaces those line(s); a
  reader may apply it directly.
- **Reference fix** — the fix is a new resource, a new file, or a refactor that
  spans multiple locations and cannot be dropped in at one line. Present it as
  an illustrative code block in the appropriate language fence (`hcl`, `python`,
  `yaml`, etc.) and label it as a reference, not an in-place edit. Never present
  code as an applicable fix when it does not belong at that exact line.

---

## Reviewer Notes

- **The change set is a snapshot, not a stream.** A PR may contain dozens of
  commits across days. Look at the diff as a whole; some findings only emerge
  when changes are composed.
- **Do not duplicate findings unnecessarily.** If the same issue appears on
  several lines (e.g., five resources missing the same tag), report one finding
  per resource — not one per line within each resource. A reader should not see
  twenty findings for one root cause.
- **Severity is your responsibility.** Apply the severity rubric consistently.
  When uncertain between two severities, choose the lower and say so.
- **Findings only — no praise, no nitpicks.** Keep the report focused on
  security and compliance findings. If something fits neither perspective, it is
  out of scope — omit it rather than adding noise.

---

## Common Rationalizations (do not fall for these)

- *"A per-commit hook already ran, so this is clean."* The PR scope sees the
  composed change set; cross-commit issues only surface here. Review the whole
  diff anyway.
- *"This secret/PII looks fake, so I'll skip it."* Flag it as Low (placeholder-
  like) rather than dropping it; let the human confirm.
- *"This Terraform is just a small change, compliance can't apply."* A single
  resource can still open SSH to the world or disable encryption. Run the
  compliance checks whenever any IaC file is in the diff.
- *"The fix is obvious, I'll mark it applicable."* Only mark a fix applicable
  when it replaces the cited line(s) exactly. Structural fixes are references.

---

## Verification

Before finishing, confirm:

- Every changed file in the diff was considered under the security perspective.
- The compliance perspective ran if and only if at least one IaC file is present
  (and its absence/presence is noted at the top of the report).
- Every finding has a severity, a perspective (`security` or `compliance`), a
  clear explanation, and — for security — an OWASP reference where applicable,
  or — for compliance — the NIST 800-53 Rev 5 control ID (plus CMS ARS 5.1 where
  it differs).
- Each suggested fix is correctly classified as an applicable in-place fix or a
  reference fix.
- No duplicate findings for a single root cause; no praise or out-of-scope
  nitpicks.
- The context-file ceiling (≤ 15 per perspective) was respected, with any
  limitation noted.