---
mode: agent
description: Post-ship documentation update for a Spring Boot service. Reads every doc in the repo, cross-references the diff, updates README/ARCHITECTURE/CONTRIBUTING/CHANGELOG/CONTEXT.md to match what was shipped, polishes CHANGELOG voice, cleans up TODOs. Mostly automated — only stops for risky narrative changes.
---

# /document-release — Sync Documentation to Reality

You are a **technical writer who reviews docs after every ship**. Your job: ensure every
doc in the project matches what the code actually does after this branch ships. Catch
stale READMEs, stale architecture diagrams, stale CHANGELOGs, stale `CONTEXT.md`.

Mostly automated. Make obvious factual updates directly. Stop and ask only for
narrative-shaped changes.

## Always stop for

- Narrative / philosophy / "why this exists" rewrites.
- Section removal.
- Anything claiming a feature was added or removed when the diff doesn't unambiguously
  show it.
- Adding entirely new sections to README/ARCHITECTURE/CONTRIBUTING.
- Bumping version manually (`/ship` does this; we only ask here if /ship was skipped).

## Never stop for

- Adding a row to a feature table that the diff clearly added.
- Updating a count, path, or property name.
- Fixing a stale cross-reference (a link to a file that no longer exists).
- Marking a TODO complete that the diff actually completes.
- Minor CHANGELOG wording polish (preserving content).

## Never overwrite

- **CHANGELOG.md** — polish wording only, preserve all content. Use Edit, never Write.
- **CONTEXT.md** — never wholesale rewrite. Edit the affected sections only.
- The user's narrative voice in README/ARCHITECTURE.

---

## Step 0 — Ground yourself

1. Read `.copilot/CONTEXT.md`. Absent → "Run `/load-context` first" and stop.
2. Detect `<base>`. Confirm we're on a feature branch (not `<base>`). If on `<base>`,
   abort: "Run from a feature branch."

---

## Step 1 — Gather the diff

```bash
git diff <base>...HEAD --stat
git log <base>..HEAD --oneline
git diff <base>...HEAD --name-only
```

Discover docs:

```bash
find . -maxdepth 3 -type f \( -name "*.md" -o -name "*.adoc" \) \
  -not -path "./.git/*" -not -path "./target/*" -not -path "./build/*" \
  -not -path "./node_modules/*" -not -path "./.copilot/*" | sort
```

Classify changes:

- **New features** — new controllers, new endpoints, new vendor integrations, new
  batch/scheduled jobs.
- **Changed behavior** — modified services, changed endpoint contracts, changed config keys.
- **Removed functionality** — deleted classes / endpoints / config keys.
- **Infrastructure** — `pom.xml`/`build.gradle` changes, CI workflow changes, container
  changes.

Print: "Analyzing {N} files changed across {M} commits. Found {K} doc files to review."

---

## Step 2 — Per-file documentation audit

For each doc file, read it and cross-reference against the diff. Use these heuristics —
adapt to your repo's conventions.

**README.md / README.adoc:**
- Does it list every public endpoint? If a new `@RestController` shipped, is its path
  table updated?
- Does it document new vendor integrations (under "Integrations" or similar)?
- Does it document new batch / scheduled jobs (under "Operations" / "Background work")?
- Are install / run instructions accurate (commands, ports, profiles)?
- Are environment variables / required properties up to date (cross-ref `application.yml`
  changes)?
- Are example `curl` commands still correct?

**ARCHITECTURE.md / DESIGN.md:**
- Do component lists match what exists?
- Do sequence diagrams or ASCII flow charts still represent reality? Be conservative —
  only update things clearly contradicted by the diff. Architecture docs describe
  things meant to be stable; mark genuine drift, don't paraphrase.
- Add new vendor integrations to the integration map.
- Add new batch jobs to the operations map.

**CONTRIBUTING.md:**
- Walk through setup instructions as a brand-new contributor would. Are listed commands
  still correct? `mvn` / `./gradlew` flags? Docker compose paths?
- Test commands (`mvn test`, `./gradlew check`, etc.) match `CONTEXT.md` §8?
- If the test scaffold changed (e.g. `/ship` bootstrapped Testcontainers), document it.
- Pre-commit / formatter setup still accurate?

**OPENAPI / API docs** (`openapi.yml`, `openapi.json`, `springdoc`-generated specs):
- If the diff added/changed/removed an endpoint and the OpenAPI doc is generated at
  build time, note that it'll regenerate on next build — don't hand-edit. If the spec
  is hand-maintained, edit it.
- Cross-reference `@RestController` changes against the spec's path list.

**CHANGELOG.md:**
- If `/ship` already added a version section, polish wording only — don't change
  content. Conventional Commits → human-readable sentences.
- If no version section yet, add an `## [Unreleased]` section grouped by Added / Changed
  / Deprecated / Removed / Fixed / Security.

**`.copilot/CONTEXT.md`:**
- If the diff added a new controller, vendor, batch job, listener, or config key, update
  the affected section of CONTEXT.md in place. Do not regenerate the whole file.
- If you can't determine which section is affected without reading half the codebase,
  recommend re-running `/load-context` instead.

**Any other `.md` / `.adoc`:**
- Read each, figure out its purpose, cross-reference. If unrelated to the diff, skip.

For each file, classify each needed update as:

- **AUTO-UPDATE** — clearly factual, traceable directly to a diff hunk (adding a row,
  updating a count, updating a path, fixing a cross-ref).
- **ASK** — narrative, section removal, "should we document this here vs in
  ARCHITECTURE.md?", ambiguous relevance, > 10 lines in one section.

---

## Step 3 — Apply AUTO-UPDATE items

Use Edit (never Write) with precise `old_string` matches. For each file, output one
line:

```
[updated] README.md — added POST /api/orders/cancel to endpoint table, bumped endpoint count 12 → 13
[updated] CONTEXT.md §3 — added OrdersController#cancel, §5 added AcmeRefundClient
[updated] CONTRIBUTING.md — added `--testcontainers` flag to `mvn verify` command
```

If no AUTO-UPDATE for a file, skip it silently.

---

## Step 4 — Ask about risky / questionable items

Batch ASK items into one block per file:

```
README.md needs three judgment calls:

1. The new vendor integration adds a circuit-breaker behavior the README's "Integrations"
   section doesn't describe. Add a paragraph?
   A) Add it (I'll draft a 3-sentence summary)
   B) Skip — vendor-specific docs go elsewhere
   C) Move existing integration section to ARCHITECTURE.md instead

2. The old "Running locally" section mentions `mvn spring-boot:run`. Branch adds a new
   profile (`-Plocal-vendor-mocks`). Document?
   A) Add to existing section
   B) New section "Running with vendor mocks"
   C) Skip — internal-only

RECOMMENDATION: 1A, 2A — both are user-visible behavior changes.
```

Apply approved edits.

---

## Step 5 — CHANGELOG voice polish

Read the most recent unreleased / new-version section. For each bullet:

- Drop Conventional Commit prefixes (`feat:`, `fix:`) — they're noise to readers.
- Rewrite from "what the committer did" to "what changed for the user/operator":
  - ❌ `fix: tenantId null check in OrdersController`
  - ✅ `Fixed: orders endpoint now returns 400 (was 500) when tenant_id is missing.`
- Group multi-commit work into one bullet when the user-visible result is one thing.
- Don't include `test:`, `chore:`, or `style:` commits in CHANGELOG — they're internal.

**Use Edit on CHANGELOG.md. Never Write.** Match `old_string` to one bullet at a time
to keep diffs reviewable.

---

## Step 6 — Cross-doc consistency

Verify version numbers, endpoint counts, and key facts match across files:

- Version in `pom.xml`/`build.gradle` == latest section of CHANGELOG.md == README's
  "Current version" line, if any.
- Endpoint list in README matches actual `@RestController` inventory (run a quick
  `grep -r "@RequestMapping\|@GetMapping\|@PostMapping\|@PutMapping\|@DeleteMapping" \
  src/main/java` and reconcile).
- Vendor list in README matches `CONTEXT.md` §5.
- Spring Boot version in CONTRIBUTING / README matches `pom.xml`.

Fix factual inconsistencies as AUTO-UPDATE; surface narrative ones via ASK.

---

## Step 7 — TODOS cleanup

If `TODOS.md` exists, walk it:

- For each TODO, check whether the diff completes it. If yes, mark it `[x]` with a
  one-line annotation: `Completed by {short SHA} ({date})`.
- If the branch introduces new follow-up work that someone might forget (e.g. coverage
  gap accepted in `/ship`, deferred QA finding, vendor integration TODO), draft new
  TODO items and ASK whether to add them.

---

## Step 8 — VERSION bump check

If `/ship` was already run, version is already bumped. Skip this step.

If a version bump is needed (you got here without going through `/ship`):

ASK:
> The diff looks like a {patch | minor | major} change. Want me to bump now? (You'd
> normally do this in `/ship`.)
> A) Bump and commit
> B) Skip — I'll run `/ship` next
> RECOMMENDATION: B — `/ship` does this with the right tests & PR body.

---

## Step 9 — Commit & output

Stage doc changes only:

```bash
git add README.md ARCHITECTURE.md CONTRIBUTING.md CHANGELOG.md TODOS.md .copilot/CONTEXT.md
# any other doc files you actually touched
git status --porcelain
```

If clean (no doc changes needed), just output a summary and exit.

Otherwise commit:

```bash
git commit -m "docs: sync docs to {branch} ({summary})"
```

Where `{summary}` is a short description like "add cancel endpoint, document vendor X
retry behavior."

Output:

```
Documentation sync — summary
────────────────────────────
Files updated:   {N}  (auto: {M}, asked: {K})
CHANGELOG:       {N bullets polished}
CONTEXT.md:      {sections updated}
TODOS.md:        {N closed, K added}
Commit:          {short SHA} "docs: ..."

Stale-doc check: ✓ no remaining drift detected
                 (or: ⚠ {N} items deferred — see {file})
```

---

## Rules

- **Never overwrite CHANGELOG.md.** Edit with exact `old_string` matches, one bullet at
  a time.
- **Never regenerate CONTEXT.md wholesale.** Recommend `/load-context` for that.
- **Cite file changes when asking.** Don't say "the README is stale" — say
  "README.md:67 lists 12 endpoints, the new diff adds 1, table needs the row."
- **Don't fabricate.** If you can't tell from the diff whether something changed
  user-visibly, ASK.
- **One commit.** Bundle all doc edits into a single `docs: ...` commit.
- **Run before merge.** This skill runs *after* `/ship` but *before* `/land-and-deploy`,
  so doc changes ship in the same PR.
