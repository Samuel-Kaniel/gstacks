# Copilot prompts for a Spring Boot backend

Customized port of [Garry Tan's gstack](https://github.com/garrytan/gstack) skills
for a **Spring Boot backend** that calls external vendor APIs and runs batch / scheduled
jobs. Drops into a repo as GitHub Copilot prompt files; invoke each as `/promptname`
in Copilot Chat (VS Code).

## What's in here

```
.github/
  copilot-instructions.md          ← global Copilot instructions for the repo
  prompts/
    load-context.prompt.md         ← NEW (prereq) — maps your app into .copilot/CONTEXT.md
    review.prompt.md               ← /review        — pre-PR staff-engineer review
    qa.prompt.md                   ← /qa            — find bugs, fix them, generate regression tests
    qa-only.prompt.md              ← /qa-only       — same methodology, report only (no code changes)
    cso.prompt.md                  ← /cso           — security audit (OWASP + STRIDE, Spring-Boot-aware)
    ship.prompt.md                 ← /ship          — run tests, bump version, push, open PR
    land-and-deploy.prompt.md      ← /land-and-deploy — merge, deploy, verify production health
    document-release.prompt.md     ← /document-release — sync docs to what shipped
    design-consultation.prompt.md  ← /design-consultation — backend system design (produces DESIGN.md)
    plan-design-review.prompt.md   ← /plan-design-review — rate a plan 0-10 on architectural dimensions
    plan-devex-review.prompt.md    ← /plan-devex-review  — review the developer/operator experience
    autoplan.prompt.md             ← /autoplan      — run design + devex reviews with auto-decisions
    learn.prompt.md                ← /learn         — manage .copilot/learnings.jsonl
```

## Install

1. Copy the `.github/` folder into your repo's root, preserving the structure.
2. Open the repo in VS Code with GitHub Copilot enabled.
3. In Copilot Chat, type `/` — the 13 commands should appear.

## First-time use

**Run `/load-context` first.** Every other prompt expects `.copilot/CONTEXT.md` to
exist. It maps your app — controllers, vendors, batch jobs, persistence, deployment —
into a single structured document the other prompts read instead of re-deriving the
app shape every time.

After that, pick the workflow that matches what you're doing:

| You want to… | Prompt |
|---|---|
| Sanity-check a diff before you push it | `/review` |
| Have the assistant exercise the service and fix what's broken | `/qa` |
| Just get a bug list, no code edits | `/qa-only` |
| Security audit | `/cso` |
| Open a PR (tests, version bump, CHANGELOG, PR body) | `/ship` |
| Merge, deploy, and verify production | `/land-and-deploy` |
| Update docs after shipping | `/document-release` |
| Design a new vendor integration / batch job / API surface | `/design-consultation` |
| Pressure-test a plan before implementation | `/plan-design-review` + `/plan-devex-review`, or `/autoplan` for both |
| See what the prompts have learned across sessions | `/learn` |

## What was changed from gstack

Each prompt is a faithful port of the corresponding gstack skill, with these substitutions:

- **All gstack infrastructure stripped** — no `~/.claude/skills/gstack/bin/*` calls, no
  `gbrain` hooks, no `bun`/`bash`-only preambles, no telemetry, no plan-mode handshakes,
  no OpenClaw spawn detection, no skill-routing dashboards.
- **All UI / browser sections rewritten** — the `$B` browse binary and `$D` designer
  binary are gone. `/qa` exercises HTTP endpoints, batch jobs, and listeners via
  REST Assured / MockMvc / Testcontainers / WireMock instead of a real browser.
  `/cso` does code analysis, never live requests.
- **Spring-Boot-aware checks** — `/review` flags `@Transactional` self-invocation,
  N+1, vendor calls inside transactions, `@Scheduled` overlap; `/cso` flags Snakeyaml
  unsafe constructor, Jackson defaultTyping, Log4Shell, SpEL injection, mass assignment,
  actuator exposure beyond `health/info`, BCrypt vs MD5/SHA1; `/ship` runs `mvn verify`
  or `./gradlew check`, uses JaCoCo for diff coverage; `/land-and-deploy` hits
  `/actuator/health/liveness`, `/actuator/health/readiness`, `/actuator/info`.
- **A new `/load-context` prerequisite** — gstack assumes the assistant figures out
  app shape per invocation. With Copilot, that's expensive context every time. This
  skill writes `.copilot/CONTEXT.md` once, every other prompt reads it.
- **Cross-skill state is in the repo, not the user's home directory** — gstack writes
  to `~/.gstack/projects/<slug>/...`. These prompts use `.copilot/` in the repo:
  `CONTEXT.md`, `learnings.jsonl`, `reviews/`, `security-reports/`, `qa-reports/`,
  `deploy-history.jsonl`.
- **Atomic Conventional Commits** preserved — every `fix(qa): ...`, every
  `chore(release): ...`, every `docs: ...` stays as gstack designed.

## Directory the prompts write to

Add `.copilot/` to your `.gitignore` if you don't want the artifacts checked in.
Recommended: keep `CONTEXT.md` tracked (it benefits the whole team) and ignore the rest:

```gitignore
# .gitignore
.copilot/*
!.copilot/CONTEXT.md
```

## License

These prompts are derivative of [gstack](https://github.com/garrytan/gstack) — see its
LICENSE for terms. The Spring Boot adaptations are MIT.
