# Design Principles

This document captures the engineering rationale and trade-off analysis behind the MCP Tool Access Pattern. For the concrete component breakdown and deployment details, see [Architecture](architecture.md).

## Core Principles

### 1. Protocol-First: MCP as the Standard Interface

**Decision**: Use MCP as the sole interface between AI agents and backend services. No custom agent-tool bindings.

**Rationale**: Agent frameworks evolve rapidly; backend services are long-lived. By placing MCP as the contract boundary, we decouple agent evolution from service evolution. A new agent framework (or a new version of an existing one) can access existing tools without backend changes, as long as it speaks MCP.

**Trade-off**: MCP adds a serialization layer compared to direct function calls. In practice, the overhead is negligible relative to network latency and LLM inference time.

### 2. Separation of Concerns: Thin MCP Servers

**Decision**: MCP servers are thin translation layers only. They handle input validation, protocol formatting, and error mapping. All business logic lives in backend services.

**Rationale**:
- **Testability**: Backend services can be tested independently of the MCP protocol
- **Reusability**: The same backend service can serve MCP clients, REST clients, and CLI tools
- **Operational simplicity**: MCP servers are stateless and lightweight; complex state management stays in the backend

**Anti-pattern avoided**: "Fat MCP servers" that embed search logic, caching, and data transformation. These become hard to test, hard to scale, and tightly coupled to the agent framework.

### 3. Cloud-Native by Default

**Decision**: Design for serverless compute, managed storage, and infrastructure as code from the start.

**Rationale**:
- **Scale-to-zero**: Agent tool access is bursty — high during incidents, near-zero otherwise. Serverless avoids paying for idle capacity.
- **Operational overhead**: Managed services eliminate patching, capacity planning, and failover configuration for storage and compute.
- **Reproducibility**: Code-defined infrastructure stacks make the entire deployment version-controlled and reproducible across environments.

**Trade-off**: Cold start latency on serverless functions. Mitigated by:
- Compiled-language runtimes (fast cold starts, ~200ms)
- Shared filesystem for indexes (no download on cold start)
- Provisioned concurrency for latency-sensitive deployments

### 4. Dual-Mode Deployment

**Decision**: Every MCP server runs in two modes — local (stdio) for development and cloud (HTTP) for production — with identical tool logic.

**Rationale**:
- **Developer experience**: Engineers can test tool interactions locally without deploying infrastructure
- **Parity**: The same tool contract is exercised in both modes, reducing integration surprises
- **IDE integration**: Local stdio mode enables direct integration with editors and development tools

**Implementation**: Mode selection is a startup configuration flag. The tool handler functions are shared; only the transport adapter differs.

### 5. Authentication at the Edge

**Decision**: All authentication and authorization happens at the API gateway layer (cloud-native auth for internal clients, OAuth2 for external agent platforms). MCP servers and backend services trust the authenticated identity passed through.

**Rationale**:
- **Single enforcement point**: Authentication logic is not duplicated across services
- **Least privilege**: Gateway policies enforce per-client rate limits and resource access
- **Local bypass**: In development mode (stdio), no auth is needed — the local user is implicitly trusted

**Trade-off**: Requires a well-configured API gateway; misconfiguration at the edge exposes the entire backend. Mitigated by security audit checklists (see [`templates/security-audit-checklist.md`](../templates/security-audit-checklist.md)).

## Key Engineering Trade-Offs

The table below captures trade-off decisions that arise when implementing this pattern. The "reference choice" column reflects one validated set of decisions; alternatives may be preferable depending on organizational constraints.

| Decision Area | Reference Choice | Alternatives Considered | Rationale |
|--------------|-----------------|------------------------|-----------|
| Search engine | Purpose-built code search library | General-purpose search service, hosted search API | A library embedded in the function binary has a smaller operational footprint and avoids a network hop |
| Index storage | Shared filesystem (hot cache) | Direct object storage reads, block volumes | Shared filesystem provides concurrent access across function invocations without cold-start downloads |
| Pagination state | Key-value store with TTL | Cache service, in-memory | TTL provides automatic cursor cleanup with no additional infrastructure |
| Infrastructure framework | Type-safe, code-defined (same language as IDE extension) | Declarative config (HCL, YAML) | Type-safe constructs catch errors at compile time; language sharing reduces context-switching |
| MCP framework | Existing open-source MCP library | Custom implementation | A library handles protocol boilerplate; focus development effort on tool logic |
| Backend language | Compiled language with fast cold starts | Interpreted language, systems language | Fast cold starts are critical for serverless; single-binary deployment simplifies packaging |

## Extensibility: Adding New MCP Servers

The pattern is designed so that new data sources can be integrated by adding MCP servers without modifying existing components. The process follows a consistent template:

1. **Define the tool contract** — Create a new YAML schema (see `schemas/` for templates) specifying each tool's name, description, input schema, and output schema. Use `search-tool-template.yaml` for indexed/queryable data or `retrieval-tool-template.yaml` for document stores.

2. **Implement a thin MCP server** — Build a server using an MCP framework that translates tool calls into backend requests. Follow the Separation of Concerns principle: the MCP server handles protocol concerns only (validation, formatting, error mapping). All business logic belongs in the backend.

3. **Choose the backend integration pattern**:
   - **Direct storage access**: Suitable when the data source is a managed service with an SDK and operations are simple reads. The MCP server calls the storage API directly.
   - **Dedicated backend service**: Suitable when the operation requires custom compute logic, indexing, or stateful pagination. Deploy the service using the existing infrastructure stack pattern.

4. **Wire authentication** — If the new server uses cloud mode (HTTP), route it through the existing API gateway for consistent authentication. If it only needs local mode (stdio), no auth changes are required.

5. **Add operational checklists** — Extend the deployment and security checklists to cover the new server's specific concerns (storage permissions, rate limits, data sensitivity).

**When to use which pattern**: Choose direct storage access when the operation is a simple read/write against a managed service. Choose a dedicated backend service when the operation involves indexing, search, aggregation, or custom business logic. See the "When to Use Which Path" table in [Architecture](architecture.md) for a detailed comparison.

## Failure Handling Philosophy

1. **Fail fast at the protocol boundary**: MCP servers validate inputs and return structured errors before making backend calls
2. **Retry at the infrastructure layer**: API gateways and serverless platforms provide built-in retry with backoff; application code does not implement its own retry loops
3. **Degrade gracefully**: If the search index is stale (replication lag), return results with a staleness indicator rather than failing
4. **Observable failures**: Every error path emits structured logs and increments a metric counter; no silent failures
