---
title: "Development"
summary: "Run Backbone locally: Docker, Floci, tests, and Quarkus."
---

Run Backbone locally: Docker services, Floci, tests, and Quarkus dev mode.

First-time setup (repository, licence, bootstrap) is in [Onboarding](/docs/onboarding).  
For GitHub and AWS setup, CDK, and local metrics, see [Operations](/docs/operations).  
For a comprehensive task index, see [Cheatsheet](/docs/cheatsheet).

## Table of Contents

- [Quick start](#quick-start)
- [Local services](#local-services)
- [Build & Deploy](#build--deploy)
- [Troubleshooting](#troubleshooting)
- [Useful Debugging Commands](#useful-debugging-commands)
- [Cursor Agent Skills](#cursor-agent-skills)

---

## Quick start

To get started, minus the explanatory commentary in further reading below, issue the following commands:

```bash
# local infra
task docker:start -- floci jaeger postgres redis [prometheus grafana]
task dev:floci
task seed:floci

# dev servers
task build:install
mert start

# tests
task test:unit
task seed:it:up
task test:integration
```

## Local services

### 1. Docker backing services

Start required Docker services for local development. Floci emulates AWS on `:4566` (S3, DynamoDB, SES, SSM, Secrets Manager, Cognito, and more).

#### Service management

Services can be stopped, started, and restarted individually or all at once:

```bash
task docker:restart -- floci jaeger postgres
task docker:start -- floci jaeger postgres
task docker:stop -- floci jaeger postgres
```

Service status can be queried with:

```bash
task docker:status
```

#### Provision Floci resources

Create Floci AWS resources (S3, DynamoDB, Cognito pools → SSM/Secrets) and seed development data:

```bash
task dev:floci
task seed:floci
```

### 2. Integration test fixtures

Seed IT fixtures into Floci Cognito and sync actor identities into Postgres `actor.actors` (Cognito holds credentials; Postgres holds actor metadata):

```bash
task seed:it:up
```

**Prerequisites**: Service `actor-service` needs to have been started to create the `ACTORS` table (via Flyway) in Postgres.

```bash
# run all quarkus dev services
mert start
# run actor-service only
task dev:actor
```

To teardown **perf** data only (ephemeral `testuser*` from k6, `loaduser*` from perf TSV, audit truncate, S3)—**not** alice/bob/eve:

```bash
task seed:perf:down
```

---

## Build & Deploy

### 1. Build Tasks

Taskfile provides convenient wrappers around the standard Maven commands:

```bash
# Display all available tasks
task

task build:nuke                                    # Delete the Maven build cache + stop mvn daemons
task build:clean
task build:compile
task build:package
task build:install

task test:unit
task test:integration
task test:integration MODULE=services/auth-service # Run a specific module's integration tests
task test:all                                      # Run all unit and integration tests
task test:static                                   # OWASP(NVD)/PMD/SpotBugs/Checkstyle; OSS Index in CI only
```

#### Code coverage

Weekly CI publishes Clover coverage to Codecov. Local runs only write HTML under `target/site/clover/` (no upload).

[`backbone-kit`](https://github.com/get-backbone/backbone-kit)   [![coverage](https://codecov.io/github/get-backbone/backbone-kit/graph/badge.svg?token=RP8Z2NWG9L)](https://app.codecov.io/github/get-backbone/backbone-kit)
[`backbone-core`](https://github.com/get-backbone/backbone-core)   [![coverage](https://codecov.io/github/get-backbone/backbone-core/graph/badge.svg?token=6AJ2I85TKH)](https://app.codecov.io/github/get-backbone/backbone-core)

```bash
task _clover:unit
task _clover:integration                           # needs Floci + Postgres, same as the weekly job
task _clover:report                                # writes target/site/clover/
```

### 2. Development Runtime

Run services in Quarkus dev mode:

#### Services/Applications/UIs

```bash
mert start                                         # Run all quarkus dev services locally

task dev:audit                                     # Run a single quarkus dev service locally
task dev:auth
task dev:actor
task dev:document
task dev:notification
task dev:bff                                       # actor-bff BFF application
task dev:web                                       # web-actor static/stateless ui
```

#### Quarkus Servers

Check the status of or kill any running Quarkus server processes:

```bash
task quarkus:status
task quarkus:kill
```

For the complete task index, see [cheatsheet.md](/docs/cheatsheet).

---

## Troubleshooting

### 1. Basic Checks

1. **Docker services not starting**
   - Check Docker/OrbStack is running
   - Verify ports are not in use: `task docker:status`

2. **Quarkus port conflicts**
   - Check running Quarkus servers: `task quarkus:status`
   - Kill conflicting processes: `task quarkus:kill`

### 2. Maven Build Errors

1. **Build cache**
   - The [Maven build cache extension](https://maven.apache.org/extensions/maven-build-cache-extension/) is implemented to improve build times
   - This can be unreliable and produce maven build errors, however
   - It is generally resolved with the following clean build command, which stops the maven daemon and clears the cache:

```bash
task build:clean build:nuke build:install
```

---

## Useful Debugging Commands

### 1. Floci

1. notification-service:      ➜ `awslocal ses verify-email-identity --email hello@backbonehq.io` · inspect mail: `curl http://localhost:4566/_aws/ses`
2. document-service:          ➜ `awslocal dynamodb describe-table --table-name DOCUMENTS 2>&1 | grep -A 10 "KeySchema"`
3. authenticate Docker to ECR ➜

```bash
aws ecr get-login-password --region "$AWS_REGION" \
| docker login --username AWS --password-stdin "$(echo $ECR_URI | cut -d/ -f1)"
```

1. actor-service:             ➜ `docker exec -it postgres psql -U postgres -d backbone -c "DELETE * FROM actor.actors;"`
                              ➜ `POOL_ID=$(awslocal ssm get-parameter --name COGNITO_ACTOR_POOL_ID --query Parameter.Value --output text)`
                              ➜ `awslocal cognito-idp admin-delete-user \
                                    --user-pool-id "$POOL_ID" \
                                    --username 'any@example.com'`

---

## Cursor Agent Skills

Skills are on-demand playbooks for Cursor Agent (multi-step workflows). Rules stay always-on constraints; skills load only when invoked or when Agent matches your ask to a skill description.

Installed [Plinth](https://github.com/jabrena/plinth) skills live in this repo:

- `.agents/skills/` — skill folders (committed)
- `skills-lock.json` — source (`jabrena/plinth`) + content hashes (committed)

Spring Boot, Micronaut, Jira, and Azure DevOps skills are intentionally omitted.

### 1. How to use them

In Agent chat:

1. **Slash** — type `/`, search a skill name (e.g. `401-frameworks-quarkus-core`), run it
2. **Attach** — type `@`, pick a skill, then ask your question
3. **Automatic** — describe the work; Agent may pull a matching skill from its description (less predictable)

Examples that fit Backbone:

| Goal             | Skill                                        |
|------------------|----------------------------------------------|
| Quarkus patterns | `/401-frameworks-quarkus-core`               |
| REST resources   | `/402-frameworks-quarkus-rest`               |
| Security         | `/404-frameworks-quarkus-security`           |
| Maven hygiene    | `/110-java-maven-best-practices`             |
| Secure coding    | `/124-java-secure-coding`                    |
| Unit tests       | `/421-frameworks-quarkus-testing-unit-tests` |

### 2. Keeping them in sync

Skills are local copies, not live links to GitHub. They do not auto-update.

```bash
npx skills update -p -y
```

That refreshes **already installed** skills from `jabrena/plinth`, updates `.agents/skills/` and `skills-lock.json`. Review the diff and commit when you want the team on the new versions.

Removed skills stay removed unless you `npx skills add` them again.

Next: [Operations](/docs/operations) to deploy with GitHub Actions, OIDC, and CDK.
