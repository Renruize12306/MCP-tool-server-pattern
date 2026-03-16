# Design Principles

This document captures the engineering rationale and trade-off analysis behind the MCP tool server design patterns. For the concrete component breakdown and data flow details, see [Architecture](architecture.md).

## Core Principles

### 1. Thin MCP Server: Protocol Adapter Only

**Decision**: The MCP server is a thin translation layer. It handles input validation, protocol formatting, pagination, and error mapping. All business logic lives in the backend service.

**Rationale**:
- **Testability**: Backend services can be tested independently of the MCP protocol
- **Reusability**: The same backend can serve MCP clients, REST clients, CLI tools, and IDE extensions
- **Operational simplicity**: MCP servers are stateless and lightweight; complex state management stays in the backend
- **Upgradeability**: MCP protocol evolves independently from backend logic — upgrading one doesn't break the other

**Anti-pattern avoided**: "Fat MCP servers" that embed search logic, caching, and data transformation. These become hard to test, hard to scale, and tightly coupled to the MCP framework.

### 2. Dual-Mode Transport

**Decision**: Every MCP server runs in two modes — local (stdio) for development and cloud (HTTP+SSE) for production — with identical tool logic.

**Rationale**:
- **Developer experience**: Engineers can test tool interactions locally without deploying infrastructure or configuring auth
- **Parity**: The same tool contract is exercised in both modes, reducing "works locally, breaks in production" surprises
- **IDE integration**: Local stdio mode enables direct integration with editors (VS Code, JetBrains, Neovim)

**Implementation**: Mode selection is a startup configuration flag. The tool handler functions are shared; only the transport adapter differs.

**Auth difference by mode**:

| Mode | Agent → Server | Server → Backend |
|------|---------------|-----------------|
| Local (stdio) | None (local trust) | None (localhost) |
| Cloud (HTTP) | OAuth2 at gateway | Cloud-native auth (AWS: SigV4 / Azure: Managed Identity) |

### 3. Cursor-Based Pagination for Search

**Decision**: Search-domain tools use cursor-based pagination with server-side state (key-value store + TTL), not offset-based pagination.

**Rationale**:
- **Stability**: Offset-based pagination breaks when the underlying index changes between pages (results shift, duplicates appear). Cursors capture a snapshot of the query state.
- **Agent-friendly**: Agents don't need to track page numbers or compute offsets. They just pass `next_cursor` from the previous response. This is simpler for LLM-generated tool calls.
- **Automatic cleanup**: Cursors expire via TTL (e.g., 15 minutes), so no garbage collection or client-side cleanup is needed.
- **Context window safety**: `max_results` per page is bounded (default 50, max 200), preventing a single tool call from returning megabytes of data.

**Trade-off**: Server-side cursor state requires a key-value store. Mitigated by:
- Using managed services with TTL (DynamoDB, Cosmos DB Table API, Redis)
- Cursors are small (query hash + offset pointer) — minimal storage cost
- In local mode, cursors can be stored in-memory

**When to use offset-based instead**: For retrieval-domain tools where the document corpus is small (hundreds to low thousands), changes infrequently, and query volume is low. See `schemas/retrieval-tool-template.yaml`.

### 4. Structured Error Handling with Request IDs

**Decision**: Every error path produces a structured MCP error response with a JSON-RPC 2.0 error code and a unique request ID. No raw HTTP errors are forwarded to agents.

**Rationale**:
- **Agent reasoning**: Agents can branch on structured error codes (e.g., retry on `BACKEND_UNAVAILABLE`, re-query on `CURSOR_EXPIRED`, stop on `NOT_FOUND`). Raw HTTP status codes are ambiguous.
- **Observability**: Request IDs enable cross-system log correlation (agent → MCP server → backend). When an investigation involves "why did the agent get this result?", the request ID traces the full path.
- **Security**: Raw backend errors may leak internal details (stack traces, internal URLs). Error mapping ensures only safe, structured information reaches the agent.

**Error code mapping**:

| Backend Condition | MCP Error Code | JSON-RPC Code | Agent Action |
|------------------|---------------|---------------|-------------|
| HTTP 400 / validation failure | `INVALID_PARAMS` | -32602 | Fix input and retry |
| HTTP 404 / resource not found | `NOT_FOUND` | -32001 | Try different resource |
| HTTP 503 / backend down | `BACKEND_UNAVAILABLE` | -32002 | Wait and retry |
| Cursor expired | `CURSOR_EXPIRED` | -32003 | Re-issue query without cursor |
| Unexpected failure | `INTERNAL_ERROR` | -32603 | Report to operator |

### 5. Content Slicing for Large Results

**Decision**: Retrieval tools support content slicing (line ranges, byte ranges, page ranges) to prevent returning entire large files in a single response.

**Rationale**:
- **Context window limits**: LLM context windows are finite. A single `get_file` call that returns a 10,000-line file will blow past the limit.
- **Agent efficiency**: Agents typically need a specific section of a file (e.g., lines 100-150 around a search match), not the whole thing.
- **Network efficiency**: Smaller payloads reduce latency and cost.

**Implementation**: `slice` parameter in retrieval tools with `start` and `end` fields. The `truncated` boolean in the response tells the agent whether more content exists.

## Key Engineering Trade-Offs

| Decision Area | Reference Choice | Alternatives Considered | Rationale |
|--------------|-----------------|------------------------|-----------|
| Server thickness | Thin (protocol adapter only) | Fat (embed search logic) | Thin servers are testable, reusable, and independently deployable |
| Search pagination | Cursor-based (server-side state) | Offset-based (stateless) | Cursors are stable under index changes and agent-friendly |
| Retrieval pagination | Offset-based (stateless) | Cursor-based | Small corpus, infrequent changes — simplicity wins |
| Error mapping | Structured MCP codes + request ID | Forward raw HTTP errors | Agents need structured codes; raw errors leak internals |
| Transport | Dual-mode (stdio + HTTP) | HTTP only | Local stdio enables IDE integration and zero-config testing |
| MCP framework | Existing open-source library | Custom implementation | Library handles protocol boilerplate; focus on tool logic |
| Backend coupling | REST API boundary | Direct function calls | REST boundary enables independent deployment and testing |
| Content delivery | Sliced retrieval with truncation flag | Full content always | Protects agent context windows; reduces payload size |

## Extensibility: Adding New Tool Categories

The pattern is designed so that new data sources can be integrated by adding tool categories to the MCP server without modifying existing tools or the backend integration path:

1. **Define the tool contract** — Create a new YAML schema (see `schemas/` for templates). Use `search-tool-template.yaml` for indexed/queryable data or `retrieval-tool-template.yaml` for document stores.

2. **Implement tool handlers** — Add handlers to the Tool Layer. Each follows the same three-layer flow: Tool Layer (validation) → Request Translation (MCP-to-HTTP) → Error Handling (structured codes).

3. **Extend the backend REST API** — Add API endpoints for the new tool category. All tools go through HTTP + auth to the backend, ensuring consistent authentication and rate limiting.

4. **Wire authentication** — In cloud mode, new tools automatically inherit the existing gateway auth (Layer 1: OAuth2) and backend auth (Layer 2: cloud-native). In local mode, no auth changes needed.

5. **Add operational checklists** — Extend deployment and security checklists to cover the new tool category's concerns (see [MCP-tool-deployment-pattern](../../MCP-tool-deployment-pattern/)).

## Failure Handling Philosophy

1. **Fail fast at the protocol boundary**: Validate inputs and return structured errors before making backend calls
2. **Retry at the infrastructure layer**: API gateways and serverless platforms provide built-in retry with backoff; the MCP server does not implement its own retry loops
3. **Degrade gracefully**: If the search index is stale (replication lag), return results with a staleness indicator rather than failing
4. **Observable failures**: Every error path emits structured logs and increments a metric counter; no silent failures
5. **Cursor failures are recoverable**: Expired cursor → `CURSOR_EXPIRED` error → agent re-issues query. Not a fatal failure.
