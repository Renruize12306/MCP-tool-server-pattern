# Deployment Checklist

Step-by-step checklist for deploying the MCP Tool Access Pattern to a cloud environment. This checklist is intended as a reusable operational template. Adapt terminology to your cloud provider and infrastructure framework.

## Pre-Deployment

- [ ] **Index preparation**: Search indexes are built and uploaded to durable object storage
  - [ ] Index format validated
  - [ ] Object storage versioning enabled
  - [ ] Encryption at rest confirmed

- [ ] **Infrastructure configuration reviewed**
  - [ ] Infrastructure configuration values set for target environment (account, region, network CIDR)
  - [ ] Serverless function memory and concurrency limits appropriate for expected load
  - [ ] API gateway throttling limits configured
  - [ ] TTL enabled on pagination key-value store

- [ ] **Security review completed** (see [Security Audit Checklist](security-audit-checklist.md))

- [ ] **Execution roles follow least privilege**
  - [ ] Compute role: only object storage read, key-value store read/write (pagination), filesystem mount
  - [ ] API gateway: only invoke compute function
  - [ ] No wildcard resource identifiers

## Networking Layer Deployment

- [ ] Deploy `NetworkingLayer`
  - [ ] Virtual network created with expected CIDR and availability zone count
  - [ ] NAT gateway reachable from private subnets
  - [ ] Security groups created (compute SG, filesystem SG)
  - [ ] Verify filesystem SG allows mount protocol from compute SG only

## Data Layer Deployment

- [ ] Deploy `DataLayer`
  - [ ] Object storage created with versioning and lifecycle rules
  - [ ] Shared filesystem created with encryption at rest
  - [ ] Filesystem mount targets in each private subnet
  - [ ] Replication task created and tested
    - [ ] Manual replication execution succeeds
    - [ ] Files appear on shared filesystem at expected mount path
    - [ ] Schedule configured (e.g., hourly)

## Processing Layer Deployment

- [ ] Deploy `ProcessingLayer`
  - [ ] Serverless function deployed (compiled binary, ARM architecture)
  - [ ] Function attached to virtual network (private subnets, compute SG)
  - [ ] Shared filesystem access point mounted at configured path
  - [ ] Environment variables set (`INDEX_PATH`, `PAGINATION_TABLE`, `LOG_LEVEL`)
  - [ ] API gateway deployed with authentication enabled
  - [ ] All routes mapped to function integration

- [ ] **Smoke test: health checks**
  - [ ] `GET /livez` returns 200
  - [ ] `GET /readyz` returns 200 (index loaded)

- [ ] **Smoke test: search**
  - [ ] `POST /search` with a known query returns results
  - [ ] Pagination cursor works (request page 1, use cursor for page 2)
  - [ ] Content retrieval endpoint returns a file

## Access Control Layer Deployment (if applicable)

- [ ] Deploy `AccessControlLayer`
  - [ ] Token authorizer function deployed and configured
  - [ ] Usage plan created with appropriate rate limits
  - [ ] Client keys generated for each consumer
  - [ ] Test token exchange flow end-to-end

## MCP Server Verification

- [ ] **Local mode (stdio)**
  - [ ] MCP server starts without errors
  - [ ] All tools listed in tool discovery response
  - [ ] Each tool returns expected output with test input

- [ ] **Cloud mode (HTTP)**
  - [ ] MCP server connects to API gateway endpoint
  - [ ] Authentication succeeds
  - [ ] Tool calls routed through API gateway to serverless function
  - [ ] Responses match local mode output for same inputs

## Rollback Plan

- [ ] **Infrastructure rollback procedure documented**
  - [ ] Previous layer versions identified (diff run before deploy)
  - [ ] Layers should be rolled back in reverse dependency order: Access Control -> Processing -> Data -> Networking

- [ ] **Compute rollback**
  - [ ] Previous function version/alias available for instant rollback
  - [ ] If using provisioned concurrency, verify rollback target has capacity

- [ ] **Data rollback**
  - [ ] Object storage versioning allows restoring previous index versions
  - [ ] Pagination store can be cleared (cursors are ephemeral)
  - [ ] Re-run replication task to restore previous index from object storage

- [ ] **Traffic cutover**
  - [ ] API gateway stage can be repointed to previous function version
  - [ ] DNS or client config can revert to previous endpoint if needed

## Post-Deployment

- [ ] **Monitoring configured**
  - [ ] Log groups created for serverless function
  - [ ] Metrics endpoint accessible (if applicable)
  - [ ] Alerts set for error rate thresholds
  - [ ] Alerts set for latency P99 thresholds

- [ ] **Documentation updated**
  - [ ] Endpoint URLs recorded
  - [ ] Client key distribution documented
  - [ ] Runbook for common operational scenarios created

- [ ] **Tag release**
  - [ ] Git tag created (e.g., `v1.0.0`)
  - [ ] CHANGELOG updated
