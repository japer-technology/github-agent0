# Report: Adapting Agent Zero Design Theories for .GITCLAW

**Date:** February 22, 2026  
**Classification:** Technical Architecture Report  
**Related:** See [ANALYSIS.md](./ANALYSIS.md) for the full theoretical breakdown.

---

## 1. Purpose

This report translates the design theory analysis of Agent Zero into concrete, actionable adaptation recommendations for the .GITCLAW system. Where the Analysis examines *what* Agent Zero does and *why*, this Report focuses on *how* those principles become .GITCLAW architecture.

---

## 2. Agent Zero at a Glance

| Attribute | Agent Zero | Relevance to .GITCLAW |
|-----------|-----------|----------------------|
| **Core Identity** | General-purpose AI agent framework | Governance-purpose enforcement framework |
| **Behavior Source** | Markdown prompt templates | Policy definition templates |
| **Execution Model** | Hierarchical multi-agent delegation | Parallel governance check orchestration |
| **Learning Model** | Vector-store persistent memory | Governance precedent database |
| **Tool Philosophy** | OS-as-tool, code synthesis | Repository-as-tool, check synthesis |
| **Extension Model** | Lifecycle hook plugins | Git-event hook plugins |
| **Isolation Model** | Docker containers | Ephemeral evaluation sandboxes |
| **Expertise Model** | SKILL.md contextual loading | POLICY.md contextual activation |

---

## 3. Recommended Adaptations

### 3.1 Policy-as-Prompt Architecture

**Source Theory:** Prompt-driven behavioral architecture (§2.1 of Analysis)

**Recommendation:** Implement a `.gitclaw/policies/` directory structure where governance rules are expressed as declarative templates with variable interpolation.

**Proposed Structure:**
```
.gitclaw/
├── policies/
│   ├── main.md                          # Root policy — includes others
│   ├── main.role.md                     # Defines .GITCLAW's governance role
│   ├── main.environment.md              # Repository context variables
│   ├── checks/
│   │   ├── license-compliance.md        # License policy template
│   │   ├── security-scanning.md         # Security policy template
│   │   ├── code-quality.md              # Quality gate template
│   │   └── contributor-agreement.md     # CLA/DCO policy template
│   └── overrides/
│       └── hotfix-exception.md          # Override for emergency merges
├── extensions/
│   ├── pre-merge/
│   │   ├── _10_license_check.py
│   │   ├── _20_security_scan.py
│   │   └── _30_quality_gate.py
│   └── post-merge/
│       └── _10_notify_compliance.py
└── memory/
    └── precedents.db                    # Vector store of past decisions
```

**Variable System:**
```markdown
# License Compliance Policy

## Repository Context
- Repository: {{repo_name}}
- Organization: {{org_name}}
- Primary License: {{declared_license}}
- Last audit: {{last_audit_date}}

## Rules
All dependencies must be compatible with {{declared_license}}.
Files modified in this PR: {{changed_files_count}}
New dependencies added: {{new_dependencies}}
```

**Dynamic Loaders (mirroring Agent Zero's `.py` co-location pattern):**
```python
# license-compliance.py — co-located with license-compliance.md
class LicenseComplianceVars:
    def get_variables(self, repo_context):
        return {
            "declared_license": repo_context.get_license(),
            "new_dependencies": repo_context.get_new_deps_from_diff(),
            "changed_files_count": len(repo_context.changed_files),
        }
```

---

### 3.2 Governance Agent Hierarchy

**Source Theory:** Hierarchical multi-agent delegation (§2.2 of Analysis)

**Recommendation:** Implement a root governance orchestrator that delegates specialized checks to subordinate agents, each with its own scoped context.

**Proposed Flow:**
```
PR Opened / Updated
       │
       ▼
┌──────────────────┐
│  .GITCLAW Root   │  ← Orchestrator: reads PR metadata, determines applicable policies
│  Governance Agent│
└──────┬───────────┘
       │
       ├──────────────────┬──────────────────┬──────────────────┐
       ▼                  ▼                  ▼                  ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│   License   │  │  Security   │  │   Quality   │  │     CLA     │
│   Agent     │  │   Agent     │  │   Agent     │  │   Agent     │
│             │  │             │  │             │  │             │
│ Context:    │  │ Context:    │  │ Context:    │  │ Context:    │
│ dependency  │  │ security-   │  │ linting,    │  │ contributor │
│ manifests,  │  │ relevant    │  │ test files, │  │ identity,   │
│ license     │  │ code paths, │  │ coverage    │  │ agreement   │
│ files only  │  │ CVE data    │  │ reports     │  │ records     │
└─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘
       │                  │                  │                  │
       ▼                  ▼                  ▼                  ▼
   PASS/FAIL          PASS/FAIL          PASS/FAIL          PASS/FAIL
   + reasoning        + reasoning        + reasoning        + reasoning
       │                  │                  │                  │
       └──────────────────┴──────────────────┴──────────────────┘
                                   │
                                   ▼
                          ┌─────────────────┐
                          │  Consensus       │
                          │  Aggregator      │
                          │                  │
                          │  ALL PASS → ✅   │
                          │  ANY FAIL → ❌   │
                          │  MIXED  → ⚠️     │
                          └─────────────────┘
```

**Key Differences from Agent Zero:**
- **Parallel execution** rather than sequential delegation (latency constraint)
- **Consensus requirement** rather than single-chain reporting (governance constraint)
- **Scoped context** per agent prevents information leakage between checks
- **Deterministic profiles** — each subordinate agent profile is fixed, not dynamically created

---

### 3.3 Governance Precedent Memory

**Source Theory:** Memory-augmented learning (§2.3 of Analysis)

**Recommendation:** Implement a vector-store-backed precedent system that records all governance decisions with full reasoning chains.

**Memory Categories (adapted from Agent Zero's four-category model):**

| Category | Agent Zero Equivalent | .GITCLAW Implementation |
|----------|----------------------|------------------------|
| **Decisions** | Fragments | Each PR governance outcome stored with PR metadata, policy versions, and check results |
| **Precedents** | Solutions | Successful resolution patterns — how specific policy violations were resolved |
| **Exceptions** | User-provided info | Explicitly granted governance exceptions with expiration dates and justifications |
| **Metadata** | Metadata | Contributor profiles, repository compliance scores, trend data |

**Precedent Retrieval Flow:**
```
New Policy Violation Detected
         │
         ▼
    ┌─────────────┐
    │ Query        │ "dependency X violates license Y"
    │ Precedent    │
    │ Memory       │ threshold: 0.8, limit: 5
    └──────┬──────┘
           │
           ▼
    ┌─────────────────────────────────────────────┐
    │ Similar Precedents Found:                    │
    │                                              │
    │ 1. PR #423 — same dep, resolved by replacing │
    │    with dep Z (similarity: 0.92)             │
    │ 2. PR #891 — exception granted for 90 days   │
    │    (similarity: 0.87)                         │
    │ 3. PR #102 — rejected, contributor updated    │
    │    (similarity: 0.85)                         │
    └─────────────────────────────────────────────┘
           │
           ▼
    Include in governance decision context
    (suggested resolution + historical precedent)
```

**Memory Consolidation (adapted from Agent Zero):**
- Periodically merge related decisions into summary records
- Old individual decisions compressed into statistical trends
- Recent decisions kept in full detail
- Consolidation driven by utility model (same pattern as Agent Zero)

---

### 3.4 Semantic Policy Activation

**Source Theory:** SKILL.md contextual expertise (§2.7 of Analysis)

**Recommendation:** Implement `POLICY.md` files following a standard analogous to SKILL.md, activated semantically based on PR content.

**POLICY.md Format:**
```yaml
---
name: "cryptography-compliance"
description: "Enforces cryptographic library standards and export control compliance. Activates when crypto-related files or dependencies are modified."
version: "2.1.0"
author: "Security Team"
severity: "blocking"
tags: ["security", "cryptography", "export-control", "compliance"]
trigger_paths:
  - "**/*crypto*"
  - "**/*cipher*"
  - "**/*tls*"
  - "**/*ssl*"
trigger_dependencies:
  - "openssl"
  - "libsodium"
  - "pycryptodome"
---

# Cryptography Compliance Policy

## When This Activates
This policy activates when any PR modifies files matching the trigger paths
or adds/modifies dependencies matching trigger_dependencies.

## Requirements
1. All cryptographic libraries must be from the approved list
2. No custom cryptographic implementations allowed
3. Export control classification must be documented
4. FIPS 140-2 compliance required for government-facing services

## Approved Libraries
- OpenSSL >= 3.0
- libsodium >= 1.0.18
- Go stdlib crypto/*

## Validation Script
See `scripts/validate_crypto.py` for automated checking.
```

**Activation Logic:**
- On PR creation, compute semantic embeddings of changed file paths, diff content, and dependency changes
- Match against indexed POLICY.md embeddings (tags, description, trigger patterns)
- Only load policies with similarity above threshold (reduces noise and evaluation cost)
- Always load policies matching explicit `trigger_paths` or `trigger_dependencies` rules

---

### 3.5 Extension Hook Architecture

**Source Theory:** Extensibility-first modularity (§2.5 of Analysis)

**Recommendation:** Define .GITCLAW lifecycle hooks analogous to Agent Zero's extension points.

**Proposed Hook Points:**

| Hook | Trigger | Use Case |
|------|---------|----------|
| `pr_opened` | PR created | Initial policy evaluation, automatic labeling |
| `pr_updated` | New commits pushed | Re-evaluation of changed policies |
| `pre_merge` | Merge requested | Final blocking checks, consensus verification |
| `post_merge` | Merge completed | Compliance logging, notification, precedent recording |
| `release_gate` | Release tag created | Release-specific governance (changelog, version, signing) |
| `policy_changed` | `.gitclaw/` files modified | Re-index policies, validate policy syntax |
| `schedule` | Cron-based | Periodic compliance audits, memory consolidation |

**Extension Loading (mirroring Agent Zero's mechanism):**
```
1. Load organization-level extensions from .github/.gitclaw/extensions/{hook}/
2. Load repository-level extensions from .gitclaw/extensions/{hook}/
3. Merge: repository-level overrides organization-level by filename match
4. Execute in alphabetical order (numeric prefix for ordering)
```

---

### 3.6 Containerized Evaluation Sandboxes

**Source Theory:** Containerized environment isolation (§2.6 of Analysis)

**Recommendation:** Execute all governance checks in ephemeral containers.

**Properties:**
- **Ephemeral:** Destroyed after each evaluation — no state leakage between PRs
- **Deterministic:** Container image pins all tool versions
- **Auditable:** Container image hash recorded with every governance decision
- **Isolated:** PR code cannot modify the governance evaluation environment
- **Reproducible:** Any past governance decision can be re-evaluated using the archived container image + PR snapshot

**Container Lifecycle:**
```
PR Event Trigger
       │
       ▼
  Pull governance container image (pinned hash)
       │
       ▼
  Mount PR diff + policy templates (read-only)
       │
       ▼
  Execute governance checks
       │
       ▼
  Capture structured results + reasoning logs
       │
       ▼
  Destroy container
       │
       ▼
  Publish results to PR + audit log
```

---

## 4. Adaptation Risk Matrix

| Agent Zero Theory | .GITCLAW Adaptation | Risk Level | Primary Risk |
|-------------------|---------------------|------------|--------------|
| Prompt-driven behavior | Policy templates | 🟢 Low | Template syntax complexity |
| Multi-agent delegation | Parallel governance checks | 🟡 Medium | Latency; consensus logic complexity |
| Memory-augmented learning | Precedent database | 🟡 Medium | Memory staleness; bias amplification |
| Tool-as-code synthesis | Dynamic check generation | 🔴 High | Non-deterministic governance outcomes |
| Extension modularity | Hook-based plugins | 🟢 Low | Extension conflict resolution |
| Container isolation | Ephemeral sandboxes | 🟢 Low | Container startup latency |
| SKILL.md expertise | POLICY.md activation | 🟢 Low | False-negative policy activation |

---

## 5. What Should NOT Be Adapted

Certain Agent Zero design choices are inappropriate for .GITCLAW:

1. **Unrestricted code generation and execution:** Agent Zero generates and runs arbitrary code. .GITCLAW governance checks must be pre-validated or sandboxed with strict resource limits.

2. **Single-chain hierarchy:** Agent Zero uses one-superior, one-subordinate chains. .GITCLAW needs consensus among independent agents — a graph, not a chain.

3. **Behavioral self-modification:** Agent Zero can modify its own behavior rules at runtime via user instruction. .GITCLAW policies should require explicit, audited changes through version control — never self-modification during evaluation.

4. **Implicit trust model:** Agent Zero trusts its own outputs. .GITCLAW must implement verification — governance decisions should be independently verifiable, not self-affirming.

5. **Context window dependence:** Agent Zero relies on LLM context windows for reasoning. .GITCLAW critical-path decisions should have deterministic fallbacks that don't depend on LLM availability.

---

## 6. Proposed .GITCLAW Architecture Summary

```
.gitclaw/
├── policies/                    # ← From prompt-driven architecture
│   ├── main.md
│   ├── checks/
│   │   ├── *.md                 # POLICY.md format files
│   │   └── *.py                 # Dynamic variable loaders
│   └── overrides/
│       └── *.md
├── agents/                      # ← From multi-agent delegation
│   ├── license-checker/
│   │   ├── profile.yaml
│   │   └── policies/            # Agent-specific policy overrides
│   ├── security-auditor/
│   └── quality-gate/
├── extensions/                  # ← From extensibility modularity
│   ├── pr_opened/
│   ├── pre_merge/
│   ├── post_merge/
│   └── release_gate/
├── memory/                      # ← From memory-augmented learning
│   ├── precedents.db
│   ├── exceptions.db
│   └── contributor-profiles.db
├── skills/                      # ← From SKILL.md expertise
│   └── *.POLICY.md
├── container.yaml               # ← From container isolation
└── config.yaml                  # Global .GITCLAW configuration
```

---

## 7. Implementation Priority

| Priority | Component | Justification |
|----------|-----------|---------------|
| **P0** | Policy-as-prompt templates | Foundation — everything builds on this |
| **P0** | Extension hook architecture | Required for any CI/CD integration |
| **P1** | Containerized evaluation | Required for security and reproducibility |
| **P1** | Governance agent hierarchy | Required for scalable multi-check workflows |
| **P2** | Semantic policy activation | Optimization — reduces unnecessary evaluation |
| **P2** | Precedent memory system | Learning layer — improves over time |
| **P3** | Dynamic check synthesis | High risk — implement after core is stable |

---

## 8. Conclusion

Agent Zero's design philosophy — *organic, transparent, extensible, and learning* — translates powerfully into the governance domain when constrained by determinism, auditability, and adversarial-input assumptions. The five core adaptations (policy templates, governance agent hierarchy, precedent memory, semantic policy activation, and containerized sandboxes) form a coherent architecture that gives .GITCLAW:

- **Customizability** without code changes (policy templates)
- **Scalability** through parallel specialized agents (governance hierarchy)
- **Intelligence** that improves with use (precedent memory)
- **Efficiency** through targeted activation (semantic policies)
- **Trustworthiness** through isolation and reproducibility (containers)

The result is a governance framework that inherits Agent Zero's most powerful qualities while transforming its creative-agent paradigm into a deterministic-governance paradigm suitable for version control enforcement.
