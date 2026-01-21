# Veatures: the security risk hiding in “legitimate” features

Most security programs hunt for bugs. Fewer hunt for Veatures—features that behave exactly as designed, are documented, and still open exploitable paths. In fast-moving OSS and third-party ecosystems, Veatures pass code review, survive testing, and quietly ship to production. This guide explains what Veatures are, how to detect them reliably, how to secure them, and how to automate detection in pipelines with LLMs.

## Table of contents
- [What is a Veature?](#what-is-a-veature)
- [Why traditional tools miss Veatures](#why-traditional-tools-miss-veatures)
- [A practical way to detect Veatures](#a-practical-way-to-detect-veatures)
- [What you’ll see: example outputs](#what-youll-see-example-outputs)
- [How to secure a Veature](#how-to-secure-a-veature)
- [Make it continuous: pipelines + LLMs](#make-it-continuous-pipelines--llms)
- [Condensed Veature Detection Prompt (copy/paste)](#condensed-veature-detection-prompt-copypaste)
- [Examples: using the prompt](#examples-using-the-prompt)
- [Call to action](#call-to-action)

---

## What is a Veature?

A Veature is a documented, intentional feature that works as designed yet creates a realistic path for abuse. It typically:

- documented in release notes, changelogs, docs, or code comments  
- functional (not a defect or CVE by itself)  
- exploitable in a security context under plausible conditions  
- optional or configurable (sometimes enabled by default)  
- introduced for flexibility, debugging, compatibility, or (rarely) planted maliciously

Common patterns:

- message substitutions that resolve external lookups (e.g., `${jndi:...}` in logging)  
- debug or compatibility flags that weaken auth or validation  
- embedded interpreters, deserialization hooks, reflective loaders  
- JSONP or permissive CORS helpers that exfiltrate data  
- hidden egress features that can be steered to attacker infrastructure

## Why traditional tools miss Veatures

- SCA focuses on known CVEs; Veatures are risky semantics, not necessarily CVEs  
- SAST treats many calls/flags as “legitimate” and misses default-on risk  
- DAST only catches what you probe at runtime; Veatures often need crafted preconditions

## A practical way to detect Veatures

Look through the lens of library and version changes. Compare what changed, and match those changes to what the docs claim.

1. review changelogs and release notes for new flags, altered defaults, or interpreters/hooks. watch for keywords: unsafe, legacy, debug, compat, eval/exec, deserialize, script, shell, JNDI, reflection, SSRF, proxy  
2. review code diffs for new execution paths, dynamic loading, deserialization, command runners, widened file/network access, or exposure of internal APIs  
3. trace input paths to confirm whether untrusted input can reach the feature and whether it’s default-on  
4. classify using a simple taxonomy: Insecure Defaults, Dangerous Reflection Hooks, Dual-Use Flags, Unsafe Interpreters, Third-Party Overreach  
5. map to OWASP Top 10 (A03 Injection, A05 Security Misconfiguration, A06 Vulnerable & Outdated Components, A08 Software & Data Integrity Failures, A10 SSRF)  
6. score risk consistently: `Risk = (Exploitability × Exposure × Impact) ÷ 12` on a 1–5 scale each (0–100 overall), and assign Confidence (Low/Medium/High) based on evidence quality

---

## What you’ll see: example outputs

### Good artifact (no Veatures)

```
Library & Versions: perf-utils v1.8:v1.9
Potential Veature Found: No
Notes: Performance refactors only; no new flags/defaults/interpreters.
Evidence: Release notes v1.9; diff shows internal algorithmic changes.
OWASP: n/a
Severity: Low
Risk Score: 0
Confidence: High
Recommendation: Proceed; monitor future releases.
```

### Bad artifact (Veature found)

```
Library & Versions: example-web v2.3:v2.4
Potential Veature Found: Yes
Feature: JSONP endpoint support (default-on)
Category: Insecure Defaults / Third-Party Overreach
Why It’s Risky: Cross-domain script responses → data exfiltration/XSS.
Evidence: Release notes “Add JSONP”; diff adds callback execution path as default.
OWASP: A03 Injection; A05 Security Misconfiguration
Severity: High
Risk Score (0–100): 48   # Exploitability 4 × Exposure 4 × Impact 3 ÷ 12
Confidence: High
Mitigation: Default-off JSONP; prefer strict CORS; deprecate; add telemetry.
```

### Another risky example

```
Library & Versions: logging-core v2.14:v2.15
Potential Veature Found: Yes
Feature: Message lookup substitution with remote resolvers (${jndi:...})
Category: Unsafe Interpreters / Dangerous Reflection Hooks
Why It’s Risky: Attacker-controlled placeholders can trigger remote lookups → potential RCE.
Evidence: Docs mention message lookups; diff shows default-on resolver path pre-fix.
OWASP: A03 Injection; A08 Software & Data Integrity Failures
Severity: Critical
Risk Score: 95
Confidence: High
Mitigation: Remove remote resolver; default-off local resolution; defense-in-depth checks; backport fix.
```

### Batch run summary

```
| Rank | Library & Versions       | Feature                         | Category                          | OWASP    | Risk | Severity | Confidence |
| 1    | logging-core v2.14:v2.15 | Message lookup remote resolvers | Unsafe Interpreters / Reflection  | A03; A08 | 95   | Critical | High       |  <-- 🔴
| 2    | example-web v2.3:v2.4    | JSONP endpoint support          | Insecure Defaults / Overreach     | A03; A05 | 48   | High     | High       |  <-- 🟡/🟠
```

Heatmap legend:

```
🔴 81–100 Critical
🟠 61–80 High
🟡 31–60 Medium
🟢 0–30 Low
```

---

## How to secure a Veature

1. default-off: flip insecure defaults to safe values; provide migration notes  
2. narrow capability: constrain inputs/hosts/protocols; remove remote resolvers or risky schemes  
3. guardrails: policy/allowlists, sandboxing, timeouts, least-privilege I/O  
4. telemetry: log attempted use; alert on suspicious parameters/hosts  
5. documentation: mark as advanced/unsafe with safe settings and runbooks  
6. backport and communicate: patch supported branches; publish clear notes  
7. abuse tests: add negative/abuse tests so reintroductions fail the build

---

## Make it continuous: pipelines + LLMs

Automate Veature detection at:

- dependency PRs (diffs and notes for updated libs)  
- nightly/weekly scans of top-blast-radius dependencies  
- pre-release gates (no Critical/High Veatures without explicit waiver)  
- SBOM enrichment with Veature risk metadata

LLMs can read notes/commits, extract flags/defaults, classify by taxonomy, map to OWASP, score risks, and draft mitigation text and abuse tests. Keep them disciplined:

- force structured outputs (JSON fields)  
- require quoted evidence from notes/diffs  
- lower confidence when evidence is thin  
- add rule-based hard blocks (e.g., eval/exec + default-on)

---

## Condensed Veature Detection Prompt (copy/paste)

```
System role:
You are an AppSec analyst detecting Veatures.

Definition:
A Veature is a documented, intentional feature that works as designed yet creates a realistic path for abuse. It is not a defect. Common traits: documented; functional; exploitable under plausible conditions; optional/configurable (sometimes default-on); introduced for flexibility/debug/compatibility or, rarely, maliciously.

Input syntax (batch accepted):
lib:<library-name> vX.Y:vZ.W
# example single:
# lib:logging-log4j2 v2.14:v2.15
# example batch:
# lib:xz-utils v5.4.4:v5.4.5
# lib:<ADD_LIBRARY_NAME> v<ADD_START>:v<ADD_END>   <-- add more

Task:
1) Retrieve and analyze changelogs, release notes, and code diffs between the versions.
2) Flag features that meet the Veature definition.
3) Classify using Veature taxonomy and map to OWASP Top 10.
4) Score risk and provide mitigations.
5) If multiple inputs, output a ranked summary with heatmap labels.

Heuristics (tight):
- Watch for: unsafe, legacy, debug, compat, eval/exec, deserialize, script, shell, JNDI, reflection, SSRF, proxy, permissive CORS/JSONP.
- Diffs: new execution paths, dynamic loading, deserialization hooks, command runners, widened FS/network access, internal API exposure.
- Input paths: can untrusted input reach it? default-on?
- Evidence first: quote release notes or show diff hunk; lower confidence if thin.

Taxonomy → OWASP:
- Insecure Defaults → A05; A06
- Dangerous Reflection Hooks → A03
- Dual-Use Flags → A05; A07
- Unsafe Interpreters → A03; A08
- Third-Party Overreach → A06; A08; A10 (if external calls)

Risk scoring:
Risk = (Exploitability × Exposure × Impact) ÷ 12   # each 1–5
Severity: 0–30 Low; 31–60 Medium; 61–80 High; 81–100 Critical
Confidence: Low/Medium/High based on evidence quality

Per-library output (strict order):
- Library & Versions
- Potential Veature Found [Yes/No]
- Feature Name/Description
- Category (taxonomy)
- Why It’s Risky
- Evidence (release note quote, changelog URL, diff path:lines)
- OWASP Mapping
- Severity
- Risk Score (0–100)
- Confidence
- Mitigation Recommendation

Batch summary (if multiple):
| Rank | Library & Versions | Feature | Category | OWASP | Risk | Severity | Confidence |
Heatmap labels: 🔴 81–100, 🟠 61–80, 🟡 31–60, 🟢 0–30

Optional adds:
- [ADD_LIBRARY_SOURCES] internal mirrors/SBOM/artifacts
- [ADD_KEYWORDS] stack-specific risky terms
- [ADD_ALLOWED_PROTOCOLS] permitted resolvers/protocols
- [ADD_MITIGATION_STANDARDS] links to secure-defaults/runbooks
- [ADD_OWNER_TEAMS] routing targets in CI/CD
- [ADD_POLICY] gates: block Critical, approval High, ticket Medium
```

---

## Examples: using the prompt

### Single good artifact

```
Input:
lib:perf-utils v1.8:v1.9

Output:
Library & Versions: perf-utils v1.8:v1.9
Potential Veature Found: No
Notes: Performance-only changes; no new flags/defaults/interpreters.
Evidence: Release notes v1.9; diff shows isolated refactors.
OWASP: n/a
Severity: Low
Risk Score: 0
Confidence: High
Recommendation: Proceed; monitor future releases.
```

### Single bad artifact

```
Input:
lib:example-web v2.3:v2.4

Output:
Library & Versions: example-web v2.3:v2.4
Potential Veature Found: Yes
Feature: JSONP endpoint support (default-on)
Category: Insecure Defaults / Third-Party Overreach
Why It’s Risky: Script-based cross-domain access → data exfiltration/XSS.
Evidence: Release notes “Add JSONP”; diff adds callback path enabled by default.
OWASP: A03; A05
Severity: High
Risk Score: 48
Confidence: High
Mitigation: Default-off; strict CORS; deprecate; telemetry.
```

### Batch run

```
Input:
lib:logging-core v2.14:v2.15
lib:example-web v2.3:v2.4

Outputs:
Library & Versions: logging-core v2.14:v2.15
Potential Veature Found: Yes
Feature: Message lookup substitution (${jndi:...})
Category: Unsafe Interpreters / Dangerous Reflection Hooks
Why It’s Risky: Attacker-controlled placeholders → remote lookups → possible RCE.
Evidence: Docs note message lookups; diff shows default-on resolver pre-fix.
OWASP: A03; A08
Severity: Critical
Risk Score: 95
Confidence: High
Mitigation: Remove remote resolver; default-off local; DI checks; backport.

Library & Versions: example-web v2.3:v2.4
Potential Veature Found: Yes
Feature: JSONP endpoint support
Category: Insecure Defaults / Third-Party Overreach
Why It’s Risky: Cross-domain script response enables data exfiltration/XSS.
Evidence: Release notes and diff showing callback execution default.
OWASP: A03; A05
Severity: High
Risk Score: 48
Confidence: High
Mitigation: Default-off JSONP; strict CORS; deprecate; telemetry.

Ranked summary:
| Rank | Library & Versions       | Feature                         | Category                           | OWASP   | Risk | Severity | Confidence |
| 1    | logging-core v2.14:v2.15 | Message lookup remote resolvers | Unsafe Interpreters / Reflection   | A03;A08 | 95   | Critical | High       |  🔴
| 2    | example-web v2.3:v2.4    | JSONP endpoint support          | Insecure Defaults / Overreach      | A03;A05 | 48   | High     | High       |  🟡/🟠
```

---

## Call to action

If this guide helps, pilot Veature detection in one dependency PR workflow this week. Want a JSON/CSV schema and a GitHub Actions snippet that enforces the same scoring and evidence rules? Open an issue or reach out and we’ll add a ready-to-commit example.
