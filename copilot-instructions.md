# GitHub Copilot — Project Instructions

This repository uses a set of Copilot **prompt files** (in `.github/prompts/`) adapted from
[Garry Tan's gstack](https://github.com/garrytan/gstack). They give Copilot Chat a set of
role-based slash commands for a **Spring Boot backend** that consumes external vendor APIs
and runs batch / scheduled jobs (no UI).

## How to use

In Copilot Chat (VS Code), type `/` and pick one of:

| Command | Role | When to use |
|---|---|---|
| `/load-context` | Onboarding | **Run first.** Reads README, wiki, `pom.xml`/`build.gradle`, `application.yml`, controllers, batch jobs, vendor clients → writes `.copilot/CONTEXT.md`. Every other command relies on it. |
| `/review` | Staff Engineer | Pre-PR review of the current diff. Finds bugs that pass CI but blow up in production. |
| `/plan-devex-review` | DevEx Reviewer | Reviews a plan/design for a developer-facing surface (API, SDK, CLI). |
| `/design-consultation` | System Designer | Backend system-design consultation (no UI). Produces `DESIGN.md`. |
| `/plan-design-review` | Senior Designer (API/system) | Rates a plan 0–10 across architectural design dimensions, then improves it. |
| `/qa` | QA Lead | Finds bugs in the running app (API + batch), fixes them with atomic commits, generates regression tests. |
| `/qa-only` | QA Reporter | Same methodology as `/qa` but report only — no code changes. |
| `/cso` | Chief Security Officer | OWASP Top 10 + STRIDE, with Spring Boot–aware false-positive filtering. Read-only. |
| `/ship` | Release Engineer | Run tests, audit coverage, push, open PR. Bootstraps a test framework if none exists. |
| `/land-and-deploy` | Release Engineer | Merge the PR, wait for CI and deploy, verify production health. |
| `/document-release` | Technical Writer | Updates README/ARCHITECTURE/CONTRIBUTING/CHANGELOG to match what was just shipped. |
| `/autoplan` | Review Pipeline | One command. Runs design → eng → DX reviews on a plan with auto-decisions. |
| `/learn` | Memory | Manage `.copilot/learnings.jsonl` — patterns, pitfalls, and preferences captured across sessions. |

## Shared conventions

These hold for every prompt in this folder:

- **Context first.** If `.copilot/CONTEXT.md` does not exist, the prompt should run
  `/load-context` (or ask the user to) before doing real work. Do not re-derive the
  application shape on every invocation.
- **No gstack binaries.** None of these prompts call `~/.claude/skills/gstack/bin/*`,
  `gbrain`, `bun`, `$B` (browse), or `$D` (designer). They were stripped during port.
- **No browser/UI assumptions.** This is a backend codebase. `/qa` and `/cso` test
  via HTTP requests, MockMvc, REST Assured, Testcontainers, batch-job execution, and
  log/metric inspection — never via a browser.
- **Java / Spring Boot defaults.** Build with Maven (`mvn`) or Gradle (`./gradlew`),
  test with JUnit 5 + Mockito + Spring Boot Test, integration-test with Testcontainers,
  schedule via `@Scheduled` / Spring Batch, integrate vendors via `RestClient` / `WebClient` /
  Feign. If the codebase uses something else, `CONTEXT.md` overrides these defaults.
- **No assumptions about a UI.** Skip every UI/design/visual-mockup section. Where a
  gstack skill would generate visual mockups, these prompts generate sequence diagrams,
  request/response shapes, and API contracts instead.
- **Atomic commits.** One fix = one commit. Conventional Commits style
  (`fix(scope): ...`, `feat(scope): ...`, `chore(scope): ...`).
- **Don't fabricate.** If a fact (a version, a class name, a config key) can't be
  verified from the codebase or context file, say so. Never invent.
- **Learnings file.** When a prompt instructs you to capture a learning, append a
  JSON line to `.copilot/learnings.jsonl` (the lightweight replacement for gstack's
  per-project store). Schema lives in `/learn`.

## Directory layout

```
.copilot/
  CONTEXT.md              # produced by /load-context — application shape
  learnings.jsonl         # append-only learnings (one JSON object per line)
  reviews/                # /review outputs
  security-reports/       # /cso outputs
  qa-reports/             # /qa and /qa-only outputs
  release-notes/          # /document-release drafts
.github/
  copilot-instructions.md # this file
  prompts/
    load-context.prompt.md
    review.prompt.md
    plan-devex-review.prompt.md
    design-consultation.prompt.md
    plan-design-review.prompt.md
    qa.prompt.md
    qa-only.prompt.md
    cso.prompt.md
    ship.prompt.md
    land-and-deploy.prompt.md
    document-release.prompt.md
    autoplan.prompt.md
    learn.prompt.md
```

Add `.copilot/` to `.gitignore` if you don't want artifacts checked in — but keep
`CONTEXT.md` tracked (it benefits the whole team).
