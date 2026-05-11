# Copilot Skills Setup

A generic guide and automation script for setting up [Copilot Skills](https://docs.github.com/copilot) with a personal/team skills repo, working both **locally (Copilot CLI)** and on the **GitHub.com cloud agent**.

---

## 🤖 Instructions for Copilot CLI (run this gist as a prompt)

> **Copilot:** if you are reading this file as a user prompt, follow the steps in this section to automate the setup. Use `ask_user` for the inputs marked with 📝. Skip any step whose result is already in place.

### Inputs to gather (use `ask_user` once, with all fields)

1. 📝 **GitHub username or org** that will own the skills repo (e.g. `markusrt`)
2. 📝 **Skills repo name** — default `copilot-skills`
3. 📝 **Visibility** — `public` (recommended, simpler) or `private`
4. 📝 **Baseline skills to install** (multi-select) from <https://github.com/mattpocock/skills>:
   - `tdd` (engineering) — test-driven development workflow
   - `diagnose` (engineering) — structured debugging
   - `zoom-out` (engineering) — step back and rethink
   - `prototype` (engineering) — quick exploratory builds
   - `grill-me` (productivity) — challenge your reasoning
   - `caveman` (productivity) — strip ideas to essentials
   - `write-a-skill` (meta) — author new skills
5. 📝 **Generate a personal `unit-test-naming` skill?** (yes/no, default yes). If yes, ask:
   - **Preferred naming convention** (single-select), with one example each:
     - `MethodName_StateUnderTest_ExpectedBehavior` (Roy Osherove, common in C#/.NET) — `Sum_WithNegativeNumbers_ThrowsException`
     - `MethodName_ExpectedBehavior_WhenStateUnderTest` — `Sum_ThrowsException_WhenGivenNegativeNumbers`
     - `Should_ExpectedBehavior_When_StateUnderTest` — `Should_ThrowException_When_GivenNegativeNumbers`
     - `Given_Precondition_When_Action_Then_Result` (BDD-style) — `Given_NegativeNumbers_When_Sum_Then_ThrowsException`
     - `When_StateUnderTest_Expect_ExpectedBehavior` — `When_GivenNegativeNumbers_Expect_Exception`
     - `it_should_<behavior>_when_<state>` (Jest/Mocha BDD) — `it('should throw an exception when given negative numbers')`
     - `test_<method>_<scenario>` (snake_case, Python/JUnit-style) — `test_sum_with_negative_numbers`
     - **Plain sentence** (Jest `describe`/`it` natural language) — `it('throws an exception for negative numbers')`
     - **Custom** — let the user type their own pattern + one example
   - **Test structure preference** (single-select):
     - **Arrange-Act-Assert** with blank lines between sections (most common)
     - **Given-When-Then** comments (BDD)
     - **No mandated structure** — just clear test logic
   - **Languages this convention applies to** (multi-select, free text or pick from): C#, Java, Kotlin, JavaScript, TypeScript, Python, Go, Rust, Ruby, all
   - **Any extra rules?** (free-text, optional) — e.g., "never test private methods", "one assertion per test", "use FluentAssertions", "use AAA blank lines", "no `[Fact]`-only — prefer `[Theory]` with data"
6. 📝 **Wire up `copilot-setup-steps.yml` in the current repo?** (yes/no) — only meaningful if cwd is a git repo

### Steps to perform

1. **Check if the skills repo exists** on GitHub (`gh repo view <owner>/<name>`).
   - If not, create it: `gh repo create <owner>/<name> --<visibility> --description "Personal Copilot skills" --add-readme`
2. **Check if `~/.copilot/skills` exists** and is a git clone of the repo.
   - If it exists but is **not** a git repo (i.e. user already has loose local skills), see step 3 — migrate first, don't clone over them.
   - If it doesn't exist or is empty: `git clone https://github.com/<owner>/<name>.git ~/.copilot/skills` (use `$env:USERPROFILE\.copilot\skills` on Windows)
   - If already cloned: `git pull` to refresh
3. **Migrate pre-existing local skills** (if `~/.copilot/skills` already had skills before this setup):
   - List every skill folder found and ask the user which ones to migrate to the new repo (default: all).
   - **Scan each selected skill for risky content before staging it:**
     - Possible secrets: regex match for `ghp_`, `github_pat_`, `sk-`, `xox[abps]-`, `AKIA`, `AIza`, lines containing `password`, `secret`, `token`, `api[_-]?key`, `bearer`, private-key headers (`-----BEGIN`).
     - Local-machine references: absolute paths (`C:\Users\...`, `/home/...`, `/Users/...`, `D:\...`), hostnames like `localhost:<port>`, IP addresses, machine-specific env paths.
     - Personal identifiers: email addresses, full names, internal/company URLs.
   - **For every match, print a clear warning** showing the skill name, file, line number, and matched snippet. **Do not auto-redact.** Ask the user to confirm migration of that skill, edit it first, or skip it.
   - Move (don't copy) confirmed-clean skills into a temp staging area, replace `~/.copilot/skills` with a fresh clone of the new repo, then move the skills back into the cloned working tree.
   - Skills the user chose to skip remain in their original location (warn them it's outside the repo and will not be synced or available to the cloud agent).
4. **Install baseline skills** chosen by the user:
   - Fetch each skill folder from `mattpocock/skills` (preserving the whole directory — many skills have companion files like `tests.md`, `mocking.md`)
   - Copy into `~/.copilot/skills/<skill-name>/` (flatten — drop the `engineering/`, `productivity/` prefix)
   - Skip any skill that already exists locally (don't overwrite migrated or user-customized skills)
5. **Generate the `unit-test-naming` skill** (if user opted in):
   - Skip if `~/.copilot/skills/unit-test-naming/SKILL.md` already exists (don't overwrite).
   - Otherwise write a `SKILL.md` using the template in [§ Generated `unit-test-naming` skill](#generated-unit-test-naming-skill) below, substituting the user's chosen convention, example, structure preference, languages, and extra rules.
6. **Commit and push** new and migrated skills to the skills repo. Use a descriptive commit message distinguishing migrated vs. baseline vs. generated additions.
7. **If wiring up `copilot-setup-steps.yml`** and cwd is a git repo:
   - Skip if `.github/workflows/copilot-setup-steps.yml` already exists.
   - Otherwise create it with the template in [§ Cloud agent setup](#cloud-agent-setup) below, replacing `<owner>/<name>` with the user's values.
   - Stage but do NOT commit — let the user review.
8. **Print a summary** of what was done, which skills were migrated, which were skipped (and why), and what the user still needs to do manually (e.g., add secrets to the `copilot` environment if any skill requires them — see [§ Secrets](#secrets-for-skills-that-need-them)).

If any step fails (e.g., `gh` not authenticated, no network), report the failure clearly and continue with the remaining independent steps.

---

## Concepts

### The three customization layers

| Layer | File | When loaded | Use for |
|---|---|---|---|
| **Instructions** | `.github/copilot-instructions.md` (repo) or `~/.copilot/copilot-instructions.md` (global) | **Always** — every interaction | Domain knowledge, tech stack, conventions, build commands |
| **AGENTS.md** | `AGENTS.md` (repo root or any directory) | **Always** in scope | Cross-tool agent rules (Claude, Gemini, Copilot all read this) |
| **Skills** | `~/.copilot/skills/<name>/SKILL.md` (global) or `.github/skills/<name>/SKILL.md` (repo) | **On demand** — when description matches your prompt, or invoked with `/skill-name` | Multi-step workflows, methodologies, detailed procedures |

**Rule of thumb:**
- Always-true facts → **instructions**
- Methodology you invoke when you need it → **skill**
- Custom agent personas in `.github/agents/` → use sparingly; most things fit better as instructions or skills

### How Copilot picks a skill

The `description` field in a skill's YAML frontmatter is the trigger. Copilot reads your prompt, compares it against all loaded skill descriptions, and injects matching skills.

```yaml
---
name: tdd
description: Test-driven development workflow. Use when writing new code, adding features, or fixing bugs that need test coverage.
---
```

Three triggering paths:

1. **Automatic** — description matches your prompt
2. **Explicit** — you type `/skill-name`
3. **Manual toggle** — `/skills` to enable/disable per session

**Tip:** write descriptions starting with **"Use when..."** — that's exactly how Copilot evaluates them.

---

## Why a dedicated skills repo

| | Local CLI | VS Code | GitHub.com cloud agent |
|---|:---:|:---:|:---:|
| `~/.copilot/skills/` | ✅ | ✅ | ❌ (cloud has no access to your machine) |
| `.github/skills/` (in repo) | ✅ | ✅ | ✅ |
| **Public skills repo + `copilot-setup-steps.yml`** | ✅ (via local clone) | ✅ | ✅ (cloned at session start) |

A **public skills repo** + `copilot-setup-steps.yml` is the cleanest single-source-of-truth for solo developers and small teams. No submodule machinery, no per-repo updates, fresh skills on every cloud agent run.

> **Recommend:** keep the skills repo **public**. Skills are prompts, not code — there's rarely anything sensitive. Public means no auth required from any environment.

---

## Setup

### Step 1 — Create the skills repo

```bash
gh repo create <owner>/copilot-skills --public --description "Personal Copilot skills" --add-readme
```

### Step 2 — Clone locally (once per machine)

**Windows (PowerShell):**
```powershell
git clone https://github.com/<owner>/copilot-skills.git "$env:USERPROFILE\.copilot\skills"
```

**macOS / Linux:**
```bash
git clone https://github.com/<owner>/copilot-skills.git ~/.copilot/skills
```

### Step 3 — Migrate any existing local skills

If `~/.copilot/skills/` already contained skills from before, migrate them into the new repo so they're versioned and available to the cloud agent.

**Before committing any pre-existing skill, scan it for content that should not be pushed to a (potentially public) repo:**

| Risk | What to look for |
|---|---|
| **Secrets** | Tokens (`ghp_`, `github_pat_`, `sk-`, `xox[abps]-`, `AKIA`, `AIza`), API keys, passwords, bearer tokens, `-----BEGIN ... PRIVATE KEY-----` blocks |
| **Local paths** | Absolute paths like `C:\Users\<name>\...`, `/home/<user>/...`, `/Users/<name>/...` — these break on other machines and leak usernames |
| **Local references** | `localhost:<port>`, internal IP addresses, machine-specific environment paths |
| **Personal identifiers** | Email addresses, full names, company-internal URLs you don't want public |

**Recommended workflow:**

```bash
# scan everything before pushing
cd ~/.copilot/skills
grep -rIEn 'ghp_|github_pat_|sk-[A-Za-z0-9]{20,}|AKIA[0-9A-Z]{16}|-----BEGIN|password|secret|api[_-]?key|bearer' .
grep -rIEn 'C:\\\\Users|/home/[^/]+|/Users/[^/]+|localhost:[0-9]+|127\.0\.0\.1' .
```

**For each match, decide:**

- **Replace with an env var reference.** E.g., turn `token: ghp_abc123...` in a skill into "use the token from the `MY_GITHUB_PAT` env var" (see [§ Secrets](#secrets-for-skills-that-need-them)).
- **Generalize the path.** Replace `C:\Users\markus\projects\foo` with `<your project root>` or document it as a required input.
- **Skip the skill.** If a skill is too tied to your machine, leave it outside the repo (it'll still work locally but won't sync to the cloud agent).

Once clean, push:

```bash
cd ~/.copilot/skills
git add .
git commit -m "feat: migrate existing local skills"
git push
```

> **The Copilot automation script in [§ Instructions for Copilot CLI](#-instructions-for-copilot-cli-run-this-gist-as-a-prompt) does this scan automatically and warns you per-finding before staging anything.**

### Step 4 — Add baseline skills

Pick a starting set from [mattpocock/skills](https://github.com/mattpocock/skills) and copy whole skill folders into the root of your local clone — flat, one folder per skill:

```text
~/.copilot/skills/
  tdd/
    SKILL.md
    tests.md         ← companion files come with the skill
    mocking.md
  diagnose/SKILL.md
  zoom-out/SKILL.md
  grill-me/SKILL.md
```

Then commit and push:

```bash
cd ~/.copilot/skills
git add .
git commit -m "feat: add initial curated skills"
git push
```

### Step 5 — Verify in Copilot CLI

```text
/skills reload
/skills list
/skills info tdd
```

### Step 6 — Wire up the GitHub.com cloud agent <a id="cloud-agent-setup"></a>

In each repo where you want the cloud agent to have access to your skills, create `.github/workflows/copilot-setup-steps.yml`:

```yaml
name: "Copilot Setup Steps"

on:
  workflow_dispatch:
  push:
    paths:
      - .github/workflows/copilot-setup-steps.yml
  pull_request:
    paths:
      - .github/workflows/copilot-setup-steps.yml

jobs:
  copilot-setup-steps:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - name: Clone shared Copilot skills
        run: git clone https://github.com/<owner>/copilot-skills.git ~/.copilot/skills
```

> The job MUST be named `copilot-setup-steps` and the file MUST be on the default branch for the cloud agent to pick it up.

For **private** skills repos, replace the clone step with:

```yaml
      - name: Clone shared Copilot skills
        env:
          GH_TOKEN: ${{ secrets.SKILLS_REPO_PAT }}
        run: |
          git clone https://x-access-token:${GH_TOKEN}@github.com/<owner>/copilot-skills.git ~/.copilot/skills
```

…and add `SKILLS_REPO_PAT` as a secret in the repo's `copilot` environment (see [§ Secrets](#secrets-for-skills-that-need-them)).

### Step 7 — Repo-specific skills (optional)

For a skill that only makes sense in one project, drop it directly in the repo as a plain directory — no submodule, no extra setup:

```text
.github/skills/
  my-project-deploy/SKILL.md
```

Both `~/.copilot/skills/` and `.github/skills/` are loaded and merged.

---

## Secrets for skills that need them <a id="secrets-for-skills-that-need-them"></a>

Some skills need credentials (a GitHub PAT for cross-repo issue reads, an API key for an external service, etc.). The pattern is:

> **The skill references an environment variable by name. The secret value lives outside the skill.**

In your `SKILL.md` (public, safe to commit):

```markdown
## Authentication

Use the GitHub API to fetch the issue. Authenticate with the token in the
`MY_GITHUB_PAT` environment variable. If the variable is unset, ask the user
to set it before continuing.
```

The skill **never contains the value** — only the variable name.

### Local machine (Copilot CLI)

Set the variable in your shell profile, once per machine:

**Windows (PowerShell `$PROFILE`):**
```powershell
$env:MY_GITHUB_PAT = "ghp_..."
```

**macOS / Linux (`~/.zshrc`, `~/.bashrc`):**
```bash
export MY_GITHUB_PAT="ghp_..."
```

### GitHub.com cloud agent

Add the value as a secret in the **`copilot` environment** of each repo (or once at the org level for org-wide repos):

1. Repo → **Settings → Environments → `copilot`** (create if missing)
2. **Add environment secret** → name: `MY_GITHUB_PAT`, value: `ghp_...`

The cloud agent has automatic access to all secrets in the `copilot` environment — no changes to `copilot-setup-steps.yml` needed.

> **Shortcut for current-repo access:** if the skill only needs to read issues/PRs in the same repo, you don't need a PAT at all. The cloud agent has `$GITHUB_TOKEN` injected automatically. Only cross-repo or elevated-permission scenarios need a custom PAT.

### Org-wide secrets

For team setups, define the secret once at the org level: **Org Settings → Secrets and variables → Actions → `copilot` environment**. It propagates to all repos automatically — no per-repo configuration.

---

## Generated `unit-test-naming` skill <a id="generated-unit-test-naming-skill"></a>

When the bootstrap script generates this skill, it writes the file below into `~/.copilot/skills/unit-test-naming/SKILL.md`, substituting the placeholders in `{{ ... }}` with the user's choices.

```markdown
---
name: unit-test-naming
description: Unit test naming and structure conventions. Use when writing, reviewing, refactoring, or generating unit tests in {{LANGUAGES}}.
---

## Naming convention

Test methods follow the pattern:

**`{{CONVENTION_PATTERN}}`**

Example:

\`\`\`
{{CONVENTION_EXAMPLE}}
\`\`\`

Apply this pattern consistently. When asked to add or refactor tests, rename
existing tests to match if they don't already.

## Structure

{{STRUCTURE_INSTRUCTIONS}}

## Additional rules

{{EXTRA_RULES_BULLETS}}

## When to apply

- Writing new unit tests
- Reviewing or refactoring existing tests
- Generating tests for newly added code
- Renaming tests that don't follow the convention
```

### Substitutions

| Placeholder | Source | Notes |
|---|---|---|
| `{{LANGUAGES}}` | Languages multi-select | Comma-separated, or `all languages` if "all" was chosen |
| `{{CONVENTION_PATTERN}}` | Naming convention choice | The literal pattern, e.g. `MethodName_StateUnderTest_ExpectedBehavior` |
| `{{CONVENTION_EXAMPLE}}` | Naming convention choice | The example shown to the user during selection |
| `{{STRUCTURE_INSTRUCTIONS}}` | Structure choice | Expand to a paragraph: see mapping below |
| `{{EXTRA_RULES_BULLETS}}` | Free-text extra rules | Split user input by lines/sentences into bullet points; if empty, write `_None._` |

### Structure choice → instructions

| Choice | Substitute |
|---|---|
| Arrange-Act-Assert | `Use the **Arrange-Act-Assert** pattern. Separate each section with a blank line. Do not add comments labelling the sections — the blank lines make them obvious.` |
| Given-When-Then | `Use the **Given-When-Then** pattern with `// Given`, `// When`, `// Then` comments delimiting each section.` |
| No mandated structure | `No mandated structure. Keep tests linear and easy to read.` |

### Why a generated skill (and not just `copilot-instructions.md`)?

- It's **portable across all your repos** — instructions are per-repo, but a global skill applies everywhere you write tests.
- It's **on-demand** — only injected when Copilot detects a test-related task, keeping context lean.
- It's **invocable** — type `/unit-test-naming` to apply it deliberately even when the description doesn't auto-match.

If you want the convention enforced for **one specific repo** instead of globally, drop the same SKILL.md into that repo's `.github/skills/unit-test-naming/SKILL.md` and don't generate the global one.

---

## Migrating existing `.github/agents/` files

With skills available, most custom agent personas in `.github/agents/` can and should be split into:

| Content type | Move to | Why |
|---|---|---|
| Domain knowledge / project facts | `.github/copilot-instructions.md` | Always relevant — should never be absent |
| Coding conventions / commit style | `.github/copilot-instructions.md` | Project-wide rules |
| Methodologies (TDD, clean code, refactoring patterns) | Global skill in `~/.copilot/skills/` | Reusable across projects |
| Workflows specific to one repo | Repo skill in `.github/skills/` | Scoped, on-demand |
| Specialist personas you deliberately invoke | Keep in `.github/agents/` | Rare — only when there's truly a distinct persona to switch into |

Most existing agent files are doing several of these jobs at once. Splitting them clarifies intent and makes the rules portable.

> **Note on `/plan`:** the built-in `/plan` command in Copilot CLI replaces most "implementation planner" agents. Delete those agents and put any output-format preferences into `copilot-instructions.md` so `/plan` picks them up automatically.

---

## Updating skills

```bash
cd ~/.copilot/skills
# add / edit skill folders
git add .
git commit -m "feat: add new skill"
git push
```

- **Local**: run `/skills reload` in Copilot CLI, or pull on each machine when convenient.
- **Cloud agent**: nothing to do — the next session clones the latest commit automatically.

---

## Quick reference

| Task | Command |
|---|---|
| List loaded skills | `/skills list` |
| Inspect a skill | `/skills info <name>` |
| Reload skills | `/skills reload` |
| Disable a skill for this session | `/skills disable <name>` |
| Invoke a skill explicitly | `/<skill-name>` in your prompt |
| Update local skills | `cd ~/.copilot/skills && git pull` |
