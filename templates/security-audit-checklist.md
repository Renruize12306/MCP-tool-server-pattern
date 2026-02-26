# Security Audit Checklist

Security review checklist for MCP Tool Access Pattern deployments. This checklist covers authentication, authorization, data protection, and operational security concerns specific to deploying MCP servers in cloud environments. Adapt terminology to your cloud provider.

## Authentication

- [ ] **API gateway authentication enabled**
  - [ ] Cloud-native auth configured for all routes
  - [ ] No unauthenticated endpoints (except health checks, if intentional)
  - [ ] Health check endpoints (`/livez`, `/readyz`) assessed: do they expose sensitive info?

- [ ] **Token authorizer (if using Access Control Layer)**
  - [ ] Token validation logic reviewed
  - [ ] Token expiration enforced
  - [ ] Scopes properly restrict access (e.g., `search:read` cannot write)
  - [ ] Client credentials rotated on a schedule

- [ ] **MCP transport security**
  - [ ] HTTP mode uses TLS (gateway enforces HTTPS)
  - [ ] stdio mode only used in local/trusted environments
  - [ ] No credentials embedded in MCP server configuration files

## Authorization

- [ ] **Compute execution role follows least privilege**
  - [ ] Object storage: read-only access to index bucket only
  - [ ] Key-value store: read/write to pagination table only
  - [ ] Shared filesystem: mount + read only (no write to index data)
  - [ ] No wildcard resource identifiers
  - [ ] No role-assumption permissions unless explicitly needed

- [ ] **API gateway resource policies**
  - [ ] Source IP restrictions applied (if applicable)
  - [ ] Cross-account access explicitly denied unless needed
  - [ ] Web application firewall rules attached (if applicable)

- [ ] **Per-client rate limiting**
  - [ ] Usage plans enforce per-client request quotas
  - [ ] Burst limits prevent single-client saturation

## Data Protection

- [ ] **Encryption at rest**
  - [ ] Object storage: encryption enabled
  - [ ] Shared filesystem: encryption at rest enabled
  - [ ] Key-value store: encryption at rest enabled

- [ ] **Encryption in transit**
  - [ ] All API calls over HTTPS/TLS
  - [ ] Filesystem mount uses encryption in transit
  - [ ] Replication transfers encrypted

- [ ] **Sensitive data handling**
  - [ ] Search indexes do not contain secrets (credentials, API keys, tokens)
  - [ ] Search results do not leak sensitive content to unauthorized clients
  - [ ] Pagination cursors are opaque (do not encode query details)
  - [ ] Function environment variables do not contain long-lived secrets (use a secrets manager)

## Network Security

- [ ] **Network isolation**
  - [ ] Serverless function runs in private subnets (no public IP)
  - [ ] Filesystem mount targets in private subnets only
  - [ ] NAT gateway for outbound access (if needed)

- [ ] **Security group rules**
  - [ ] Filesystem SG: ingress only from compute SG on mount port
  - [ ] Compute SG: egress only (no ingress from internet)
  - [ ] No overly permissive rules (0.0.0.0/0 ingress)

- [ ] **API gateway endpoint type**
  - [ ] Regional or private endpoint (not edge-optimized, unless intentional)
  - [ ] Private network endpoint for internal access (if applicable)

## Operational Security

- [ ] **Logging**
  - [ ] API gateway access logs enabled
  - [ ] Function execution logs collected
  - [ ] Log retention policy set (not infinite)
  - [ ] No sensitive data in log output (query content assessed)

- [ ] **Monitoring and alerting**
  - [ ] 4xx/5xx error rate alerts configured
  - [ ] Unusual traffic patterns detectable (spike in requests from single client)
  - [ ] Function throttling alerts

- [ ] **Incident response**
  - [ ] Client keys can be revoked quickly
  - [ ] Function concurrency can be set to zero to halt traffic
  - [ ] Runbook exists for "compromised client key" scenario
  - [ ] Runbook exists for "unauthorized data access" scenario

## MCP-Specific Security Risks

- [ ] **Tool response injection**
  - [ ] Tool outputs do not contain unescaped content that could manipulate agent behavior
  - [ ] Retrieved content is treated as data, not instructions, by the consuming agent
  - [ ] Document content is sanitized or flagged if it contains prompt-like patterns

- [ ] **Tool input validation**
  - [ ] All MCP tool inputs are validated against their schema before backend calls
  - [ ] Regex patterns in search queries are bounded (no catastrophic backtracking)
  - [ ] Path inputs are sanitized (no path traversal: `../`, absolute paths)

- [ ] **Tool enumeration and discovery**
  - [ ] Tool list does not expose internal system architecture to unauthorized clients
  - [ ] Tool descriptions do not leak sensitive operational details

- [ ] **Agent-to-agent isolation**
  - [ ] One agent's MCP session cannot access another agent's pagination cursors
  - [ ] Rate limiting is enforced per-agent-identity, not just per-IP

## Supply Chain

- [ ] **Dependencies reviewed**
  - [ ] Backend dependencies: checksums verified, no unvetted packages
  - [ ] MCP server dependencies: pinned versions, no known CVEs
  - [ ] Infrastructure framework constructs: pinned to known-good versions
  - [ ] Container images (if any): from trusted registries, pinned by digest

- [ ] **Build pipeline**
  - [ ] Function binary built from source in CI (not downloaded pre-built)
  - [ ] Infrastructure deployment runs from a locked dependency set
  - [ ] Deployment requires approval for production environment
