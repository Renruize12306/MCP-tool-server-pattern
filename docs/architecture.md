# Architecture

This document describes the component architecture, data flow, and deployment modes of the MCP Tool Access Pattern. For the engineering rationale behind these choices, see [Design Principles](design-principles.md).

## System Overview

The system bridges AI agent frameworks and organizational backend services through the Model Context Protocol (MCP). It consists of four layers:

1. **MCP Server Layer** — Thin protocol adapters exposing backend services as agent-accessible tools
2. **Backend Service Layer** — High-performance services implementing search, retrieval, and data access
3. **Cloud Infrastructure Layer** — Serverless compute, managed storage, and networking (code-defined)
4. **Developer Integration Layer** — IDE extensions and local development tooling

## Technology Mapping

Throughout this document, architectural roles are described generically. The table below lists common technology options for each role. Any combination that satisfies the role requirements can be used.

| Architectural Role | Examples |
|-------------------|----------|
| MCP framework | FastMCP (Python), mcp-python-sdk, custom implementation |
| Search engine | Zoekt, Elasticsearch, Sourcegraph, Ripgrep |
| Backend language | Go, Rust, Java, C++ (compiled binary recommended for cold starts) |
| Serverless compute | AWS Lambda, Google Cloud Functions, Cloud Run, Azure Functions |
| API gateway | AWS API Gateway, Google Cloud Endpoints, Azure API Management, Kong, Envoy |
| Object storage | AWS S3, Google Cloud Storage, Azure Blob Storage, MinIO |
| Shared filesystem | AWS EFS, Google Filestore, Azure Files, NFS |
| Key-value store (cursors) | DynamoDB, Redis, Firestore, Memcached (TTL support required) |
| Data replication | AWS DataSync, rsync, Google Storage Transfer Service |
| Infrastructure framework | CDK, Terraform, Pulumi, CloudFormation |
| Cloud-native auth | Cloud-native request signing (e.g., SigV4), JWT, mTLS, API keys |
| IDE integration | VS Code Extension API, JetBrains plugin SDK, Neovim plugin |
| Agent framework | Strands Agents SDK, LangChain, AutoGen, custom |

## Component Breakdown

### 1. MCP Servers

Thin protocol wrappers built with an MCP framework that translate MCP tool calls into backend service requests. Each server is stateless and handles protocol-level concerns only (input validation, response formatting, error mapping).

MCP servers in this pattern fall into two categories, each with a distinct backend integration style:

**Search-domain servers** expose tools for querying indexed data (code, logs, metrics). These forward requests to a dedicated backend compute service through an API gateway. Tools follow the discovery → search → retrieval pattern described in [`schemas/search-tool-template.yaml`](../schemas/search-tool-template.yaml).

**Retrieval-domain servers** expose tools for accessing document stores (runbooks, wikis, policies). These access object storage directly via SDK calls, without an intermediate compute layer. Tools follow the list → search metadata → retrieve pattern described in [`schemas/retrieval-tool-template.yaml`](../schemas/retrieval-tool-template.yaml).

**Dual-mode operation**: Each server supports two transport modes:

| Mode | Transport | Auth | Use Case |
|------|-----------|------|----------|
| **Local (stdio)** | stdin/stdout JSON-RPC | None (local trust) | Development, IDE integration, testing |
| **Cloud (HTTP)** | HTTP + Server-Sent Events | Cloud-native auth at API gateway | Production multi-tenant deployment |

Mode selection is determined at startup via configuration; the tool logic is identical in both modes.

### 2. Backend Search Service

A high-performance REST API wrapping a code search engine. This service handles compute-intensive search operations and provides cursor-based pagination for large result sets.

The service compiles to a single binary with **dual entry points**: a standalone HTTP server for local development, and a serverless function handler for production. In cloud mode, the serverless function executes the binary directly — the "search service" is not a separate process but the application logic running inside the function.

> See [`schemas/backend-api-template.yaml`](../schemas/backend-api-template.yaml) for the endpoint template.

**Key design decisions**:

- **Cursor-based pagination** (key-value store with TTL) rather than offset-based, to support stable iteration over changing index data
- **Dual entry points**: Standalone HTTP server for local development; serverless function handler for production (same binary, different entrypoint)
- **Shared filesystem for indexes**: Search indexes are large binary files; a shared filesystem provides low-latency access across function invocations without cold-start re-download
- **Structured logging + metrics**: Operational observability built in from the start

### 3. Cloud Infrastructure

Multi-layer deployment organized by resource lifecycle:

| Layer | Resources | Rationale |
|-------|-----------|-----------|
| **Networking** | Virtual network, subnets, NAT gateway, execution roles, security groups / firewall rules | Shared networking; changes rarely |
| **Data** | Object storage (durable index), shared filesystem (hot cache), replication task | Data plane; independent scaling |
| **Processing** | Serverless function (compiled binary), API gateway (authenticated) | Request plane; scales to zero |
| **Access Control** | Token-based authorizer, usage plans, client keys | Access control for external agent platforms |

**Data flow**: Search indexes are built offline and uploaded to object storage. A replication task syncs them to the shared filesystem. The serverless function reads indexes from the shared filesystem at invocation time (no cold-start download).

> See [`schemas/infrastructure-layer-template.yaml`](../schemas/infrastructure-layer-template.yaml) for layer definitions and [`templates/infrastructure-config-template.yaml`](../templates/infrastructure-config-template.yaml) for a configuration skeleton.

### 4. Developer Integration

IDE extension providing:

- **Search UI**: Webview-based interface for interactive search
- **Virtual file system**: Custom URI scheme for browsing indexed codebases directly in the editor without cloning repositories
- **MCP client integration**: Connects to local MCP server (stdio mode) for agent-assisted search within the IDE

## Data Flow

The system has two distinct data paths depending on the type of MCP server being invoked:

### Search Path (compute-backed)

Used by search-domain MCP servers that require a backend compute service for query execution.

```
[Agent issues MCP tool call (e.g., search, get_file)]
         │
         ▼
[MCP Server: validate input, translate to HTTP request]
         │
         ▼
[API Gateway: authenticate, rate limit, route]
         │
         ▼
[Serverless function: read index from shared filesystem, execute search, paginate via key-value store]
         │
         ▼
[MCP Server: format response as MCP tool result]
         │
         ▼
[Agent receives structured result]
```

Pagination: **Cursor-based** (key-value store with TTL). Cursors are opaque tokens that enable stable iteration even when the underlying index changes between pages.

### Retrieval Path (direct storage access)

Used by retrieval-domain MCP servers that access document storage directly.

```
[Agent issues MCP tool call (e.g., list_documents, get_document)]
         │
         ▼
[MCP Server: validate input, query object storage directly via SDK]
         │
         ▼
[Object storage: return listing or content]
         │
         ▼
[MCP Server: parse metadata / format content, return MCP tool result]
         │
         ▼
[Agent receives structured result]
```

Pagination: **Offset-based** (in-memory). Document metadata is cached locally in the MCP server. This simpler approach is appropriate when the document corpus is small (hundreds to low thousands) and changes infrequently.

> **Note**: The retrieval path bypasses the API gateway and serverless function entirely — the MCP server communicates with object storage directly. This avoids unnecessary latency for simple read operations.

### When to Use Which Path

| Factor | Search Path | Retrieval Path |
|--------|------------|----------------|
| Data volume | Large (millions of files/records) | Small (hundreds to thousands of documents) |
| Query complexity | Full-text search, regex, filtering | Keyword matching, metadata filtering |
| Compute needs | Indexing engine, ranked results | Simple reads, no indexing |
| Pagination | Cursor-based (server-side state) | Offset-based (stateless) |
| Backend dependency | Dedicated compute service required | Object storage SDK only |

## Deployment Modes

| Mode | Components Running | Storage | Auth | Typical Use |
|------|-------------------|---------|------|-------------|
| **Local development** | MCP server (stdio) + search service (HTTP) | Local filesystem | None | Individual developer, testing |
| **Cloud production** | MCP server (HTTP/SSE) + API gateway + serverless function | Object storage + shared filesystem + key-value store | Cloud-native auth / OAuth2 | Multi-tenant, production agents |
| **Hybrid** | MCP server (stdio, local) → API gateway (cloud) | Cloud storage | Cloud-native auth | IDE extension accessing cloud indexes |
