# DataVisor Workspace

## Projects

| Project | Path | Description |
|---------|------|-------------|
| **charts** | `charts/` | K8s Helm values (apiserver, dcluster, dv-liquibase, etc.) |
| **feature-platform (fp)** | `feature-platform/` | Java 8 backend - feature metadata, rule-engine, RAG/Milvus |
| **api-server** | `internal-ui/api-server/` | Java 17 backend - case review, BI, AI chat agent framework |
| **ngsc** | `ngsc/` | Node.js bridge service between frontend and backend |
| **dv-e2e** | `qa-test/dv-e2e/` | Cucumber E2E tests |
| **infra** | `infra/` | Infrastructure configs |

---

# AI Agent Framework Architecture

## Overview

The AI agent framework enables clients to create features, rules, and other entities using LLM-powered agents. It spans two main projects:

| Project | Role |
|---------|------|
| **api-server** (Java 17) | Core agent framework - chat execution, tools, MCP integration |
| **feature-platform** (Java 8) | RAG infrastructure via Milvus + AI entity management |

---

## 1. API-Server: Chat Agent Framework

### 1.1 Core Architecture

```
com.datavisor.chatagent/
├── entity/           # JPA entities (AiAgent, AiChatRun, AiChatMemory, etc.)
├── repository/       # Spring Data JPA repositories
├── service/          # ChatService, ChatMemoryService, ToolSetService
├── controller/       # REST endpoints
├── tools/            # DelegationTool, ToolSetProvider, SseTool
├── mcp/              # MCP protocol conversion utilities
├── config/           # LangChain4j + executor configuration
└── dto/              # Request/response objects
```

### 1.2 Key Entities

| Entity | Purpose |
|--------|---------|
| `AiAgent` | Agent definition with system prompt, tool sets, sub-agents |
| `AiChatRun` | Chat session (PENDING/RUNNING/COMPLETED/FAILED/STOPPED) |
| `AiChatMemory` | Append-only message history (USER/SYSTEM/ASSISTANT/TOOL_RESULT/STOP) |
| `AiChatRunRelation` | Parent-child relationships for sub-agent delegation |
| `AiToolSet` | Tool provider config (INTERNAL/INTERNAL_MCP/EXTERNAL_MCP) |
| `AiToolApprovalRequest` | Tool execution approval workflow |
| `AiEntityAcceptanceRequest` | Entity acceptance workflow |
| `AiModelConfig` | LLM endpoint configuration |

### 1.3 Chat Execution Flow (7 Phases)

```
startChat() → Create AiChatRun + initial messages
    ↓
runChat() → Submit to executor pool
    ↓
runFlow():
  PHASE 1-3: Load history, validate, setup
  PHASE 4:   Stream AI response via LangChain4j
  PHASE 5:   Persist assistant message + create relations
  PHASE 6:   No tools → COMPLETED
  PHASE 7:   Execute tools in parallel
    ↓
callTool() → Async tool execution
    ↓
callback() → Check all tools fulfilled → resume runChat()
```

### 1.4 Sub-Agent Delegation

- **DelegationTool**: `__DV_DELEGATE_TO_AGENT__` tool
- Creates child `AiChatRun` with `isSubChat=true`
- Tracks via `AiChatRunRelation` (parentChatId, childChatId, rootChatId)
- Child executes independently, returns result to parent

### 1.5 Tool System

**Tool Types**:
- `INTERNAL`: Direct API calls via ToolService
- `INTERNAL_MCP`: MCP server with auth headers
- `EXTERNAL_MCP`: Public MCP server

**Tool Naming**: `{toolSetName}__{toolName}` (double underscore delimiter)

**Approval Workflow**:
1. Check `needsApproval()` before execution
2. Create `AiToolApprovalRequest` (PENDING)
3. Send SSE event to frontend
4. Wait for APPROVED/DENIED status

### 1.6 SSE Streaming Events

| Event | Purpose |
|-------|---------|
| `init` | Execution start |
| `token` | LLM token streaming |
| `turn_complete` | End of LLM response |
| `request` | Tool approval needed |
| `agent` | Sub-agent delegation |
| `done` | Chat completed |
| `error` | Error occurred |

### 1.7 Key Files

| File | Location |
|------|----------|
| ChatService | `internal-ui/api-server/core/src/main/java/com/datavisor/chatagent/service/ChatService.java` |
| AiAgent | `internal-ui/api-server/core/src/main/java/com/datavisor/chatagent/entity/AiAgent.java` |
| AiChatRun | `internal-ui/api-server/core/src/main/java/com/datavisor/chatagent/entity/AiChatRun.java` |
| DelegationTool | `internal-ui/api-server/core/src/main/java/com/datavisor/chatagent/tools/DelegationTool.java` |
| ToolSetProvider | `internal-ui/api-server/core/src/main/java/com/datavisor/chatagent/tools/ToolSetProvider.java` |
| MCP SDK | `internal-ui/api-server/mcp-sdk/` |

---

## 2. Feature-Platform: RAG & AI Entity Management

### 2.1 Milvus/RAG Infrastructure

**Purpose**: Vector database for semantic + full-text hybrid search on features/rules

**Key Components**:
| Component | Purpose |
|-----------|---------|
| `MilvusManager` | Wrapper around Milvus SDK with retry/backoff |
| `RagSearchService` | Hybrid search using DSL |
| `RagIngestionService` | Document CRUD operations |
| `RagEntityRegistry` | Entity type definitions (FEATURE, RULE) |
| `RagFieldRegistry` | Field configurations with embedding models |

**Search DSL**:
- Semantic search (vector similarity)
- Full-text search (BM25)
- Scalar filtering
- Result re-ranking

### 2.2 AI Entity Status

New entity status: `AI_DRAFT` (in addition to DRAFT, PUBLISHED, HIDDEN, ARCHIVED)

**Workflow**:
1. Agent creates entity with `AI_DRAFT` status + `rootChatId`
2. Entity indexed in Milvus for searchability
3. User accepts → converted to `DRAFT`
4. User rejects → deleted

### 2.3 AI API Endpoints

| Endpoint | Purpose |
|----------|---------|
| `POST /{tenant}/feature/ai/create` | Create AI-draft feature |
| `POST /{tenant}/feature/ai/search` | Hybrid search on features |
| `POST /{tenant}/feature/ai/update` | Update AI-draft feature |
| `POST /{tenant}/ai_entity/accept` | Accept AI_DRAFT → DRAFT |
| `POST /{tenant}/ai_entity/reject` | Reject/delete AI_DRAFT |

### 2.4 Key Files

| File | Location |
|------|----------|
| RagSearchService | `feature-platform/feature/src/main/java/com/datavisor/rag/service/RagSearchService.java` |
| MilvusManager | `feature-platform/feature/src/main/java/com/datavisor/rag/infrastructure/milvus/MilvusManager.java` |
| RagEntityRegistry | `feature-platform/feature/src/main/java/com/datavisor/storage/model/rag/RagEntityRegistry.java` |
| FeatureController (AI) | `feature-platform/api/src/main/java/com/datavisor/fp/controller/FeatureController.java` |
| MultipleVersionEntityController | `feature-platform/api/src/main/java/com/datavisor/fp/controller/MultipleVersionEntityController.java` |

### 2.5 RAG Indexing

#### Manual Indexing APIs

| Endpoint | Purpose |
|----------|---------|
| `POST /admin/rag/internal/{FEATURE\|RULE}/register` | Register entity schema (one-time) |
| `POST /{tenant}/rag/internal/{FEATURE\|RULE}/reindex-all` | Reindex all entities for tenant |
| `DELETE /admin/rag/internal/delete-all` | Delete all RAG registrations + changelog records |

#### Automatic Indexing via DVChangelog

On FP startup or tenant creation, `DVChangelogService.maybeExecuteActions()` runs registered changelogs:

```
activeChangelogNames = [
    ...
    REGISTER_AND_INDEX_FEATURE_IN_RAG,  // register_and_index_feature_in_rag
    REGISTER_AND_INDEX_RULE_IN_RAG,     // register_and_index_rule_in_rag
    ...
]
```

**Trigger Points**:
1. **FP Startup** (`EntityRepositories.java:6310`) - for each tenant, only on consumer FP instances
2. **Tenant Creation** (`TenantService.java:308`) - immediate execution for new tenants

**Execution Logic** (`DVChangelogService.java:291-314`):
```java
// 1. Check if already executed (idempotent)
Boolean newlyCreated = daoManager.checkOrCreateDVChangelogByName(recordName);
if (!newlyCreated) return;  // Skip if already run

// 2. Execute action
executeOneAction(actionName);

// 3. Mark completed (or delete record on failure to retry later)
daoManager.updatedStatusForDVChangelogByName(recordName, COMPLETED);
```

**RAG Indexing Actions** (`DVChangelogService.java:1229-1249`):
```java
registerAndIndexFeatureInRag() {
    ragBuiltInService.registerFeatureEntity();    // Create schema + Milvus collection
    ragBuiltInService.deleteAllFeatureIndexes();  // Clear existing
    ragBuiltInService.indexAllFeatures();         // Index all features
}

registerAndIndexRuleInRag() {
    ragBuiltInService.registerRuleEntity();
    ragBuiltInService.deleteAllRuleIndexes();
    ragBuiltInService.indexAllRules();
}
```

#### Indexing Key Files

| File | Location |
|------|----------|
| RagBuiltInService | `feature-platform/feature/src/main/java/com/datavisor/rag/service/RagBuiltInService.java` |
| RagBuiltInController | `feature-platform/api/src/main/java/com/datavisor/rag/controller/RagBuiltInController.java` |
| RagIngestionService | `feature-platform/feature/src/main/java/com/datavisor/rag/service/RagIngestionService.java` |
| DVChangelogService | `feature-platform/feature/src/main/java/com/datavisor/service/DVChangelogService.java` |

---

## 3. Integration Flow

```
User → Chat UI
    ↓
api-server (ChatController)
    ↓
ChatService.runFlow()
    ↓
LangChain4j + Claude LLM
    ↓
Tool Execution (e.g., feature-tools__create_feature)
    ↓
feature-platform API (FeatureController.createAIFeature)
    ↓
Feature stored as AI_DRAFT with rootChatId
    ↓
Indexed in Milvus (RagIngestionService)
    ↓
AiEntityAcceptanceRequest created
    ↓
User accepts/rejects via UI
```

---

## 4. Technology Stack

| Layer | Technology |
|-------|------------|
| LLM | Claude Sonnet 4 via LiteLLM gateway |
| AI Framework | LangChain4j 1.8.0+ |
| Tool Protocol | MCP (Model Context Protocol) |
| Vector DB | Milvus |
| SQL DB | MySQL |
| NoSQL | Cassandra/YugabyteDB (streaming) |
| API | REST + SSE |

---

## 5. Key Architectural Patterns

- **Append-only history**: Immutable chat memory with seq-based ordering
- **Optimistic locking**: Version fields on AiChatRun, sequence on memory head
- **Hierarchical agents**: Parent-child delegation via AiChatRunRelation
- **Tool abstraction**: Unified interface for internal/MCP tools
- **Event streaming**: Real-time SSE for chat updates
- **Approval workflows**: Tool execution + entity acceptance gates
