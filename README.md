# Test Arbiter

Your AI QA engineer. Test Arbiter handles the full testing lifecycle — finding gaps in your coverage, suggesting what tests to write, executing your test suites, triaging failures so you know what's a real bug versus noise, and healing tests that break when your code changes.

One install, two ways to use it:

- **CLI** — run `qa-agent <command>` from your terminal or CI. Talks directly to an AI provider (Anthropic, OpenAI, Gemini, or Bedrock) using your own API key.
- **MCP server** — run `qa-agent mcp` and let your AI assistant (Claude Code, Cursor, Copilot, Claude Desktop) call Test Arbiter's tools straight from chat. No separate API key needed — the assistant does the reasoning, since it's already authenticated.

Jump to the guide for how you're using it — each is written to stand on its own:

- [→ Using Test Arbiter as a CLI](#using-test-arbiter-as-a-cli)
- [→ Using Test Arbiter as an MCP Server](#using-test-arbiter-as-an-mcp-server)

---

## Install

```bash
npm install -g test-arbiter
```

Requires Node 18+. This single package provides both the `qa-agent` CLI and the `qa-agent mcp` server — install it once, use it either way.

---

<!-- tab:cli:start -->

## Using Test Arbiter as a CLI

Run `qa-agent <command>` from your terminal or a CI pipeline. The CLI calls an AI provider directly using your own API key to do gap analysis, generate tests, triage failures, heal broken tests, and more.

### Setup

```bash
npm install -g test-arbiter
```

Requires Node 18+.

### Initialize your project

```bash
cd your-project
qa-agent config init
```

Detects jest/vitest/playwright/mocha from your `package.json` scripts and writes `.test-arbiter/config.yaml` wired up with JUnit output, prompting you to accept, edit, or skip each suite it finds. Nothing detected? It falls back to a starter template you can edit by hand — see [Config file](#config-file-test-arbiterconfigyaml) below for the full structure.

### Set up your AI provider

Test Arbiter calls an AI provider (Anthropic, OpenAI, or Gemini) to do its analysis. Set whichever one you already have — it auto-detects:

```bash
export ANTHROPIC_API_KEY=sk-ant-...   # Anthropic Claude
export OPENAI_API_KEY=sk-...          # OpenAI GPT
export GEMINI_API_KEY=...             # Google Gemini
```

If more than one is set, Anthropic is used first, then OpenAI, then Gemini. Override with `--provider`.

No API key handy — only an AWS account? Use **Bedrock** instead. It authenticates via your ambient AWS credentials (profile, SSO, IAM role) rather than a key, so it must be selected explicitly — it's never auto-detected:

```bash
qa-agent triage --provider bedrock
# or set it once:
# provider: bedrock
# aws_region: us-east-1   # optional; defaults to AWS_REGION/AWS_DEFAULT_REGION
```

Test Arbiter picks the model to match the job instead of always reaching for the most expensive one — cheap/fast for a quick read, a stronger model for reading and writing code, and the strongest for judgment calls like triage verdicts, where being wrong is costly:

| Job | Commands | Anthropic | OpenAI | Gemini | Bedrock |
|---|---|---|---|---|---|
| **fast** — quick reads | `summarize` | `claude-haiku-4-5` | `gpt-4o-mini` | `gemini-2.0-flash` | `anthropic.claude-haiku-4-5` |
| **code** — reading/writing code | `gaps`, `generate`, `heal` | `claude-sonnet-5` | `gpt-4o` | `gemini-2.0-flash` | `anthropic.claude-sonnet-5` |
| **reasoning** — triage verdicts | `test`, `run-regression`, `triage`, `watch` | `claude-opus-5` | `gpt-4o` | `gemini-2.0-flash` | `anthropic.claude-opus-5` |

Override any of these with `-m, --model` per command, or set `model:` in `.test-arbiter/config.yaml` to force one model everywhere.

> **Note:** you don't have to `export` these — Test Arbiter automatically loads `.env.local` and `.env` from your current directory if present (checked in that order). A real exported shell variable always takes priority over both files, and `.env.local` takes priority over `.env`.

Or skip the export and pass a key explicitly per command:
```bash
qa-agent gaps --provider openai --api-key sk-... --model gpt-4o
```

> Using Test Arbiter through your AI assistant instead? Skip this section entirely — the [MCP server](#using-test-arbiter-as-an-mcp-server) never needs a key of its own.

### Quick start

Verify the install works with a throwaway sample JUnit file — no project setup needed:

```bash
cat > /tmp/sample-results.xml <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<testsuites>
<testsuite name="tests.api" tests="4" failures="1" errors="0" skipped="0" time="0.892">
  <testcase name="test_list_users" classname="tests.api.TestAPI" time="0.112"/>
  <testcase name="test_get_user" classname="tests.api.TestAPI" time="0.098"/>
  <testcase name="test_create_user" classname="tests.api.TestAPI" time="0.445">
    <failure message="AssertionError: Expected 201, got 500">
      tests/api/test_api.py:55: AssertionError
      AssertionError: Expected status 201, got 500
      Response body: {"error": "duplicate key value violates unique constraint users_email_key"}
    </failure>
  </testcase>
  <testcase name="test_delete_user" classname="tests.api.TestAPI" time="0.237"/>
</testsuite>
</testsuites>
EOF

qa-agent run-regression --junit /tmp/sample-results.xml --project myapp --branch main
```

Then, in your own project (after [Initialize your project](#initialize-your-project) above):

```bash
cd your-project

# Get an instant AI overview: tech stack, test coverage, structure
qa-agent summarize

# Find untested code and get suggested tests (--src . checks the whole project root)
qa-agent gaps --src .

# Run your test suite and triage any failures
qa-agent test
```

### Config file (`.test-arbiter/config.yaml`)

Run `qa-agent config init` to generate one — it detects jest/vitest/playwright/mocha from your `package.json` scripts and proposes a `command` + `junit_glob` (and reporter env vars, where the runner needs them) per suite, prompting you to accept, edit, or skip each. It also offers to install a missing JUnit reporter (e.g. `jest-junit`) as a dev dependency. In a non-interactive shell (CI) or with `--yes`, it accepts everything detected without prompting; with nothing detected, it falls back to a static starter template.

Most commands work fine without a config file (using defaults and flags), but it lets you define suites, per-project overrides, and a model override once instead of passing flags every time. Full structure:

```yaml
version: 1
project: myapp
model: claude-opus-5   # optional — omit to use each provider's default
store_dir: .test-arbiter   # default; override to change where run history/gap reports are stored
api_url: https://api.testarbiter.com   # reserved for the hosted dashboard — no effect until it ships

suites:
  unit:
    command: "npx jest --reporters=default --reporters=jest-junit"
    junit_glob: "test-results/unit/*.xml"
    env:                                   # optional — passed to the command's environment
      JEST_JUNIT_OUTPUT_DIR: test-results/unit
  e2e:
    junit_glob: "test-results/e2e/*.xml"
    branch: main

projects:
  payments:
    suites:
      unit:
        junit_glob: "packages/payments/test-results/*.xml"
```

### Commands

#### `qa-agent config`

```bash
qa-agent config init             # detect test runners, prompt, write/merge .test-arbiter/config.yaml
qa-agent config init --yes       # accept all detected suites, no prompts, no dep installs
qa-agent config init --force     # regenerate from scratch instead of merging
qa-agent config show             # show resolved config
```

---

#### `qa-agent summarize`

Get an instant AI-powered overview of any project — useful for onboarding, auditing a new codebase, or getting a quick read on test health before diving into gap analysis.

```bash
# Summarize the current project
qa-agent summarize

# Analyze from the repo root instead of src/
qa-agent summarize --src .

# Machine-readable output
qa-agent summarize --output json
```

**Output**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Test Arbiter — Project Summary
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  payments-api  ·  REST API

  A Node.js API for processing payments via Stripe, with webhook handling
  and subscription management.

  Tech Stack
  ──────────────────────────────────────────────────
  Languages:    TypeScript
  Runtime:      Node.js 20
  Frameworks:   Express, Prisma
  Testing:      Jest
  Databases:    PostgreSQL
  Key deps:     Stripe, Zod, Winston, bull

  Test Coverage
  ──────────────────────────────────────────────────
  Source files: 34
  Test files:   6
  Coverage:     Partial ~

  Unit tests cover the auth and webhook modules. The payments and
  subscription services have no tests. No integration or e2e tests found.

  Structure
  ──────────────────────────────────────────────────
  src/routes/      HTTP route handlers (8 files)
  src/services/    Business logic — payments, subscriptions (12 files)
  src/lib/         Shared utilities and Prisma client
  src/middleware/  Auth, error handling, rate limiting
  tests/           Jest unit tests

  Observations
  ──────────────────────────────────────────────────
  · Secrets loaded via environment variables — good practice
  · No CI configuration found (no .github/workflows/)
  · 2 service files exceed 400 lines — consider splitting

  Suggested Next Steps
  ──────────────────────────────────────────────────
  → Run qa-agent gaps --src src/services/ to find untested business logic
  → Run qa-agent generate to write tests for critical payment flows
  → Add a GitHub Actions workflow for automated test runs
```

**Options**

| Flag | Description |
|---|---|
| `--src <path>` | Root directory to analyze (default: project root) |
| `--provider <name>` | AI provider: `anthropic`, `openai`, `gemini`, `bedrock` |
| `--api-key <key>` | API key for the chosen provider |
| `-m, --model <model>` | Model override |
| `-o, --output <format>` | `rich` or `json` |

---

#### `qa-agent gaps`

Analyze your source code for test coverage gaps. Claude reads your source files and existing tests, identifies what is untested, ranks gaps by risk, and suggests what tests to write — including the type of test (unit, api, e2e, ui, etc.).

`--src` defaults to `src/`. If your project doesn't have a `src/` directory (e.g. a Next.js app with `app/`, `lib/`, `components/` at the repo root), it'll silently report 0 files — pass `--src` explicitly:

```bash
# Analyze src/ against co-located tests (auto-detected)
qa-agent gaps --src src/

# No src/ directory? Point at the real root, or a specific folder
qa-agent gaps --src .
qa-agent gaps --src app/

# Scope to a specific directory
qa-agent gaps --src src/api/

# Glob pattern
qa-agent gaps --src "src/**/*.ts"

# Explicit test directory
qa-agent gaps --src src/ --tests e2e/

# JSON output
qa-agent gaps --src src/ --output json

# Large repo — smaller chunks per AI request
qa-agent gaps --chunk-size 30

# Cap total source files analyzed
qa-agent gaps --max-files 200
```

**Options**

| Flag | Description |
|---|---|
| `--src <path>` | Source directory or glob (default: `src/`) |
| `--tests <path>` | Test directory or glob (auto-detected if omitted) |
| `--max-files <n>` | Max source files to analyze (default: unlimited) |
| `--chunk-size <n>` | Source files per AI request (default: 50) |
| `-p, --project <name>` | Project name |
| `-m, --model <model>` | Claude model override |
| `-o, --output <format>` | `rich` or `json` |

**Gap report output**

Each gap includes:
- A short id (`g1`, `g2`, …) you can hand to `qa-agent generate --gap-id` to generate just that gap
- File and symbol name (function, class, endpoint, component)
- Risk level: `critical` / `high` / `medium` / `low`
- Description of what the code does
- 2–3 suggested tests with type badges: `[unit]` `[api]` `[e2e]` `[ui]` `[integration]` `[performance]` `[security]`

Source files are split into chunks of up to `--chunk-size` files (default 50) per AI request and merged back into one report — a chunk also ends early if its accumulated content approaches a safe token budget, so a handful of large files don't overflow the model's context window the way a pure file-count limit would. Existing test files (sent as context so the AI knows what's already covered) are similarly capped, prioritizing tests whose name matches a source file in the current chunk — if your project has more than fits, you'll see a note that some were sampled rather than all included.

The report (with ids) is saved to `<storeDir>/gaps/<project>.json` so a later `qa-agent generate --gap-id <id>` can pick specific gaps without re-running analysis.

---

#### `qa-agent generate`

Run gap analysis and immediately write production-ready test files for every gap found. Where the files land follows each framework's own discovery convention (see **Supported frameworks** below) and they're ready to run with your existing test suite.

```bash
# Analyze src/, generate tests for all gaps
qa-agent generate

# Only generate for high-priority gaps
qa-agent generate --risk critical,high

# Force a specific test framework
qa-agent generate --framework vitest

# Write all generated tests to a single directory
qa-agent generate --out tests/generated/

# Preview what would be generated without writing any files
qa-agent generate --dry-run

# Limit to the 10 most critical gaps
qa-agent generate --risk critical --max-gaps 10

# Large repo — smaller chunks per AI request during gap analysis
qa-agent generate --chunk-size 30

# Generate only specific gaps found by a previous `qa-agent gaps` run
qa-agent generate --gap-id g3
qa-agent generate --gap-id g1 --gap-id g4
qa-agent generate --gap-id g1,g4
```

**How it works:**

1. Scans source files and existing tests (same as `qa-agent gaps`)
2. If `--gap-id` is given, loads the report saved by the last `qa-agent gaps` run and picks those gaps — no AI gap analysis is re-run. Otherwise, runs gap analysis with the AI to identify untested code, filters by `--risk` level, sorts critical-first, and saves the report for later `--gap-id` use.
3. Groups gaps by source file; generates one test file per source file
4. Picks the closest existing test files as style references so generated code matches your conventions
5. Writes the file wherever the target framework actually discovers tests from — see **Supported frameworks** below

**Output example**

```
  Test Arbiter — Test Generation

  Project: myapp  Framework: vitest  Gaps: 8  Files: 3

  ✓ src/lib/auth.generated.test.ts
    5 tests · 3 gaps addressed · written
  ✓ src/api/payments.generated.test.ts
    7 tests · 4 gaps addressed · written
  ✓ src/utils/format.generated.test.ts
    2 tests · 1 gap addressed · written

  Review the generated files, adjust as needed, then move them into your test suite.
```

**Options**

| Flag | Description |
|---|---|
| `--src <path>` | Source directory or glob (default: `src/`) |
| `--tests <path>` | Test directory for style reference (auto-detected) |
| `--risk <levels>` | Comma-separated risk levels: `critical,high,medium,low` (default: all) |
| `--gap-id <id>` | Generate only this gap id from a previous `qa-agent gaps` run (repeatable or comma-separated; omit for all gaps, ignores `--risk`/`--max-gaps`) |
| `--framework <name>` | Test framework: `jest`, `vitest`, `playwright`, `pytest`, `mocha`, `cypress` |
| `--out <dir>` | Output directory for generated tests (default: framework-specific — see **Supported frameworks** below) |
| `--max-gaps <n>` | Maximum number of gaps to generate for (default: unlimited) |
| `--chunk-size <n>` | Source files per AI request during gap analysis (default: 50) |
| `--dry-run` | Preview what would be generated without writing files |
| `--provider <name>` | AI provider: `anthropic`, `openai`, `gemini`, `bedrock` |
| `--api-key <key>` | API key for the chosen provider |
| `-m, --model <model>` | Model override |
| `-o, --output <format>` | `rich` or `json` |

The framework is auto-detected from `package.json` dependencies (`vitest`, `jest`, `@playwright/test`, `cypress`, `mocha`) and pytest config files. Use `--framework` to override.

**Supported frameworks**

| Framework | Default output location | File naming | Why |
|---|---|---|---|
| `jest` | Alongside the source file | `<name>.generated.test.ts` | Matches Jest's default `testMatch` (anywhere in the tree) |
| `vitest` | Alongside the source file | `<name>.generated.test.ts` | Matches Vitest's default `test.include` pattern |
| `mocha` | Alongside the source file | `<name>.generated.test.ts` | Matches the `spec:` pattern in the `.mocharc.yml` `qa-agent generate` scaffolds for you |
| `pytest` | Alongside the source file | `test_<name>_generated.py` | Matches pytest's default `test_*.py` / `*_test.py` discovery |
| `playwright` | `tests/` (mirroring the source file's directory underneath) | `<name>.spec.ts` | Playwright discovers specs by `testDir` + a `.spec.ts`-style `testMatch`, not by scanning the whole repo — writing alongside source with a generic suffix would never be picked up |
| `cypress` | `cypress/e2e/` (mirroring the source file's directory underneath) | `<name>.cy.ts` | Matches Cypress's default `specPattern` (`cypress/e2e/**/*.cy.{js,jsx,ts,tsx}`) |

For Playwright and Cypress, the source file's relative directory is preserved underneath the target directory (e.g. `src/app/about/page.tsx` → `tests/src/app/about/page.spec.ts`) so same-named files in different directories — very common with Next.js `page.tsx`/`route.ts` — don't overwrite each other. If your project uses a non-default `testDir`/`testMatch`/`specPattern` (e.g. splitting suites into `tests/ui/` vs `tests/api/`), pass `--out` to place generated files where your config actually looks.

Like `qa-agent gaps`, the gap-analysis step chunks source files by both count and content size, and caps existing test files sent as context the same way, so large repos and large test suites don't blow past the model's context limit.

---

#### `qa-agent heal`

Automatically fix tests that broke because of source code changes — refactors, renamed functions, changed signatures, moved modules. Works from a JUnit XML file so you can heal right after a failing test run.

```bash
# Heal tests based on a failing test run
qa-agent heal --junit test-results/*.xml

# Heal specific test files using git diff for context
qa-agent heal --tests "src/**/*.test.ts"

# Preview what would change without writing files
qa-agent heal --junit results.xml --dry-run

# Include the last 3 commits of source changes as context
qa-agent heal --junit results.xml --since HEAD~3
```

**How it works:**

1. Parses JUnit XML to find failing tests with error messages and stack traces
2. Maps failures back to their test files (via stack trace file paths or test name matching)
3. Finds the corresponding source file for each test file (by name convention)
4. Gets `git diff` between the current state and `--since` (default: `HEAD~1`) for context
5. Asks the AI to update each test file to match the current source — preserving test intent, not just making tests pass
6. Writes the healed files (or shows a preview with `--dry-run`)

**AI guardrail:** if a failure looks like it's catching a real bug rather than a stale test, the model adds `// REVIEW: This test may be catching a real bug — verify before merging` instead of silently patching the assertion. Those files are flagged in the report.

**Output**

```
  Test Arbiter — Test Healing

  Project: myapp   Failures: 12   Files healed: 3

  ✓ src/lib/auth.test.ts — 4 tests healed
    Updated login() call signature to match new AuthOptions param · written

  ✓ src/api/payments.test.ts — 6 tests healed
    Updated processPayment() opts object and response shape · written

  ⚠ src/lib/db.test.ts — 2 flagged for review
    Connection pool test may be catching a real leak — verify before merging · written

  1 file flagged — may be catching real bugs. Review before committing.

  Run your test suite to verify the healed files, then commit.
```

**Options**

| Flag | Description |
|---|---|
| `--junit <glob>` | JUnit XML file(s) with test failures |
| `--tests <glob>` | Heal specific test files directly (uses git diff, no failure context) |
| `--src <path>` | Source directory (default: `src/`) |
| `--since <ref>` | Git ref for diff context (default: `HEAD~1`) |
| `--dry-run` | Show what would change without writing files |
| `--provider <name>` | AI provider: `anthropic`, `openai`, `gemini`, `bedrock` |
| `-m, --model <model>` | Model override |
| `-o, --output <format>` | `rich` or `json` |

---

#### `qa-agent plan`

Turn a PR description or feature spec into a structured test plan — scope, out-of-scope, risks, and concrete test cases grouped by area, each with a type (unit/integration/api/e2e/ui/performance/security) and priority. Useful for reviewers who want a checklist before approving, or for pasting into the PR itself.

Pull the spec from a GitHub PR (via the `gh` CLI — description **and** diff), a local file, inline text, or stdin. When a diff is available, the plan is grounded in the actual code change instead of just the prose description.

```bash
# Plan from a GitHub PR (fetches title, body, and diff via gh)
qa-agent plan --pr 482

# Plan from the current branch's PR
qa-agent plan --pr

# Plan from a local feature spec file
qa-agent plan --file docs/feature-spec.md

# Plan from inline text
qa-agent plan --input "Add SSO login via SAML for enterprise accounts"

# Plan from stdin
cat spec.md | qa-agent plan

# Ground a file-based spec in the actual diff against main
qa-agent plan --file spec.md --diff main

# Markdown output, ready to paste into a PR comment
qa-agent plan --pr 482 --output markdown
```

**Output**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Test Arbiter — Test Plan: SSO Login via SAML
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Adds SAML-based SSO for enterprise accounts, gated behind a new
  org-level setting. Highest risk is session handling during IdP redirects.

  Project: payments-api  Areas: 3  Cases: 11  Model: anthropic/claude-opus-5

  Risks
  ──────────────────────────────────────────────────
  ⚠ IdP metadata parsing isn't covered in the spec — assumed strict XML validation

  Auth Flow (5)
  Login redirect to IdP and callback handling
  ────────────────────────────────────────────────────────────
  CRITICAL  [e2e] User logs in via SAML and lands on dashboard
       1. Click "Sign in with SSO"
       2. Complete IdP login
       3. Return to app
       → User is authenticated with correct org context

  HIGH  [security] Rejects a forged SAML assertion
       ...
```

**Options**

| Flag | Description |
|---|---|
| `--pr [number]` | Fetch a PR's title, body, and diff via the GitHub CLI (omit the number to use the current branch's PR) |
| `--file <path>` | Read the spec from a local file |
| `--input <text>` | Provide the spec as literal text |
| `--diff <ref>` | Include a git diff against this ref as extra context; overrides the PR diff when used with `--pr` |
| `--src <path>` | Root directory for repository context — dir tree, manifest, README (default: project root) |
| `--no-context` | Skip gathering repository context |
| `--provider <name>` | AI provider: `anthropic`, `openai`, `gemini`, `bedrock` |
| `-m, --model <model>` | Model override |
| `-o, --output <format>` | `rich`, `json`, or `markdown` |

`--pr` requires the [GitHub CLI](https://cli.github.com) (`gh`) installed and authenticated (`gh auth login`).

---

#### `qa-agent test`

Run your test suite and immediately triage any failures — all in one command. If no suite has a `command` configured yet and you're in an interactive shell, it offers to run the `qa-agent config init` wizard right there instead of just erroring out.

```bash
# Run all suites that have a command in .test-arbiter/config.yaml
qa-agent test

# Run a specific suite
qa-agent test --suite unit

# One-off without config
qa-agent test --cmd "npm test" --junit "test-results/*.xml"
qa-agent test --cmd "npx vitest run --reporter=junit" --junit "test-results/vitest.xml"
qa-agent test --cmd "pytest --junitxml=results.xml" --junit results.xml
```

**Config** — add a `command` field to your suites in `.test-arbiter/config.yaml`:

```yaml
suites:
  unit:
    command: "npm run test:unit -- --reporter=junit"
    junit_glob: "test-results/unit/*.xml"
  e2e:
    command: "npx playwright test --reporter=junit"
    junit_glob: "test-results/e2e/*.xml"
```

**Options**

| Flag | Description |
|---|---|
| `--cmd <command>` | Command to run (e.g. `npm test`, `pytest`) |
| `--junit <glob>` | JUnit XML path/glob (required with `--cmd`) |
| `-s, --suite <name>` | Run a specific suite from config |
| `--branch <branch>` | Branch name (default: auto-detected from git) |
| `--commit <sha>` | Commit SHA (default: auto-detected from git) |
| `--provider <name>` | `anthropic`, `openai`, `gemini`, or `bedrock` (auto-detected; bedrock requires explicit --provider) |
| `--api-key <key>` | API key for the chosen provider |
| `-m, --model <model>` | Model override |
| `-o, --output <format>` | `rich` (default) or `json` |
| `--no-triage` | Skip AI triage — just run the tests |
| `--no-save` | Skip saving to local store |
| `--no-push` | Skip pushing to dashboard |

> Any runner that outputs JUnit XML works here — see [Test runner setup](#test-runner-setup) below for copy-paste configs.

---

#### `qa-agent run-regression`

Parse JUnit XML results from your test runner, triage every failure with Claude, and save the run locally. Use this instead of `qa-agent test` when your CI already ran the tests and just needs the results triaged.

```bash
# Point directly at JUnit XML
qa-agent run-regression --junit "test-results/**/*.xml" --project myapp

# Via .test-arbiter/config.yaml config (suites defined there)
qa-agent run-regression

# Specific suite only
qa-agent run-regression --suite unit

# JSON output — pipe to jq, post to Slack, etc.
qa-agent run-regression --output json | jq .action_items
```

**Options**

| Flag | Description |
|---|---|
| `--junit <glob>` | Path or glob to JUnit XML files |
| `-p, --project <name>` | Project name (overrides config) |
| `-s, --suite <name>` | Run a single suite from config |
| `--branch <branch>` | Branch name (default: main) |
| `--commit <sha>` | Commit SHA |
| `--provider <name>` | `anthropic`, `openai`, `gemini`, or `bedrock` (auto-detected; bedrock requires explicit --provider) |
| `--api-key <key>` | API key for the chosen provider |
| `-m, --model <model>` | Model override (e.g. `gpt-4o`, `gemini-2.0-flash`) |
| `-o, --output <format>` | `rich` (default) or `json` |
| `--no-save` | Skip saving to local store |
| `--no-push` | Skip pushing to dashboard |
| `-c, --config <path>` | Path to .test-arbiter/config.yaml |

Each run gets a stable ID like `myapp-20260726-143022-a1b2`.

---

#### `qa-agent triage`

Show the triage report for a stored run (from `qa-agent test` or `qa-agent run-regression`). Caches the Claude analysis — use `--re-triage` to force a fresh one.

```bash
qa-agent triage --run-id myapp-20260726-143022-a1b2
qa-agent triage --run-id myapp-20260726        # prefix match works
qa-agent triage --run-id <id> --re-triage      # fresh Claude analysis
qa-agent triage --run-id <id> --output json
```

**Options**

| Flag | Description |
|---|---|
| `--run-id <id>` | Run ID or prefix (required) |
| `--re-triage` | Force fresh Claude analysis |
| `-m, --model <model>` | Claude model override |
| `-o, --output <format>` | `rich` or `json` |

---

#### `qa-agent runs`

List recent runs stored locally.

```bash
qa-agent runs
qa-agent runs --project myapp
qa-agent runs --limit 50 --output json
```

**Options**

| Flag | Description |
|---|---|
| `-p, --project <name>` | Filter by project |
| `-n, --limit <n>` | Max runs to show (default: 20) |
| `-o, --output <format>` | `rich` or `json` |

---

#### `qa-agent flakiness`

Show which tests have been flaky across historical runs. Every time you run `qa-agent test` or `qa-agent run-regression`, results are recorded in a local SQLite database. This command queries that history and surfaces tests ranked by fail rate.

```bash
qa-agent flakiness
qa-agent flakiness --project myapp
qa-agent flakiness --min-runs 5      # only show tests seen at least 5 times
qa-agent flakiness --output json | jq '.[] | select(.failRate > 0.3)'
```

**Flakiness levels**

| Level | Fail rate |
|---|---|
| `suspect` | 1–20% |
| `flaky` | 20–50% |
| `very flaky` | >50% |

**Options**

| Flag | Description |
|---|---|
| `--min-runs <n>` | Minimum runs before a test appears (default: 3) |
| `--limit <n>` | Max tests to show (default: 50) |
| `-p, --project <name>` | Project to report on |
| `-o, --output <format>` | `rich` or `json` |

The database lives at `.test-arbiter/flakiness.db` and is populated automatically on every `qa-agent test` or `qa-agent run-regression` run. Historical flakiness data is also passed to the AI triage prompt so verdicts are more accurate — a test that has failed intermittently across 10 runs will be classified as flaky rather than a real regression.

---

#### `qa-agent watch`

Run in a terminal alongside your test suite. Every time tests complete and write new JUnit XML, Test Arbiter picks it up automatically and triages the failures — no manual commands needed.

```bash
# Watch all suites defined in .test-arbiter/config.yaml
qa-agent watch

# Watch a specific glob
qa-agent watch --junit "test-results/**/*.xml"

# Check every 10 seconds instead of 30
qa-agent watch --interval 10

# Wait for one run, triage it, then exit
qa-agent watch --once
```

**How it works:**

1. On start, records the state of any existing XML files as a baseline
2. Every `--interval` seconds, re-scans the glob pattern
3. Files that are new or have a newer modification time since the last check are picked up
4. Changed files in the same poll cycle are merged into a single triage run
5. Results are displayed immediately and saved to the local store

```
  Test Arbiter — Watch Mode

  Project:  myapp
  Patterns: test-results/**/*.xml
  Interval: 30s
  Model:    anthropic/claude-opus-5

  Found 3 existing files — watching for new results…
  Press Ctrl+C to stop.

  [14:32:05] Detected 2 new/updated files
    test-results/unit/auth.xml
    test-results/unit/payments.xml

  24 tests  3 failed  21 passed
  Triaging 3 failures…

  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Test Arbiter — Triage Report
  ...
```

**Options**

| Flag | Description |
|---|---|
| `--junit <glob>` | JUnit XML glob to watch (default: all suites in config) |
| `--interval <n>` | Poll interval in seconds (default: 30) |
| `--branch <branch>` | Branch name for the run record (default: main) |
| `--once` | Exit after the first triage |
| `--provider <name>` | AI provider: `anthropic`, `openai`, `gemini`, `bedrock` |
| `-m, --model <model>` | Model override |

---

#### `qa-agent auth`

> **Not available yet.** The hosted dashboard is still on the [roadmap](#roadmap) — there's
> no service to connect to and no way to obtain a token, so these commands currently do
> nothing. Every other command works entirely locally and needs no account. Run history is
> stored in `.test-arbiter/` on your machine.

Reserved for connecting to the Test Arbiter hosted dashboard. Once it ships, authenticating
will make each `run-regression` push its results automatically after triage.

```bash
qa-agent auth login --token <token>   # save your dashboard token
qa-agent auth status                  # verify connection
qa-agent auth logout                  # remove saved token
```

### Test runner setup

Any runner that outputs JUnit XML works with `qa-agent test`, `qa-agent run-regression`, and `qa-agent watch`. Below are copy-paste configs for the most common ones.

#### Playwright

```ts
// playwright.config.ts
export default defineConfig({
  reporter: [['junit', { outputFile: 'test-results/results.xml' }]],
})
```

```yaml
# .test-arbiter/config.yaml
suites:
  e2e:
    command: "npx playwright test"
    junit_glob: "test-results/results.xml"
```

#### Vitest

```ts
// vite.config.ts (or vitest.config.ts)
export default defineConfig({
  test: {
    reporters: ['junit'],
    outputFile: 'test-results/vitest.xml',
  },
})
```

```yaml
# .test-arbiter/config.yaml
suites:
  unit:
    command: "npx vitest run"
    junit_glob: "test-results/vitest.xml"
```

Or without a config file:
```bash
qa-agent test --cmd "npx vitest run --reporter=junit --outputFile=test-results/vitest.xml" \
              --junit test-results/vitest.xml
```

#### Jest

Install the reporter:
```bash
npm install -D jest-junit
```

```js
// jest.config.js
module.exports = {
  reporters: [
    'default',
    ['jest-junit', { outputDirectory: 'test-results', outputName: 'jest.xml' }],
  ],
}
```

```yaml
# .test-arbiter/config.yaml
suites:
  unit:
    command: "npx jest"
    junit_glob: "test-results/jest.xml"
```

#### pytest

No extra packages needed:

```yaml
# .test-arbiter/config.yaml
suites:
  unit:
    command: "pytest --junitxml=test-results/results.xml"
    junit_glob: "test-results/results.xml"
```

Or one-off:
```bash
qa-agent test --cmd "pytest --junitxml=test-results/results.xml" \
              --junit test-results/results.xml
```

#### Cypress

```js
// cypress.config.js
module.exports = defineConfig({
  reporter: 'junit',
  reporterOptions: {
    mochaFile: 'test-results/cypress-[hash].xml',
  },
})
```

```yaml
# .test-arbiter/config.yaml
suites:
  e2e:
    command: "npx cypress run"
    junit_glob: "test-results/cypress-*.xml"
```

#### JUnit / Maven

```xml
<!-- pom.xml — surefire outputs XML by default -->
<build>
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-surefire-plugin</artifactId>
    </plugin>
  </plugins>
</build>
```

```yaml
# .test-arbiter/config.yaml
suites:
  unit:
    command: "mvn test"
    junit_glob: "target/surefire-reports/*.xml"
```

### GitHub Actions

Drop Test Arbiter into any CI workflow. After your tests run, it triages failures and posts the report directly as a PR comment — so developers see what's a real bug vs noise without digging into logs.

```yaml
# .github/workflows/test.yml
name: Test

on:
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest

    # Required to post the report as a PR comment — GITHUB_TOKEN is
    # read-only by default, so without this the comment step fails.
    permissions:
      contents: read
      pull-requests: write

    # Set your API key at job level, not on the step.
    env:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}

    steps:
      - uses: actions/checkout@v4

      - name: Run tests
        run: npm test   # configure your runner to output JUnit XML

      - name: Triage with Test Arbiter
        uses: rcsarro/test-arbiter@v1
        with:
          junit: test-results/**/*.xml
          project: myapp
```

The action posts (and updates in place on re-runs) a PR comment like:

> **🤖 Test Arbiter — Triage Report**
>
> 3 failures triaged — 🔴 1 real · 🟡 1 flaky · ⚪ 1 noise
>
> | Severity | Test | Verdict | Action |
> |---|---|---|---|
> | HIGH | `PaymentService.charge` | NullPointerException at refund path | Fix null guard at line 45 |

**Inputs**

| Input | Description | Default |
|---|---|---|
| `junit` | Path or glob to JUnit XML (required) | — |
| `project` | Project name | — |
| `provider` | `anthropic`, `openai`, or `gemini` | auto-detected |
| `model` | Model override | provider default |
| `post-comment` | Post result as PR comment | `true` |
| `github-token` | Token for posting the comment | `github.token` |

**Outputs**

| Output | Description |
|---|---|
| `run-id` | The Test Arbiter run ID |
| `real-failures` | Count of real failures |
| `flaky-tests` | Count of flaky tests |

**Use the outputs** to fail the job only on real bugs (not noise or flaky tests):

```yaml
      # Uses the same job-level env and permissions as the example above
      - name: Triage with Test Arbiter
        id: triage
        uses: rcsarro/test-arbiter@v1
        with:
          junit: test-results/**/*.xml

      - name: Fail on real bugs only
        if: steps.triage.outputs.real-failures > 0
        run: exit 1
```

<!-- tab:cli:end -->

---

<!-- tab:mcp:start -->

## Using Test Arbiter as an MCP Server

Expose Test Arbiter's features as tools your AI assistant (Cursor, Copilot, Claude Code, Claude Desktop) can call directly from chat — no terminal commands needed.

**No API key required.** The MCP tools gather data (source files, test failures, JUnit results) and hand it to whichever assistant is hosting the session — the assistant does the actual reasoning, since it's already authenticated. This is the setup for anyone who only has a Claude Code/Copilot/Cursor subscription login and no separate API key.

### Setup

```bash
npm install -g test-arbiter
```

Requires Node 18+. No AI provider key needed — that's only for [CLI usage](#using-test-arbiter-as-a-cli).

### Quick start

Start the server:

```bash
qa-agent mcp
```

This is a **stdio** server — your AI client launches it itself using the setup snippets below. Running `qa-agent mcp` by hand in a separate terminal won't work; there's nothing for a manually-started process to connect to.

Register it once with your client (Claude Code shown here — see **Setup by client** below for Cursor, Claude Desktop, and VS Code/Copilot):

```bash
claude mcp add --transport stdio test-arbiter -- qa-agent mcp
```

Then just ask, in chat:
> *"Find the test gaps in src/api/"*
> *"Triage the last run for the payments project"*
> *"What tests failed on the main branch today?"*
> *"Write a test plan for this PR"*

### Available MCP tools

| Tool | Description |
|---|---|
| `analyze_gaps` | Gather source/test files + instructions for a coverage-gap analysis (you analyze it) |
| `generate_test_plan` | Gather a PR description/diff or feature spec + repo context + instructions for a test plan (you write it) |
| `triage_run` | Gather failure data + instructions for a stored run (you analyze it), or return the cached report |
| `run_regression` | Parse JUnit XML, then gather failure data + instructions for triage (you analyze it) |
| `save_triage_report` | Persist the verdict you produced, so it's cached for future `list_runs`/`triage_run` calls |
| `list_runs` | List recent local runs |
| `summarize_project` | Gather directory structure/manifest/README/source snippets + instructions for a project overview (you write it) |
| `heal_test` | Gather a failing test file, its matched source file(s), git diff, and (optionally) a stored run's failure messages + instructions for healing it (you fix it, using your own file-editing tools) |
| `generate_tests` | Gather a source file, style-reference tests, and the correct output path for the target framework + instructions for writing test code for the given gaps (you write it, using your own file-editing tools) |

### Setup by client

**Claude Code**

```bash
claude mcp add --transport stdio test-arbiter -- qa-agent mcp
```

**Cursor**

Add to `~/.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "test-arbiter": {
      "command": "qa-agent",
      "args": ["mcp"]
    }
  }
}
```

Then in Cursor chat:
> *"Find the test gaps in src/api/"*
> *"Triage the last run for the payments project"*
> *"What tests failed on the main branch today?"*

**Claude Desktop**

Add to `~/.config/claude/claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "test-arbiter": {
      "command": "qa-agent",
      "args": ["mcp"]
    }
  }
}
```

**VS Code with GitHub Copilot**

Add to `.vscode/mcp.json` in your project:

```json
{
  "servers": {
    "test-arbiter": {
      "type": "stdio",
      "command": "qa-agent",
      "args": ["mcp"]
    }
  }
}
```

### Config file (`.test-arbiter/config.yaml`)

The MCP tools read the same `.test-arbiter/config.yaml` as the CLI, if one exists in the project — suite definitions, `junit_glob` patterns, and per-project overrides all apply. Run `qa-agent config init` from a terminal once to generate it (it detects jest/vitest/playwright/mocha from your `package.json` and proposes suites interactively); after that, everything else can happen from chat.

Full structure, for reference:

```yaml
version: 1
project: myapp
model: claude-opus-5   # optional — omit to use each provider's default
store_dir: .test-arbiter   # default; override to change where run history/gap reports are stored
api_url: https://api.testarbiter.com   # reserved for the hosted dashboard — no effect until it ships

suites:
  unit:
    command: "npx jest --reporters=default --reporters=jest-junit"
    junit_glob: "test-results/unit/*.xml"
    env:                                   # optional — passed to the command's environment
      JEST_JUNIT_OUTPUT_DIR: test-results/unit
  e2e:
    junit_glob: "test-results/e2e/*.xml"
    branch: main

projects:
  payments:
    suites:
      unit:
        junit_glob: "packages/payments/test-results/*.xml"
```

Without a config file, tools like `run_regression` and `triage_run` still work — point your assistant at a JUnit glob directly and it'll pass that through.

### Test runner setup

`run_regression` and `triage_run` both need JUnit XML from your test runner. Below are copy-paste configs for the most common ones — ask your assistant to run the tests via your normal `npm test`/`pytest` command, then point it at the resulting XML.

#### Playwright

```ts
// playwright.config.ts
export default defineConfig({
  reporter: [['junit', { outputFile: 'test-results/results.xml' }]],
})
```

#### Vitest

```ts
// vite.config.ts (or vitest.config.ts)
export default defineConfig({
  test: {
    reporters: ['junit'],
    outputFile: 'test-results/vitest.xml',
  },
})
```

#### Jest

Install the reporter:
```bash
npm install -D jest-junit
```

```js
// jest.config.js
module.exports = {
  reporters: [
    'default',
    ['jest-junit', { outputDirectory: 'test-results', outputName: 'jest.xml' }],
  ],
}
```

#### pytest

No extra packages needed:

```bash
pytest --junitxml=test-results/results.xml
```

#### Cypress

```js
// cypress.config.js
module.exports = defineConfig({
  reporter: 'junit',
  reporterOptions: {
    mochaFile: 'test-results/cypress-[hash].xml',
  },
})
```

#### JUnit / Maven

```xml
<!-- pom.xml — surefire outputs XML by default -->
<build>
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-surefire-plugin</artifactId>
    </plugin>
  </plugins>
</build>
```

<!-- tab:mcp:end -->

---

## Roadmap

- [x] `qa-agent test` — run your test suite and immediately triage results
- [x] Flakiness tracking across runs (SQLite)
- [x] GitHub Actions step — post triage as PR comment
- [x] `qa-agent watch` — poll CI and triage new runs as they complete
- [x] Test healing — auto-fix tests broken by app code changes (`qa-agent heal`)
- [x] `qa-agent generate` — write actual test code from gap analysis results
- [x] `qa-agent summarize` — AI overview of a project: tech stack, test coverage, structure, recommendations
- [ ] PR diff-aware triage — cross-reference failures against the PR diff to identify regressions vs pre-existing issues
- [ ] `qa-agent retry` — re-run only real failures after triage, skipping known flaky and noise
- [ ] Slack / webhook notifications — post triage summaries to Slack, email, or any webhook
- [x] `qa-agent plan` — generate a test plan from a PR description or feature spec
- [ ] Suite health score — single aggregate metric per project tracking gap coverage, flakiness rate, and failure rate over time
- [ ] Duration tracking — alert when tests slow down significantly across runs (performance regression detection)
- [ ] Web dashboard frontend
