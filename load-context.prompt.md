---
mode: agent
description: Build a structured understanding of this Spring Boot application by reading docs, build files, configs, controllers, batch jobs, and vendor clients. Writes .copilot/CONTEXT.md, which every other prompt in this folder relies on. Run this once, then re-run whenever the app shape materially changes.
---

# /load-context — Map the Application

You are a **senior engineer joining this codebase on day one**. Your job is to read what is
already here — README, wiki references, `pom.xml`/`build.gradle`, Spring configuration,
controllers, batch jobs, vendor API integrations, and tests — and produce a single,
structured summary the rest of the prompts will rely on.

The deliverable is `.copilot/CONTEXT.md`. It is the source of truth for every other prompt
in `.github/prompts/`. Do not skip it. Do not summarize it. Be specific, cite files.

---

## Step 0 — Decide whether to re-run

1. If `.copilot/CONTEXT.md` already exists, read it.
2. Run `git log -1 --format=%H .copilot/CONTEXT.md` to see when it was last updated.
3. Run `git log --since="<that date>" --name-only --pretty=format:` and check whether any
   of these have changed since: `pom.xml`, `build.gradle*`, `src/main/resources/application*.yml`,
   `src/main/resources/application*.properties`, anything under `src/main/java/.../config/`,
   any `*Controller.java`, any `*Job.java` or `*Scheduler.java`, any `*Client.java` under
   a vendor/integration package, the `README.md`, or anything under `docs/`.
4. If nothing material has changed, ask the user whether they want to refresh anyway. If
   they say no, print the existing context summary and stop.
5. Otherwise continue.

---

## Step 1 — Read the documentation surface

Read whatever exists. Skip anything that doesn't.

- `README.md`, `README.adoc`
- `docs/` (recurse — prioritize files named `architecture`, `overview`, `getting-started`,
  `runbook`, `integrations`, `vendors`, `batch`, `scheduler`)
- `CONTRIBUTING.md`, `CONTRIBUTING.adoc`
- `CHANGELOG.md`
- `ARCHITECTURE.md` / `DESIGN.md`
- Any `*.md` or `*.adoc` at the repo root
- `.github/` workflows (for CI/deploy understanding)
- If a wiki URL is referenced in README, list it as `WIKI:` in the output but do not try
  to fetch it (we may not have network) — ask the user to paste relevant sections.

For each documented item, note: **what it claims** vs. **whether the code agrees**.

---

## Step 2 — Read the build & runtime configuration

Detect Maven vs Gradle:

- `pom.xml` present → Maven. Read `<groupId>`, `<artifactId>`, `<version>`, `<properties>`
  (especially `<java.version>` and `<spring-boot.version>`), and the `<dependencies>` block.
- `build.gradle` / `build.gradle.kts` present → Gradle. Read `group`, `version`, the
  `plugins {}` block, `java.toolchain`, and `dependencies {}`.

Extract:

- Java version (e.g. 17, 21)
- Spring Boot version (e.g. 3.2.x)
- Spring Cloud / Spring Batch / Spring Integration presence
- Database driver(s): `postgresql`, `mysql`, `mariadb`, `mssql`, `oracle`, `h2`
- Migration tool: Flyway or Liquibase
- HTTP client: `spring-web` (`RestClient`/`WebClient`), `spring-cloud-openfeign`, OkHttp,
  Apache HttpClient
- Resilience libraries: Resilience4j, Spring Retry, Bulkhead
- Messaging: Kafka, RabbitMQ, JMS, AWS SQS
- Cache: Redis (Lettuce/Jedis), Caffeine, Hazelcast
- Observability: Micrometer, Prometheus, OpenTelemetry, Sleuth/Brave
- Auth: Spring Security, OAuth2 client/resource server, JWT
- Testing: JUnit version, AssertJ, Mockito, Testcontainers (which modules), REST Assured,
  Spring Cloud Contract, WireMock
- Build/CI: GitHub Actions, GitLab CI, Jenkins — list workflow files

Read `src/main/resources/application*.yml` / `*.properties` for every profile. List each
profile and what it overrides. **Redact** any value that looks like a secret (anything
matching `password`, `secret`, `key`, `token`, `credential`) — emit `***REDACTED***`
in the output. Note where the secret is actually pulled from (env var, Vault, AWS Secrets
Manager, Kubernetes secret).

---

## Step 3 — Map the application shape

### 3a. Entry point

Find the class annotated `@SpringBootApplication`. Note its package — that's the base
package for component scanning. Record any `@EnableXxx` annotations on it
(e.g. `@EnableBatchProcessing`, `@EnableScheduling`, `@EnableFeignClients`,
`@EnableJpaRepositories`, `@EnableKafka`).

### 3b. Inbound surface — REST controllers

Find every `@RestController` (or `@Controller` + `@ResponseBody`). For each, list:

- Path prefix from class-level `@RequestMapping`
- Each endpoint: HTTP method, path, method name, request body type, response type,
  whether it requires auth (look at `@PreAuthorize`, `@Secured`, `SecurityFilterChain`)
- Whether it talks to a vendor (cross-reference with §3d)

### 3c. Scheduled & batch jobs

- `@Scheduled` methods: cron expression / fixed delay, method, what it does
- Spring Batch jobs: `Job` beans, their `Step`s, the `ItemReader` / `ItemProcessor` /
  `ItemWriter` chain, the `JobLauncher` trigger (scheduled? on-demand? Kafka-triggered?)
- Quartz jobs if present
- `@KafkaListener` / `@RabbitListener` / `@JmsListener` / `@SqsListener` — list them with
  topic/queue + handler method

### 3d. Outbound — vendor API clients

Find every outbound HTTP integration. Look for:

- Classes ending in `Client`, `Adapter`, `Gateway`, `Connector`
- `@FeignClient` declarations
- Beans of `RestClient`, `WebClient`, `RestTemplate`
- Direct `HttpClient` / `OkHttpClient` / `URLConnection` use

For each vendor, record:

- **Vendor name** (from class name, config key, or comment)
- **Base URL** (from `application.yml` — note the property key, e.g.
  `vendor.foo.base-url`)
- **Auth scheme** (API key in header, Bearer, mTLS, OAuth2 client_credentials, HMAC)
- **Endpoints called** (method + path)
- **Resilience** (retry, circuit breaker, timeout values, fallback)
- **Idempotency** (do calls include an idempotency key? are responses cached?)
- **Failure handling** (what happens on 4xx vs 5xx vs network error)
- **Webhook receivers** for this vendor, if any (signature verification?)

### 3e. Persistence

- JPA `@Entity` classes — group by aggregate
- Spring Data repositories
- Native SQL or `jdbcTemplate` use
- Flyway/Liquibase migrations — count them, list the latest 5 by version
- Connection pool config (HikariCP defaults? overridden?)

### 3f. Cross-cutting

- Global exception handlers (`@RestControllerAdvice`)
- Filters / interceptors (auth, MDC/correlation-ID, rate limiting)
- Metrics tags and custom Micrometer meters
- Logback / log4j2 configuration — log level per package, structured logging?

---

## Step 4 — Map the test surface

- Where unit tests live (`src/test/java/...`)
- Whether `@SpringBootTest` slice tests exist (`@WebMvcTest`, `@DataJpaTest`, `@JsonTest`,
  `@RestClientTest`)
- Testcontainers usage (which images — `postgres`, `kafka`, `localstack`, vendor mocks)
- WireMock / MockServer usage (for vendor stubs)
- Contract tests (Spring Cloud Contract, Pact)
- Coverage tool (JaCoCo? threshold set in pom/gradle?)
- How to run all tests, fast tests only, integration tests only — the exact commands

---

## Step 5 — Map the deployment surface

- Container: `Dockerfile`, `docker-compose.yml`, multi-stage build?
- Orchestration: Kubernetes manifests, Helm chart, Argo CD app, Kustomize?
- Deploy target: cloud (AWS / GCP / Azure / on-prem)? Specific service (ECS, EKS,
  Cloud Run, App Engine, OpenShift)?
- Health endpoints exposed (`/actuator/health`, `/actuator/info`, `/actuator/metrics`)
- Liveness vs. readiness probes
- Where logs go (stdout → aggregator? CloudWatch? ELK? Splunk?)
- Where metrics go (Prometheus scrape? StatsD?)

If anything is unclear, **ask the user one focused question** rather than guessing.

---

## Step 6 — Write `.copilot/CONTEXT.md`

Create the file with this exact structure (substitute real values; omit sections that
truly don't apply, but never substitute "TBD" — ask the user instead):

```markdown
# Application Context

_Generated by `/load-context` on {ISO date}. Re-run when material shape changes._

## 1. Identity
- **Name:** {artifactId / project name}
- **Purpose:** {1–2 sentences from README}
- **Owner / team:** {if stated}
- **Repository:** {origin URL}
- **Wiki:** {URL, or "paste sections in here when needed"}

## 2. Stack
- **Java:** {version}
- **Spring Boot:** {version}
- **Build:** {Maven|Gradle} — {command to run app locally}
- **Database:** {driver + version}, migrations via {Flyway|Liquibase}
- **HTTP client:** {RestClient|WebClient|Feign|...}
- **Messaging:** {Kafka|Rabbit|...|none}
- **Cache:** {Redis|Caffeine|none}
- **Resilience:** {Resilience4j|Spring Retry|none}
- **Observability:** {Micrometer + Prometheus|OTel|...}
- **Auth:** {Spring Security mode}

## 3. Inbound Surface
{Table: METHOD | PATH | CONTROLLER#method | AUTH | NOTES}

## 4. Scheduled & Batch
### 4a. @Scheduled
{Table: SCHEDULE | METHOD | PURPOSE}
### 4b. Spring Batch jobs
{For each: Job → Steps → Reader/Processor/Writer → Trigger}
### 4c. Message listeners
{Table: TOPIC/QUEUE | LISTENER | HANDLER}

## 5. Outbound Vendor Integrations
{For each vendor, the §3d block}

## 6. Persistence
- **Aggregates:** {list}
- **Migrations:** {count}, latest = V{n}__{name}
- **Connection pool:** {settings}

## 7. Cross-cutting
- **Global exception handler:** {class, behavior}
- **Filters / interceptors:** {list}
- **Correlation ID:** {how it flows}
- **Logging:** {pattern, levels}

## 8. Testing
- **Run all:** `{cmd}`
- **Run unit only:** `{cmd}`
- **Run integration only:** `{cmd}`
- **Coverage gate:** {pct, tool}
- **Testcontainers:** {modules in use}
- **WireMock / MockServer:** {yes/no, base port}
- **Contract tests:** {yes/no, framework}

## 9. Deployment
- **Container:** {Dockerfile present? base image?}
- **Orchestration:** {k8s / ecs / ...}
- **Deploy target:** {env names, URLs}
- **Health:** {endpoints}
- **Probes:** {liveness, readiness paths}
- **Logs go to:** {sink}
- **Metrics go to:** {sink}

## 10. Known Conventions
- **Commit style:** {Conventional Commits? Other?}
- **Branching:** {trunk-based? GitFlow? main/develop?}
- **Base branch:** {main|master|develop}
- **PR / MR template:** {present? path?}
- **Code style:** {Checkstyle? Spotless? google-java-format?}

## 11. Open Questions
{Anything you could not determine from code or docs. Ask the user.}
```

Make sure section 11 is non-empty if there were any guesses. Better to surface uncertainty
than to bake a wrong assumption into every future prompt.

---

## Step 7 — Output and follow-up

After writing the file:

1. Print the file path.
2. Print sections 1, 2, and 11 (Identity, Stack, Open Questions) inline so the user can
   sanity-check immediately.
3. If section 11 is non-empty, ask the user to clarify those items, then re-edit
   `CONTEXT.md` to remove them.
4. Suggest the natural next step:
   - "If you're about to plan something, run `/plan-design-review` on the plan."
   - "If you're about to ship, run `/review`."
   - "If you want a quick security pass, run `/cso`."

## Important rules

- **Don't fabricate.** If a property, vendor, or behavior is not in the code or docs,
  put it in section 11 — never invent.
- **Redact secrets.** Even in section 2 — show the property key and where the value comes
  from, not the value.
- **Cite files.** Every claim in `CONTEXT.md` should be traceable to a file path on
  request. Keep an internal map while writing, then drop the citations into a hidden
  `<!-- citations -->` comment block at the end of `CONTEXT.md` so subsequent prompts
  can verify them.
- **Don't over-summarize.** If the app has 40 endpoints, list all 40 in section 3
  (a table is fine). Subsequent prompts read this file to know what exists.
