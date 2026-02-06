# DataVisor Workspace

> For workspace overview, worktree usage, and typical workflows, see [README.md](README.md).

## Projects

| Project | Path | Description |
|---------|------|-------------|
| **charts** | `charts/` | K8s Helm values (apiserver, dcluster, dv-liquibase, etc.) |
| **feature-platform (fp)** | `feature-platform/` | Java backend - feature metadata, rule-engine, RAG/Milvus (Java 17 since `DV.202602A.External`; Java 8 on `DV.202509C.External` and earlier) |
| **api-server** | `internal-ui/api-server/` | Java 17 backend - case review, BI, AI chat agent framework |
| **ngsc** | `ngsc/` | Node.js bridge service between frontend and backend |
| **dv-e2e** | `qa-test/dv-e2e/` | Cucumber E2E tests |
| **infra** | `infra/` | Infrastructure configs |

## General Rules

- When asked to **review or analyze code**, do NOT make changes unless explicitly asked. Separate review/analysis from implementation.
- When splitting work across **multiple repositories**, confirm the target repo for each file/change BEFORE starting implementation. Do not put all changes in one repo by default.

## SQL & Database

When writing SQL queries against existing databases, ALWAYS query the database first to discover actual field values, enum values, project keys, and priorities before writing final queries. Never assume or hardcode values.

## Build & Testing

- **feature-platform**: Java 17 on branch `DV.202602A.External` and later; Java 8 on `DV.202509C.External` and earlier. Check the current branch before building.
- **api-server**: Java 17.
- Do not assume Java version, JVM arguments, or API availability without checking. Run `java -version` or inspect `pom.xml` before building.
- **SDKMAN** is installed. Use `sdk use java <version>` to switch:
  - `sdk use java 17.0.17-amzn` (for feature-platform on `DV.202602A.External`+, api-server)
  - `sdk use java 8.0.462-amzn` (for feature-platform on `DV.202509C.External` and earlier)

## Debugging & Bug Fixes

Prefer simple fixes first. When debugging or fixing issues, propose the simplest possible solution before suggesting complex approaches with tracking sets, new data structures, or architectural changes.

## AI Agent Prompts / Prompt Engineering

Keep prompts and configuration modular. Never hardcode values (search keys, status values, entity names) into prompt files — query them dynamically or reference shared config. Ask before deciding where to place new content.

## Worktree Sessions

This workspace uses git worktrees to enable parallel development. When working in a directory prefixed with `WT__`, you are in an isolated session for a specific task.

### Detecting Your Context

| Working Directory | Context | Branch Pattern |
|-------------------|---------|----------------|
| `dv_workspace/` | Root workspace | Main repos on release branches |
| `dv_workspace/feature-platform/` | Main submodule | Release branch (e.g., `DV.202602A.External`) |
| `dv_workspace/WT__PI-100/` | Worktree session | Task branches (e.g., `DV.202602A.External.PI-100`) |

### AI Agent Guidelines for Worktree Sessions

When working in a `WT__<session>/` directory:

1. **Scope**: Only modify files within the session's repos. The session name (e.g., `PI-100`) usually corresponds to a ticket.

2. **Commits**: Commit to the session branch (e.g., `DV.202602A.External.PI-100`), not the base branch.

3. **Testing**: Run tests within the worktree directory. Each repo in the session is independent.

4. **Context**: If the user says "work on PI-100", navigate to `WT__PI-100/` if it exists.

### Useful Commands

```bash
# From dv_workspace root
./worktree list              # List all sessions
./worktree list PI-100       # Show repos in PI-100 session

# Check current branch in a worktree
cd WT__PI-100/feature-platform
git branch --show-current    # Shows: DV.202602A.External.PI-100
```

## Project Documentation

For product-specific architecture and technical details, see the CLAUDE.md in each submodule:

| Project | Documentation |
|---------|---------------|
| feature-platform | `feature-platform/CLAUDE.md` |
| api-server | `internal-ui/api-server/CLAUDE.md` |

### Feature-Platform Docs Index

| Document | Path | Description |
|----------|------|-------------|
| Hotspot Aggregation | `feature-platform/docs/hotspot-aggregation.md` | Hotspot triggering, backfill, and Flink batch synchronization |
| Maven Test Runner | `feature-platform/MAVEN_TEST_RUNNER_README.md` | Automated retry logic for flaky Maven tests |
| Local Manual Setup | `feature-platform/tools/src/main/localsetup/local-manual-setup/README.md` | Local dev environment setup (MySQL, Kafka, YugabyteDB, etc.) |
| FP Deps Docker Compose | `feature-platform/tools/src/main/localsetup/external-yuga-setup/README.md` | Docker Compose for FP dependencies with profiles |
| Docker Compose Deploy | `feature-platform/tools/src/main/docker-compose/README.md` | Legacy Docker Compose deployment |
| Standalone ITest | `feature-platform/itest/STANDALONE.md` | Building integration tests as standalone JAR |
| Cassandra Helm Chart | `feature-platform/tools/src/main/charts/dv-cassandra/README.md` | Kubernetes Cassandra StatefulSet deployment |
