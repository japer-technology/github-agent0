# Feasibility Report: Running Agent Zero as .GITCLAW on GitHub Infrastructure

**Date:** February 22, 2026  
**Scope:** Can Agent Zero's codebase and design run natively on GitHub's infrastructure (Actions, API, Packages, Issues, etc.) to function as a .GITCLAW governance system?  
**Verdict:** Not out-of-the-box. Feasible with significant architectural surgery. The design *theories* transfer well; the *implementation* does not.

---

## 1. Executive Summary

Agent Zero is a **long-running, stateful, interactive server** built on Flask/ASGI, WebSockets, Docker-in-Docker, local FAISS vector stores, and a local filesystem. GitHub Actions is an **ephemeral, event-driven, stateless compute platform** with 6-hour job limits, no persistent storage, and no inbound networking. These are fundamentally different runtime models.

However, the **design theories** behind Agent Zero — prompt-driven behavior, multi-agent delegation, memory-augmented learning, extensible hooks, and contextual skills — are not only compatible with GitHub-as-infrastructure, they map onto it with surprising elegance when re-implemented against GitHub-native primitives.

This report maps every Agent Zero subsystem to a GitHub infrastructure equivalent, identifies what works, what breaks, and what must be re-architected.

---

## 2. Infrastructure Mapping: Agent Zero → GitHub

### 2.1 Compute Runtime

| Agent Zero | GitHub Equivalent | Gap Analysis |
|-----------|-------------------|-------------|
| Kali Linux Docker container (persistent, root access) | GitHub Actions runner (`ubuntu-latest`, ephemeral, sudo available) | **Major gap.** A0 assumes persistent container with state across requests. Actions runners are destroyed after each job. No persistence between workflow runs. |
| Flask/ASGI web server (port 80, always-on) | GitHub Actions (event-triggered, no inbound HTTP) | **Major gap.** No web UI, no WebSocket, no persistent HTTP server. Must switch to event-driven model. |
| Supervisord process manager | GitHub Actions job orchestration | **Partial match.** Actions `jobs` with `needs` dependencies replicate process orchestration. |
| SSH into runtime container | `actions/checkout` + shell steps | **Adequate.** Shell execution in Actions steps replaces SSH. |

**Verdict:** The compute model must be **completely inverted** — from always-on server to event-triggered ephemeral jobs. This is the single largest architectural change.

### 2.2 Event System

| Agent Zero | GitHub Equivalent | Gap Analysis |
|-----------|-------------------|-------------|
| WebSocket real-time events | GitHub webhook events (`pull_request`, `push`, `issue_comment`, etc.) | **Good match for .GITCLAW.** Governance is inherently event-driven (PR opened, commit pushed, review submitted). |
| User chat messages via Web UI | GitHub Issue/PR comments, or `workflow_dispatch` inputs | **Good match.** Comments become the "chat" interface. |
| Scheduler (cron-based tasks) | GitHub Actions `schedule` trigger (cron syntax) | **Direct match.** Both use cron expressions. |
| Internal API endpoints (75+ endpoints) | GitHub REST/GraphQL API + Actions artifacts | **Partial match.** Most internal endpoints become unnecessary; GitHub API replaces them. |

**Verdict:** GitHub's event system is **superior** for governance use cases. Agent Zero's WebSocket/polling model is designed for interactive UI — governance doesn't need that.

### 2.3 Storage & State

| Agent Zero | GitHub Equivalent | Gap Analysis |
|-----------|-------------------|-------------|
| Local filesystem (`/a0/usr/`, `/a0/memory/`, etc.) | GitHub repo files + Actions artifacts + Actions cache | **Significant gap.** No persistent writable filesystem across runs. Must use repo commits, artifact storage, or external storage. |
| FAISS vector store (local `.db` files) | GitHub Actions cache (7-day TTL) or committed to repo | **Significant gap.** FAISS files can be cached between runs but cache is best-effort. For durability, must commit to repo or use external vector DB. |
| SQLite/file-based memory | GitHub repo files (committed) or GitHub Actions artifacts (90-day TTL) | **Workable.** Precedent database committed to `.gitclaw/memory/` in the repo itself. |
| Docker volumes for persistence | None (ephemeral) | **Gap.** All state must be externalized. |
| `usr/settings.json` (runtime config) | Repository file `.gitclaw/config.yaml` + GitHub repo variables/secrets | **Good match.** GitHub has native secrets and variables management. |
| `usr/secrets.env` | GitHub Actions secrets (`${{ secrets.* }}`) | **Direct match.** GitHub's secrets management is more secure than Agent Zero's file-based approach. |

**Verdict:** State management requires the most creative re-architecture. The solution is **repo-as-database** — governance state committed back to the repository, with GitHub Actions cache for ephemeral/performance state.

### 2.4 LLM Integration

| Agent Zero | GitHub Equivalent | Gap Analysis |
|-----------|-------------------|-------------|
| LiteLLM with 20+ providers | LiteLLM runs fine in Actions (it's just Python HTTP calls) | **No gap.** LLM API calls work identically from Actions runners. |
| 4 model types (chat, utility, embedding, browser) | Same — configured via GitHub secrets for API keys | **No gap.** |
| `conf/model_providers.yaml` | `.gitclaw/models.yaml` + secrets for API keys | **Direct match.** |
| Browser model (Playwright-based) | Not needed for governance | **N/A.** Browser agent irrelevant for .GITCLAW. |

**Verdict:** LLM integration is **fully compatible**. This is Agent Zero's most portable subsystem.

### 2.5 Multi-Agent Delegation

| Agent Zero | GitHub Equivalent | Gap Analysis |
|-----------|-------------------|-------------|
| In-process subordinate agent spawning | GitHub Actions matrix jobs or reusable workflows | **Good match.** Each governance check agent becomes a parallel matrix job or a called reusable workflow. |
| Agent context isolation (separate context windows) | Job isolation (each job has its own runner) | **Perfect match.** GitHub naturally isolates jobs. |
| Agent-to-agent communication (in-memory) | Job outputs → dependent job inputs, or artifact passing | **Adequate.** `jobs.<id>.outputs` passes structured data between jobs. |
| Dynamic subordinate creation at runtime | `workflow_dispatch` triggering child workflows | **Partial match.** Can't dynamically spawn arbitrary agents mid-job, but can pre-define the governance hierarchy. |

**Verdict:** GitHub's job model is **better suited** for parallel governance checks than Agent Zero's sequential subordinate chain. The hierarchy becomes a DAG of jobs rather than a tree of in-process agents.

### 2.6 Prompt System

| Agent Zero | GitHub Equivalent | Gap Analysis |
|-----------|-------------------|-------------|
| `prompts/` directory with Markdown templates | `.gitclaw/policies/` directory with Markdown templates | **Direct match.** Files in repo, read at runtime. |
| Template inclusion (`{{ include }}`) | Custom template engine or simple concatenation in Python | **Minor gap.** Need to port the template engine (trivial — it's just string replacement). |
| Variable interpolation (`{{var}}`) | Same — Python string formatting | **No gap.** |
| Dynamic variable loaders (`.py` co-located files) | Python scripts in `.gitclaw/policies/*.py` | **No gap.** |
| Profile-based prompt overrides | Organization-level `.github/.gitclaw/` → repo-level `.gitclaw/` override | **Good match.** GitHub's `.github` repository pattern already supports org-level defaults. |

**Verdict:** The prompt system is **directly portable**. This is where Agent Zero's theory shines brightest for .GITCLAW.

### 2.7 Tool System

| Agent Zero | GitHub Equivalent | Gap Analysis |
|-----------|-------------------|-------------|
| `code_execution_tool` (Python, Node.js, terminal) | GitHub Actions `run` steps with any language | **Direct match.** Actions runners have Python, Node.js, and shell pre-installed. |
| `memory_save/load/delete/forget` | Custom Python against repo-committed vector store or API calls to external DB | **Requires adaptation.** Memory operations must read from / write to persistent storage (repo files or external). |
| `search_engine` (SearXNG) | Standard HTTP search API calls | **Minor gap.** SearXNG not available, but web search APIs work from Actions. |
| `call_subordinate` | `workflow_dispatch` or matrix strategy | **Architectural change.** Pre-defined rather than dynamic. |
| `browser_agent` (Playwright) | Not needed for governance | **N/A.** |
| `skills_tool` | File reading from `.gitclaw/skills/` | **Direct match.** |

**Verdict:** Most tools **translate directly** or become unnecessary. The `code_execution_tool` pattern maps perfectly to Actions `run` steps.

### 2.8 Extension System

| Agent Zero | GitHub Equivalent | Gap Analysis |
|-----------|-------------------|-------------|
| Extension points (`agent_init`, `message_loop_*`, etc.) | GitHub Actions workflow events (`pull_request`, `push`, `schedule`, etc.) | **Conceptual match.** Events replace lifecycle hooks. |
| Extension auto-discovery from filesystem | Composite actions or reusable workflows in `.gitclaw/extensions/` | **Good match.** |
| Alphabetical execution with numeric prefix ordering | Job dependency ordering (`needs`) or step ordering within a job | **Direct match.** |
| Extension override (profile-specific replaces default by filename) | Repo-level `.gitclaw/` overrides org-level `.github/.gitclaw/` | **Good match.** |

**Verdict:** GitHub's event and workflow system is a **natural extension framework** for governance hooks.

---

## 3. The Re-Architecture: What .GITCLAW-on-GitHub Looks Like

### 3.1 Core Workflow Structure

```yaml
# .github/workflows/gitclaw-governance.yml
name: .GITCLAW Governance

on:
  pull_request:
    types: [opened, synchronize, reopened]
  pull_request_review:
    types: [submitted]
  issue_comment:
    types: [created]
  schedule:
    - cron: '0 6 * * 1'  # Weekly compliance audit

permissions:
  contents: write
  pull-requests: write
  issues: write

jobs:
  # ─── Orchestrator (Agent 0) ───
  orchestrator:
    runs-on: ubuntu-latest
    outputs:
      policies: ${{ steps.activate.outputs.policies }}
      checks: ${{ steps.activate.outputs.checks }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      
      - name: Install GITCLAW core
        run: pip install -r .gitclaw/requirements.txt
      
      - name: Load precedent memory cache
        uses: actions/cache@v4
        with:
          path: .gitclaw/memory/
          key: gitclaw-memory-${{ github.repository }}-${{ hashFiles('.gitclaw/memory/**') }}
          restore-keys: gitclaw-memory-${{ github.repository }}-
      
      - name: Determine applicable policies
        id: activate
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        run: |
          python .gitclaw/engine/orchestrator.py \
            --event "${{ github.event_name }}" \
            --pr "${{ github.event.pull_request.number }}" \
            --output-policies policies.json \
            --output-checks checks.json
          echo "policies=$(cat policies.json)" >> "$GITHUB_OUTPUT"
          echo "checks=$(cat checks.json)" >> "$GITHUB_OUTPUT"

  # ─── Parallel Governance Checks (Subordinate Agents) ───
  governance-check:
    needs: orchestrator
    runs-on: ubuntu-latest
    strategy:
      matrix:
        check: ${{ fromJson(needs.orchestrator.outputs.checks) }}
      fail-fast: false  # Run all checks even if one fails
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      
      - name: Load check-specific policy
        run: |
          python .gitclaw/engine/load_policy.py \
            --check "${{ matrix.check.name }}" \
            --policies-dir ".gitclaw/policies/"
      
      - name: Execute governance check
        id: check
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        run: |
          python .gitclaw/engine/run_check.py \
            --check "${{ matrix.check.name }}" \
            --pr "${{ github.event.pull_request.number }}" \
            --output result.json
          echo "result=$(cat result.json)" >> "$GITHUB_OUTPUT"
      
      - name: Upload check result
        uses: actions/upload-artifact@v4
        with:
          name: check-${{ matrix.check.name }}
          path: result.json

  # ─── Consensus Aggregator ───
  consensus:
    needs: governance-check
    runs-on: ubuntu-latest
    if: always()
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      
      - name: Download all check results
        uses: actions/download-artifact@v4
        with:
          path: results/
      
      - name: Aggregate governance decision
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          python .gitclaw/engine/consensus.py \
            --results-dir results/ \
            --pr "${{ github.event.pull_request.number }}"
      
      - name: Record precedent
        run: |
          python .gitclaw/engine/record_precedent.py \
            --results-dir results/ \
            --memory-dir .gitclaw/memory/
      
      - name: Update memory cache
        uses: actions/cache/save@v4
        with:
          path: .gitclaw/memory/
          key: gitclaw-memory-${{ github.repository }}-${{ hashFiles('.gitclaw/memory/**') }}
```

### 3.2 Repository Structure

```
repo/
├── .github/
│   └── workflows/
│       └── gitclaw-governance.yml      # Main workflow
├── .gitclaw/
│   ├── config.yaml                      # Global configuration
│   ├── requirements.txt                 # Python dependencies (subset of A0)
│   ├── engine/                          # Core runtime (extracted from A0)
│   │   ├── orchestrator.py              # Policy activation + check planning
│   │   ├── load_policy.py              # Template engine (from A0 prompts system)
│   │   ├── run_check.py                # Single check executor (from A0 agent loop)
│   │   ├── consensus.py                # Decision aggregator (new — not in A0)
│   │   ├── record_precedent.py         # Memory writer (from A0 memory_save)
│   │   ├── templates.py                # A0's prompt template engine (ported)
│   │   └── llm.py                       # LiteLLM wrapper (from A0 models.py)
│   ├── policies/                        # ← From A0's prompts/ directory
│   │   ├── main.md
│   │   ├── checks/
│   │   │   ├── license-compliance.md
│   │   │   ├── license-compliance.py    # Dynamic variable loader
│   │   │   ├── security-scanning.md
│   │   │   └── code-quality.md
│   │   └── overrides/
│   │       └── hotfix-exception.md
│   ├── skills/                          # ← From A0's SKILL.md system
│   │   └── dependency-audit/
│   │       ├── POLICY.md
│   │       └── scripts/validate.py
│   ├── memory/                          # ← From A0's memory system
│   │   ├── precedents.faiss             # Vector index (cached between runs)
│   │   ├── precedents.json              # Metadata store
│   │   └── solutions.json               # Cached resolution patterns
│   └── agents/                          # ← From A0's agent profiles
│       ├── license-checker/
│       │   └── profile.yaml
│       ├── security-auditor/
│       │   └── profile.yaml
│       └── quality-gate/
│           └── profile.yaml
```

---

## 4. Component-by-Component Portability Assessment

### 4.1 What Ports Directly (Little or No Change)

| A0 Component | Files | GitHub Adaptation |
|-------------|-------|-------------------|
| **Prompt template engine** | `python/helpers/files.py` (template loading, `{{ include }}`, `{{var}}` interpolation) | Port the ~200 lines of template code. Works identically reading from `.gitclaw/policies/`. |
| **LLM integration** | `models.py` (~930 lines) | Use LiteLLM directly. Trim to chat + embedding only (drop browser model). ~300 lines needed. |
| **Memory save/load** | `python/tools/memory_save.py`, `memory_load.py`, `python/helpers/` (FAISS helpers) | Port vector store operations. Use `actions/cache` for FAISS persistence between runs. |
| **Policy/prompt files** | `prompts/*.md` | Rename to `.gitclaw/policies/*.md`. Format is directly reusable. |
| **SKILL.md system** | `python/tools/skills_tool.py` | Rename to POLICY.md. Same YAML frontmatter + Markdown body structure. |
| **Dynamic variable loaders** | `prompts/*.py` co-located files | Identical pattern in `.gitclaw/policies/*.py`. |
| **Settings/config** | `python/helpers/settings.py` | Simplify to YAML config + GitHub secrets. ~50 lines. |
| **Git operations** | `python/helpers/git.py` | Directly reusable — already uses `GitPython`. |

**Estimated portable code:** ~3,000–4,000 lines out of ~33,000 total.

### 4.2 What Requires Significant Rework

| A0 Component | Why It Breaks | GitHub Solution |
|-------------|---------------|-----------------|
| **Agent main loop** (`agent.py`, 1,011 lines) | Designed for interactive chat loop with streaming. Governance is single-shot evaluation. | Rewrite as single-pass: load policy → evaluate PR → produce structured result. ~200 lines. |
| **Multi-agent hierarchy** | In-process subordinate spawning via Python classes. | Replace with GitHub Actions matrix strategy. Orchestrator job outputs check list; matrix jobs execute in parallel. |
| **WebSocket/Web UI** (`run_ui.py`, `webui/`, `python/api/`) | No web server on GitHub Actions. | **Eliminate entirely.** GitHub PR comments and status checks become the UI. |
| **Code execution tool** | Manages persistent shell sessions, multitasking. | Replace with simple `subprocess.run()` in Actions steps. No session persistence needed. |
| **Extension lifecycle hooks** | In-process Python plugin system with 11 hook points. | Replace with GitHub Actions workflow composition. Each "extension" becomes a reusable workflow or composite action. |
| **Memory consolidation** | Background deferred task that runs periodically. | Move to scheduled workflow (`cron`). Weekly consolidation job. |
| **SearXNG search** | Self-hosted search engine in Docker. | Use standard search APIs (SerpAPI, Brave Search) via HTTP. |

### 4.3 What Gets Eliminated

| A0 Component | Lines of Code | Reason for Elimination |
|-------------|---------------|----------------------|
| Web UI (`webui/`) | ~5,000+ | GitHub PR interface replaces it |
| Flask/ASGI server (`run_ui.py`) | 559 | No web server needed |
| WebSocket infrastructure | ~2,000 | No real-time streaming needed |
| Docker container management | ~300 | GitHub provides the runner |
| SSH/RFC system | ~500 | Not applicable |
| Browser agent (Playwright) | ~1,000 | Not needed for governance |
| Tunnel system (Cloudflare) | ~300 | Not applicable |
| TTS/STT (Whisper, Kokoro) | ~500 | Not applicable |
| File browser UI | ~300 | Not applicable |
| Upload system | ~200 | Not applicable |
| 75 API endpoints (`python/api/`) | ~4,000 | GitHub API replaces them |

**Estimated eliminable code:** ~15,000+ lines (roughly half the codebase).

---

## 5. GitHub-Native Primitives for .GITCLAW Functions

### 5.1 Governance Decision Output

Agent Zero outputs via WebSocket to a Web UI. .GITCLAW outputs via GitHub API:

```python
# .gitclaw/engine/consensus.py — posting governance decision to PR
import requests
import json
import os

def post_governance_decision(pr_number, decision, checks):
    """Post structured governance decision as PR comment + status checks."""
    token = os.environ["GITHUB_TOKEN"]
    repo = os.environ["GITHUB_REPOSITORY"]
    headers = {"Authorization": f"token {token}", "Accept": "application/vnd.github+json"}
    
    # Post detailed comment
    body = format_governance_comment(decision, checks)
    requests.post(
        f"https://api.github.com/repos/{repo}/issues/{pr_number}/comments",
        headers=headers,
        json={"body": body}
    )
    
    # Set commit status for each check
    sha = os.environ["GITHUB_SHA"]
    for check in checks:
        requests.post(
            f"https://api.github.com/repos/{repo}/statuses/{sha}",
            headers=headers,
            json={
                "state": "success" if check["passed"] else "failure",
                "context": f".gitclaw/{check['name']}",
                "description": check["summary"][:140],
            }
        )

def format_governance_comment(decision, checks):
    """Format as structured Markdown — machine-readable AND human-readable."""
    emoji = "✅" if decision["approved"] else "❌"
    lines = [
        f"## {emoji} .GITCLAW Governance Decision",
        "",
        f"**Verdict:** {'APPROVED' if decision['approved'] else 'BLOCKED'}",
        f"**Policies evaluated:** {len(checks)}",
        f"**Precedents referenced:** {decision.get('precedents_used', 0)}",
        "",
        "### Check Results",
        "| Check | Result | Reasoning |",
        "|-------|--------|-----------|",
    ]
    for check in checks:
        status = "✅ Pass" if check["passed"] else "❌ Fail"
        lines.append(f"| {check['name']} | {status} | {check['reasoning']} |")
    
    # Machine-readable JSON block for downstream automation
    lines.extend([
        "",
        "<details><summary>Machine-readable decision (JSON)</summary>",
        "",
        "```json",
        json.dumps({"decision": decision, "checks": checks}, indent=2),
        "```",
        "</details>",
    ])
    return "\n".join(lines)
```

### 5.2 Memory Persistence via Repository

```python
# .gitclaw/engine/record_precedent.py
import json
import subprocess
from datetime import datetime

def record_and_commit(pr_number, decision, checks, memory_dir):
    """Record governance precedent and commit to repo."""
    # Append to precedent log
    record = {
        "pr": pr_number,
        "timestamp": datetime.utcnow().isoformat(),
        "decision": decision,
        "checks": checks,
    }
    
    log_path = f"{memory_dir}/precedent_log.jsonl"
    with open(log_path, "a") as f:
        f.write(json.dumps(record) + "\n")
    
    # Update FAISS index (if using vector memory)
    # update_vector_index(record, memory_dir)
    
    # Commit back to repo on a dedicated branch
    subprocess.run(["git", "config", "user.name", "gitclaw[bot]"], check=True)
    subprocess.run(["git", "config", "user.email", "gitclaw[bot]@users.noreply.github.com"], check=True)
    subprocess.run(["git", "add", memory_dir], check=True)
    subprocess.run(["git", "commit", "-m", f"chore(gitclaw): record precedent for PR #{pr_number}"], check=True)
    subprocess.run(["git", "push"], check=True)
```

### 5.3 Organization-Level Policy Inheritance

GitHub's `.github` repository pattern provides organization-level defaults:

```
org/.github/                              # Organization-level defaults
└── .gitclaw/
    ├── policies/
    │   ├── license-compliance.md         # All repos must check licenses
    │   └── security-baseline.md          # Minimum security requirements
    └── config.yaml                        # Org-wide GITCLAW settings

repo/.gitclaw/                            # Repository-level overrides
├── policies/
│   └── security-baseline.md              # Override: stricter security for this repo
└── config.yaml                            # Repo-specific settings
```

This mirrors Agent Zero's `prompts/` (default) → `agents/<profile>/prompts/` (override) cascade exactly.

---

## 6. Resource Constraints & Limits

| Resource | GitHub Actions Limit | Agent Zero Requirement | Compatible? |
|----------|---------------------|----------------------|-------------|
| **Job duration** | 6 hours max | Governance checks should complete in minutes | ✅ Yes |
| **Concurrent jobs** | 20 (free) / 500 (enterprise) | Parallel governance checks | ✅ Yes (even free tier handles typical PR checks) |
| **Artifacts** | 500 MB per artifact, 90-day retention | FAISS index + decision logs | ✅ Yes (governance data is small) |
| **Cache** | 10 GB total, 7-day inactivity eviction | Memory/precedent cache | ⚠️ Marginal (must commit important state to repo) |
| **Secrets** | Unlimited per repo/org | LLM API keys, tokens | ✅ Yes |
| **API rate limits** | 1,000 requests/hour (Actions GITHUB_TOKEN) | PR comments + status updates | ✅ Yes |
| **Runner memory** | 7 GB RAM | FAISS + LLM client | ✅ Yes |
| **Runner disk** | 14 GB SSD | Repo checkout + dependencies | ✅ Yes |
| **Minutes per month** | 2,000 (free) / 50,000 (Team) / unlimited (Enterprise) | Depends on PR volume + LLM latency | ⚠️ Cost consideration |

---

## 7. Cost Model

### Per-PR Governance Evaluation

| Step | Duration | Actions Minutes | LLM Cost (est.) |
|------|----------|----------------|-----------------|
| Orchestrator (policy activation) | ~30s | 1 min | ~$0.01 (semantic matching) |
| 4 parallel governance checks | ~60s each | 4 min | ~$0.08 (4 × LLM evaluation) |
| Consensus aggregation | ~15s | 1 min | ~$0.02 |
| Precedent recording | ~10s | 1 min | $0.00 |
| **Total per PR** | **~2 min wall time** | **~7 min** | **~$0.11** |

### Monthly Estimate (Active Repository)

| Scenario | PRs/month | Actions Minutes | LLM Cost | Total |
|----------|-----------|----------------|----------|-------|
| Small team (20 PRs/mo) | 20 | 140 min | $2.20 | Free tier compatible |
| Medium team (100 PRs/mo) | 100 | 700 min | $11.00 | Free tier compatible |
| Large team (500 PRs/mo) | 500 | 3,500 min | $55.00 | Team plan recommended |
| Enterprise (2000 PRs/mo) | 2,000 | 14,000 min | $220.00 | Enterprise plan |

---

## 8. What Agent Zero Code Can Be Directly Reused

### Reusable Modules (Copy with Minor Adaptation)

| Module | A0 Path | Lines | Adaptation Needed |
|--------|---------|-------|-------------------|
| Template engine | `python/helpers/files.py` (template functions) | ~200 | Strip file-browser code, keep `read_file`, `include`, `interpolate` |
| LLM wrapper | `models.py` | ~300 (of 929) | Keep `ModelConfig`, `get_chat_model`, `get_embedding_model`. Drop browser model, rate limiter (Actions handles concurrency). |
| FAISS memory | `python/tools/memory_save.py`, `memory_load.py` | ~400 | Change file paths to `.gitclaw/memory/`. Add git-commit persistence. |
| Prompt/policy loader | `python/helpers/files.py` (get_variables, VariablesPlugin) | ~150 | Rename "prompts" to "policies". Keep co-located `.py` loader pattern. |
| JSON extraction | `python/helpers/dirty_json.py` | ~200 | Direct reuse — parses LLM JSON output robustly. |
| Tool extraction | `python/helpers/extract_tools.py` | ~100 | Reuse for dynamic check discovery from `.gitclaw/policies/`. |
| Token counting | `python/helpers/tokens.py` | ~50 | Direct reuse. |
| Git helpers | `python/helpers/git.py` | ~150 | Direct reuse — already uses GitPython. |

**Total directly reusable:** ~1,550 lines

### Modules Requiring Major Rework

| Module | Adaptation |
|--------|-----------|
| `agent.py` (1,011 lines) | Gut the interactive loop. Keep `AgentConfig`, prompt assembly, and tool dispatch. Rewrite as single-pass evaluator (~200 lines). |
| `python/helpers/extension.py` | Replace filesystem plugin discovery with GitHub Actions workflow composition. |
| `python/tools/call_subordinate.py` | Replace with matrix job output → dependent job input pattern. |

### Modules That Are Eliminated

Everything in `webui/`, `python/api/`, WebSocket infrastructure, Docker management, SSH, browser agent, TTS/STT, tunnel system, file browser — roughly 15,000+ lines.

---

## 9. Dependency Reduction

### Agent Zero's `requirements.txt`: 50+ packages

### .GITCLAW on GitHub needs:

```txt
# .gitclaw/requirements.txt — minimal governance dependencies
litellm>=1.0.0          # LLM provider abstraction
faiss-cpu>=1.11.0       # Vector store for precedent memory
sentence-transformers   # Embedding model
tiktoken>=0.8.0         # Token counting
python-dotenv>=1.0.0    # Environment loading
gitpython>=3.1.0        # Git operations
pyyaml>=6.0             # Config parsing
requests>=2.31.0        # GitHub API calls
```

**8 packages** vs. Agent Zero's 50+. No Flask, no WebSocket, no Playwright, no Docker SDK, no Whisper, no Kokoro.

---

## 10. Architecture Comparison: Before & After

```
┌─────────────────────────────────────────────────────────┐
│                    AGENT ZERO (Current)                   │
│                                                           │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐ │
│  │ Web UI   │  │ WebSocket│  │ Flask API│  │ Docker   │ │
│  │ (browser)│  │ (real-   │  │ (75+     │  │ container│ │
│  │          │  │  time)   │  │  endpts) │  │ (Kali)   │ │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘ │
│       └──────────────┼──────────────┘             │       │
│                      ▼                            │       │
│              ┌──────────────┐                     │       │
│              │   Agent Loop │◄────────────────────┘       │
│              │  (persistent │                             │
│              │   stateful)  │                             │
│              └──────┬───────┘                             │
│                     │                                     │
│    ┌────────────────┼────────────────┐                    │
│    ▼                ▼                ▼                    │
│ ┌──────┐     ┌──────────┐     ┌──────────┐               │
│ │ LLM  │     │  Memory  │     │  Tools   │               │
│ │      │     │  (FAISS) │     │ (23 tools│               │
│ └──────┘     └──────────┘     └──────────┘               │
└─────────────────────────────────────────────────────────┘

                          ▼ ▼ ▼

┌─────────────────────────────────────────────────────────┐
│               .GITCLAW on GitHub Infrastructure           │
│                                                           │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐               │
│  │ GitHub   │  │ GitHub   │  │ GitHub   │               │
│  │ Webhooks │  │ PR UI    │  │ Secrets  │               │
│  │ (events) │  │ (output) │  │ (config) │               │
│  └────┬─────┘  └────▲─────┘  └────┬─────┘               │
│       │              │             │                      │
│       ▼              │             ▼                      │
│  ┌──────────────────────────────────────┐                │
│  │        GitHub Actions Workflow        │                │
│  │                                       │                │
│  │  ┌─────────┐   ┌───────────────────┐ │                │
│  │  │Orchestr.│──►│ Matrix: Parallel  │ │                │
│  │  │ (Job 1) │   │ Governance Checks │ │                │
│  │  └─────────┘   │ (Jobs 2..N)      │ │                │
│  │                 └────────┬──────────┘ │                │
│  │                          ▼            │                │
│  │                 ┌─────────────────┐   │                │
│  │                 │   Consensus     │   │                │
│  │                 │   (Final Job)   │   │                │
│  │                 └────────┬────────┘   │                │
│  └──────────────────────────┼────────────┘                │
│                             │                             │
│    ┌────────────────────────┼──────────────┐              │
│    ▼                        ▼              ▼              │
│ ┌──────┐            ┌──────────┐    ┌──────────┐         │
│ │ LLM  │            │  Memory  │    │  Repo    │         │
│ │(API) │            │ (cached  │    │  (.git-  │         │
│ └──────┘            │  FAISS)  │    │   claw/) │         │
│                     └──────────┘    └──────────┘         │
└─────────────────────────────────────────────────────────┘
```

---

## 11. Final Verdict

### Can Agent Zero run on GitHub as .GITCLAW?

**As-is: No.** Agent Zero is a 33,000-line interactive server application. It requires a persistent Docker container, a web server, WebSocket connections, and stateful shell sessions. None of these exist on GitHub Actions.

**Re-architected: Yes, and it's a better fit than the original.** By extracting ~1,550 lines of directly reusable code (template engine, LLM wrapper, memory system, JSON parser, git helpers) and re-implementing the agent loop as a single-pass evaluator within GitHub Actions workflows, .GITCLAW becomes:

| Property | Agent Zero | .GITCLAW on GitHub |
|----------|-----------|-------------------|
| Lines of code | ~33,000 | ~3,000–4,000 |
| Dependencies | 50+ packages | 8 packages |
| Infrastructure | Self-hosted Docker + Web UI | Zero infrastructure (GitHub provides everything) |
| Scaling | Manual (one container) | Automatic (GitHub scales runners) |
| Security | Self-managed container isolation | GitHub-managed runner isolation |
| Audit trail | Log files (ephemeral) | Git history + PR comments (permanent, tamper-evident) |
| Cost | Self-hosted compute + LLM API | GitHub Actions minutes + LLM API |
| Setup time | Docker install + configuration | Add `.gitclaw/` directory + set secrets |

### The Bottom Line

Agent Zero's **theories** are GitHub-native in disguise:
- **Prompts** = repo files (already in a git repo)
- **Multi-agent** = parallel Actions jobs (already how CI/CD works)
- **Memory** = git history + cached artifacts (already how repos store state)
- **Extensions** = workflow composition (already how Actions are extended)
- **Isolation** = ephemeral runners (already how Actions provides security)

The irony is that GitHub infrastructure is a *better* home for .GITCLAW's governance use case than Agent Zero's Docker-based runtime. GitHub provides the event system, the compute, the storage, the secrets management, the audit trail, and the user interface — all as managed services. .GITCLAW just needs to bring the intelligence layer (LLM integration, policy templates, and precedent memory), which is exactly the ~1,550 lines of Agent Zero code worth keeping.
