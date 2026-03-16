# Architecture

This document describes the MCP tool server architecture — how to wrap an existing backend service (e.g., a code search engine, a document store) as a standardized MCP tool server that agents can invoke. For the engineering rationale behind these choices, see [Design Principles](design-principles.md).

## The Problem

You have a backend service with a REST API — a code search engine, a log query service, a document store. You want AI agents to use it. But agents can't call arbitrary REST APIs directly. They need:

1. **Typed tool contracts** — structured input schemas, output schemas, and descriptions that the agent framework can discover and validate
2. **Pagination** — search results can be large; the server must manage pagination state so agents can iterate through results without blowing up context windows
3. **Error handling** — backend HTTP errors must be translated into structured MCP error codes that agents can reason about
4. **Dual-mode transport** — developers test locally (stdio, no auth); production runs in the cloud (HTTP+SSE, with auth). The tool logic must be identical in both.

The MCP tool server solves this by acting as a **thin protocol adapter** between agents and your backend.

## System Overview

The MCP server layer bridges AI agents and backend services through the [Model Context Protocol (MCP)](https://modelcontextprotocol.io/). A single MCP server acts as a thin protocol adapter that translates agent tool calls into backend REST API requests. It handles protocol-level concerns only (input validation, response formatting, error mapping, pagination) while all business logic resides in the backend.

## Technology Mapping

Throughout this document, architectural roles are described generically. The table below lists common technology options for the **server-side** components.

| Architectural Role | Examples |
|-------------------|----------|
| MCP framework | FastMCP (Python), mcp-python-sdk, mcp-typescript-sdk, custom implementation |
| Backend: search engine | Zoekt (trigram-indexed), Elasticsearch, Sourcegraph, OpenSearch |
| Backend: document store | S3 + metadata DB, Azure Blob + Cosmos DB, PostgreSQL, DynamoDB |
| Pagination state | DynamoDB (TTL), Redis, Cosmos DB Table API, in-memory (local mode) |
| IDE integration (client-side) | VS Code Extension API, JetBrains plugin SDK, Neovim plugin |

## MCP Server Internal Layers

The MCP server is structured as three internal layers, each with a single responsibility:

### Layer 1: Tool Layer

The outermost layer facing the agent. It defines typed tool contracts (search, list, get, help), validates input parameters against schemas, and manages pagination cursors. Each tool category has its own schema (see [`schemas/`](../schemas/) for templates).

**Key responsibilities**:
- Parse and validate tool call inputs against the schema
- Reject malformed inputs before they reach the backend
- Manage pagination: create cursors, validate cursor tokens, handle expired cursors
- Enforce max_results limits and content slicing parameters

### Layer 2: Request Translation

The middle layer that converts MCP tool calls into HTTP requests targeting the backend REST API, and converts HTTP responses back into MCP tool results. This layer is responsible for serialization, header construction, and endpoint routing.

**Key responsibilities**:
- Map tool name + parameters → HTTP method + URL + body
- Attach authentication headers (cloud mode only)
- Map HTTP response body → MCP tool result format
- Handle response pagination metadata (next_cursor, total_count)

### Layer 3: Error Handling

The innermost layer before the network boundary. It maps backend HTTP errors and transport failures into structured MCP error codes (JSON-RPC 2.0), attaches request IDs for log correlation, and ensures every failure path produces an observable, structured response.

**Key responsibilities**:
- Map HTTP 4xx/5xx → specific MCP error codes (NOT_FOUND, BACKEND_UNAVAILABLE, etc.)
- Attach request IDs for cross-system log correlation
- Handle transport failures (timeout, connection refused) as structured errors
- Never leak raw backend error details to the agent

## Dual-Mode Transport

The MCP server supports two transport modes with **identical tool logic**:

| Mode | Transport | Agent → Server Auth | Server → Backend Auth | Use Case |
|------|-----------|-------------------|---------------------|----------|
| **Local (stdio)** | stdin/stdout JSON-RPC | None (local trust) | None (localhost) | Development, IDE integration, testing |
| **Cloud (HTTP)** | HTTP + Server-Sent Events | OAuth2 at gateway | Cloud-native auth | Production, multi-tenant agents |

Mode selection is determined at startup via configuration; the tool logic is identical in both modes.

### Multi-Cloud Auth Mapping

In cloud mode, two authentication layers are needed. The specific mechanism depends on the cloud provider:

| Auth Layer | Purpose | AWS | Azure |
|-----------|---------|-----|-------|
| Agent → Gateway (Layer 1) | Authenticate external agents | API Gateway + Cognito (OAuth2 JWT) | API Management + Entra ID (OAuth2) |
| Server → Backend (Layer 2) | Authenticate MCP server to backend API | IAM Auth (SigV4 request signing) | Managed Identity |
| Credential management | Rotate/revoke outbound credentials | Secrets Manager | Key Vault |

> For the full cloud deployment architecture (containerization, serverless compute, infrastructure templates), see the companion project [MCP-tool-deployment-pattern](../../MCP-tool-deployment-pattern/).

## Tool Categories

The single MCP server exposes multiple tool categories, each addressing a different data access pattern:

### Search-Domain Tools

Tools for querying indexed data (code, logs, metrics). These follow the **discovery → search → retrieval** pattern described in [`schemas/search-tool-template.yaml`](../schemas/search-tool-template.yaml).

**Pagination**: Cursor-based, backed by a key-value store with TTL. Cursors are opaque server-side tokens that expire automatically.

**Example backend**: Zoekt code search engine — the MCP server wraps Zoekt's REST API, adding typed tool contracts, input validation, cursor-based pagination, and structured error codes.

### Retrieval-Domain Tools

Tools for accessing document stores (runbooks, wikis, policies). These follow the **list → search metadata → retrieve** pattern described in [`schemas/retrieval-tool-template.yaml`](../schemas/retrieval-tool-template.yaml).

**Pagination**: Offset-based (stateless), appropriate when the document corpus is small and changes infrequently.

### Choosing a Tool Category

| Factor | Search-Domain Tools | Retrieval-Domain Tools |
|--------|-------------------|----------------------|
| Data volume | Large (millions of files/records) | Small (hundreds to thousands of documents) |
| Query complexity | Full-text search, regex, filtering | Keyword matching, metadata filtering |
| Compute needs | Indexing engine, ranked results | Simple reads, no indexing |
| Pagination | Cursor-based (server-side state, TTL) | Offset-based (stateless) |
| Example backends | Zoekt, Elasticsearch, OpenSearch | S3 + metadata, document DBs, wikis |

## Data Flow

All tool calls — regardless of category — follow a single data path:

```
[Agent issues MCP tool call (e.g., search, list_documents, get_file)]
         |
         v
[Tool Layer: validate input against schema, check pagination cursor]
         |
         v
[Request Translation: convert tool call → HTTP request + auth headers]
         |
         v
[Error Handling: attach request ID, wrap outbound request]
         |
         v
[Backend REST API: authenticate, route, execute (search/retrieval)]
         |
         v
[Error Handling: map HTTP status → MCP error code, attach request ID]
         |
         v
[Request Translation: convert HTTP response → MCP tool result]
         |
         v
[Tool Layer: attach pagination cursor if applicable]
         |
         v
[Agent receives structured result with typed output]
```

**Pagination flow**:
- **Cursor-based** (search): First call → backend creates cursor in key-value store (TTL=15min) → returns results + `next_cursor`. Next call with `cursor` → backend looks up state → returns next page. Cursor expires after TTL.
- **Offset-based** (retrieval): Each call includes `offset` and `limit`. Stateless — no server-side state needed.

## Deployment Modes

| Mode | Components Running | Storage | Auth | Typical Use |
|------|-------------------|---------|------|-------------|
| **Local** | MCP server (stdio) + backend (HTTP on localhost) | Local filesystem | None | Developer testing, IDE integration |
| **Cloud** | MCP server (HTTP/SSE) + API gateway + serverless backend | Object storage + shared filesystem + key-value store | Two-layer: OAuth2 (gateway) + cloud-native (backend) | Multi-tenant production |
| **Hybrid** | MCP server (stdio, local) → API gateway (cloud) | Cloud storage | Cloud-native auth (Layer 2 only) | IDE extension accessing cloud indexes |

> For the full cloud deployment reference (containerization, networking, operational checklists, multi-cloud mapping), see [MCP-tool-deployment-pattern](../../MCP-tool-deployment-pattern/).
