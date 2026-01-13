# DataVisor Workspace

> For workspace overview, worktree usage, and typical workflows, see [README.md](README.md).

## Projects

| Project | Path | Description |
|---------|------|-------------|
| **charts** | `charts/` | K8s Helm values (apiserver, dcluster, dv-liquibase, etc.) |
| **feature-platform (fp)** | `feature-platform/` | Java 8 backend - feature metadata, rule-engine, RAG/Milvus |
| **api-server** | `internal-ui/api-server/` | Java 17 backend - case review, BI, AI chat agent framework |
| **ngsc** | `ngsc/` | Node.js bridge service between frontend and backend |
| **dv-e2e** | `qa-test/dv-e2e/` | Cucumber E2E tests |
| **infra** | `infra/` | Infrastructure configs |

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
