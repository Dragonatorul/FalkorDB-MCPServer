# Configuration Reference

Complete configuration guide for FalkorDB MCP Server with detailed descriptions of all environment variables, configuration options, and deployment scenarios.

## Environment Variables

### Core Server Configuration

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `NODE_ENV` | string | `development` | Node.js environment (`development`, `production`, `test`) |
| `PORT` | number | `3000` | HTTP server port |

**Example:**
```env
NODE_ENV=production
PORT=8080
```

### FalkorDB Database Configuration

| Variable | Type | Default | Required | Description |
|----------|------|---------|----------|-------------|
| `FALKORDB_HOST` | string | `localhost` | ✅ | FalkorDB server hostname or IP |
| `FALKORDB_PORT` | number | `6379` | ✅ | FalkorDB server port |
| `FALKORDB_USERNAME` | string | - | ❌ | FalkorDB username (if authentication enabled) |
| `FALKORDB_PASSWORD` | string | - | ❌ | FalkorDB password (if authentication enabled) |

**Example:**
```env
# Local development
FALKORDB_HOST=localhost
FALKORDB_PORT=6379

# Production with authentication
FALKORDB_HOST=falkordb.internal.company.com
FALKORDB_PORT=6379
FALKORDB_USERNAME=mcp_service
FALKORDB_PASSWORD=secure_production_password
```

### Authentication Configuration

| Variable | Type | Default | Required | Description |
|----------|------|---------|----------|-------------|
| `MCP_API_KEY` | string | - | ✅ | API key for request authentication |

**Example:**
```env
# Development
MCP_API_KEY=dev-api-key-12345

# Production (use strong, random keys)
MCP_API_KEY=prod-a8f5b2c1d4e7f9g2h5j8k1m4n7p0q3r6s9t2v5w8x1y4z7
```

### Multi-Tenancy Configuration

| Variable | Type | Default | Required | Description |
|----------|------|---------|----------|-------------|
| `ENABLE_MULTI_TENANCY` | boolean | `false` | ❌ | Enable multi-tenant features |
| `MULTI_TENANT_AUTH_MODE` | enum | `api-key` | ❌ | Authentication mode (`api-key`, `bearer`) |
| `TENANT_GRAPH_PREFIX` | boolean | `false` | ❌ | Enable tenant prefix for graph names |

**Values for `MULTI_TENANT_AUTH_MODE`:**
- `api-key`: Use API key authentication (single tenant)
- `bearer`: Use JWT bearer token authentication (multi-tenant)

**Example:**
```env
# Enable multi-tenancy with Bearer token authentication
ENABLE_MULTI_TENANCY=true
MULTI_TENANT_AUTH_MODE=bearer
TENANT_GRAPH_PREFIX=true
```

### JWT Bearer Token Configuration

| Variable | Type | Default | Required | Description |
|----------|------|---------|----------|-------------|
| `BEARER_JWKS_URI` | string | - | ⚠️* | JWKS endpoint URL for JWT validation |
| `BEARER_ISSUER` | string | - | ⚠️* | Expected JWT issuer |
| `BEARER_ALGORITHM` | string | `RS256` | ❌ | JWT signing algorithm |
| `BEARER_AUDIENCE` | string | - | ❌ | Expected JWT audience |

**⚠️ Required when `MULTI_TENANT_AUTH_MODE=bearer`**

**Example:**
```env
# Auth0 configuration
BEARER_JWKS_URI=https://your-domain.auth0.com/.well-known/jwks.json
BEARER_ISSUER=https://your-domain.auth0.com/
BEARER_ALGORITHM=RS256
BEARER_AUDIENCE=falkordb-mcp-server

# Custom OAuth2 provider
BEARER_JWKS_URI=https://auth.company.com/.well-known/jwks.json
BEARER_ISSUER=https://auth.company.com
BEARER_ALGORITHM=RS256
BEARER_AUDIENCE=internal-api
```

## Configuration Patterns

### 1. Single-Tenant Setup (Default)

**Use Case**: Simple deployment with one application or user

```env
# Basic configuration
NODE_ENV=production
PORT=3000
FALKORDB_HOST=localhost
FALKORDB_PORT=6379
MCP_API_KEY=your-secure-api-key

# Multi-tenancy disabled (default)
ENABLE_MULTI_TENANCY=false
```

**Features**:
- ✅ Simple API key authentication
- ✅ Direct graph access without prefixes
- ✅ Zero configuration overhead
- ✅ Backward compatible with v1.0.x

### 2. Multi-Tenant Setup

**Use Case**: SaaS platforms, enterprise deployments with tenant isolation

```env
# Server configuration
NODE_ENV=production
PORT=3000
FALKORDB_HOST=falkordb.internal.company.com
FALKORDB_PORT=6379

# Multi-tenancy enabled
ENABLE_MULTI_TENANCY=true
MULTI_TENANT_AUTH_MODE=bearer
TENANT_GRAPH_PREFIX=true

# JWT authentication
BEARER_JWKS_URI=https://auth.company.com/.well-known/jwks.json
BEARER_ISSUER=https://auth.company.com
BEARER_ALGORITHM=RS256
BEARER_AUDIENCE=falkordb-mcp-api
```

**Features**:
- ✅ JWT-based tenant identification
- ✅ Automatic graph name prefixing
- ✅ Complete tenant isolation
- ✅ JWKS validation for security

### 3. Development Setup

**Use Case**: Local development and testing

```env
# Development server
NODE_ENV=development
PORT=3000
FALKORDB_HOST=localhost
FALKORDB_PORT=6379
MCP_API_KEY=dev-api-key

# Optional: Test multi-tenancy locally
ENABLE_MULTI_TENANCY=true
MULTI_TENANT_AUTH_MODE=bearer
TENANT_GRAPH_PREFIX=true

# Test JWT configuration (mock/local auth server)
BEARER_JWKS_URI=http://localhost:4000/.well-known/jwks.json
BEARER_ISSUER=http://localhost:4000
BEARER_ALGORITHM=RS256
```

**Features**:
- ✅ Hot-reload development
- ✅ Detailed logging
- ✅ Local multi-tenancy testing
- ✅ Mock authentication support

### 4. Docker Compose Setup

**Use Case**: Complete stack deployment with FalkorDB

```yaml
# docker-compose.yml
version: '3.8'
services:
  falkordb:
    image: falkordb/falkordb:latest
    ports:
      - "6379:6379"
    volumes:
      - falkordb_data:/data
    
  mcp-server:
    image: ghcr.io/dragonatorul/falkordb-mcpserver:1.1.0
    ports:
      - "3000:3000"
    environment:
      FALKORDB_HOST: falkordb
      FALKORDB_PORT: 6379
      MCP_API_KEY: ${MCP_API_KEY}
      ENABLE_MULTI_TENANCY: ${ENABLE_MULTI_TENANCY:-false}
    depends_on:
      - falkordb

volumes:
  falkordb_data:
```

**Environment file (.env)**:
```env
MCP_API_KEY=production-secure-key
ENABLE_MULTI_TENANCY=true
```

## Authentication Configuration Details

### API Key Authentication

**Configuration**:
```env
MCP_API_KEY=your-api-key
ENABLE_MULTI_TENANCY=false
```

**Request Format**:
```bash
curl -H "Authorization: Bearer your-api-key" \
     http://localhost:3000/api/mcp/metadata
```

**Security Considerations**:
- Use strong, random API keys (32+ characters)
- Rotate keys regularly
- Use different keys for different environments
- Store keys securely (environment variables, secrets management)

### JWT Bearer Token Authentication

**Configuration**:
```env
ENABLE_MULTI_TENANCY=true
MULTI_TENANT_AUTH_MODE=bearer
BEARER_JWKS_URI=https://auth-provider.com/.well-known/jwks.json
BEARER_ISSUER=https://auth-provider.com
```

**JWT Token Requirements**:
```json
{
  "iss": "https://auth-provider.com",
  "aud": "falkordb-mcp-server",
  "sub": "user123",
  "tenant_id": "tenant_abc",
  "exp": 1640995200,
  "iat": 1640908800
}
```

**Request Format**:
```bash
curl -H "Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9..." \
     http://localhost:3000/api/mcp/context
```

## Multi-Tenancy Configuration Details

### Graph Name Resolution

**With `TENANT_GRAPH_PREFIX=true`**:

| Original Graph Name | Tenant ID | Resolved Name |
|-------------------|-----------|---------------|
| `memories` | `tenant123` | `tenant123_memories` |
| `analytics` | `org_456` | `org_456_analytics` |
| `shared_data` | `tenant123` | `shared_data` (no prefix for shared_*) |

**Configuration Example**:
```env
ENABLE_MULTI_TENANCY=true
TENANT_GRAPH_PREFIX=true
```

### Tenant Isolation Modes

**1. Prefix-Based Isolation (Current)**:
```env
TENANT_GRAPH_PREFIX=true
```
- Graphs: `tenant_a_memories`, `tenant_b_memories`
- Isolation: Application-level prefixing
- Performance: Minimal overhead

**2. No Isolation (Shared)**:
```env
TENANT_GRAPH_PREFIX=false
```
- Graphs: `memories`, `analytics`
- Isolation: None (shared graphs)
- Use Case: Trusted multi-user environments

## Environment File Examples

### `.env.development`
```env
# Development Configuration
NODE_ENV=development
PORT=3000

# Local FalkorDB
FALKORDB_HOST=localhost
FALKORDB_PORT=6379

# Development API key
MCP_API_KEY=dev-key-12345

# Multi-tenancy for testing
ENABLE_MULTI_TENANCY=true
MULTI_TENANT_AUTH_MODE=bearer
TENANT_GRAPH_PREFIX=true

# Local auth server
BEARER_JWKS_URI=http://localhost:4000/.well-known/jwks.json
BEARER_ISSUER=http://localhost:4000
```

### `.env.staging`
```env
# Staging Configuration
NODE_ENV=production
PORT=3000

# Staging FalkorDB
FALKORDB_HOST=falkordb-staging.internal.company.com
FALKORDB_PORT=6379
FALKORDB_USERNAME=mcp_staging
FALKORDB_PASSWORD=staging_password

# Staging API key
MCP_API_KEY=staging-key-abcdef123456

# Multi-tenancy enabled
ENABLE_MULTI_TENANCY=true
MULTI_TENANT_AUTH_MODE=bearer
TENANT_GRAPH_PREFIX=true

# Staging auth server
BEARER_JWKS_URI=https://auth-staging.company.com/.well-known/jwks.json
BEARER_ISSUER=https://auth-staging.company.com
BEARER_AUDIENCE=falkordb-mcp-staging
```

### `.env.production`
```env
# Production Configuration
NODE_ENV=production
PORT=3000

# Production FalkorDB cluster
FALKORDB_HOST=falkordb-cluster.prod.company.com
FALKORDB_PORT=6379
FALKORDB_USERNAME=mcp_production
FALKORDB_PASSWORD=${FALKORDB_PROD_PASSWORD}

# Production API key (from secrets management)
MCP_API_KEY=${MCP_PROD_API_KEY}

# Multi-tenancy enabled
ENABLE_MULTI_TENANCY=true
MULTI_TENANT_AUTH_MODE=bearer
TENANT_GRAPH_PREFIX=true

# Production auth server
BEARER_JWKS_URI=https://auth.company.com/.well-known/jwks.json
BEARER_ISSUER=https://auth.company.com
BEARER_ALGORITHM=RS256
BEARER_AUDIENCE=falkordb-mcp-production
```

## Configuration Validation

### Startup Validation

The server validates configuration on startup:

```typescript
// Required configuration check
if (!config.mcpApiKey) {
  throw new Error('MCP_API_KEY is required');
}

// Multi-tenancy validation
if (config.enableMultiTenancy && config.multiTenantAuthMode === 'bearer') {
  if (!config.bearer.jwksUri) {
    throw new Error('BEARER_JWKS_URI is required for bearer authentication');
  }
}
```

### Runtime Configuration Checks

```bash
# Check configuration endpoint
curl -H "Authorization: Bearer your-api-key" \
     http://localhost:3000/api/mcp/metadata

# Response includes configuration status
{
  "provider": "falkordb-mcp-server",
  "version": "1.1.0",
  "capabilities": {
    "multiTenancy": true,
    "authMode": "bearer",
    "graphPrefixing": true
  }
}
```

## Security Best Practices

### API Key Security
```env
# ❌ Bad: Weak, predictable keys
MCP_API_KEY=12345
MCP_API_KEY=password

# ✅ Good: Strong, random keys
MCP_API_KEY=mcp_prod_k8h3n9m2q5t8w1z4x7c0v3b6n9m2q5t8w1z4
```

### JWT Configuration Security
```env
# ✅ Use HTTPS URLs for JWKS
BEARER_JWKS_URI=https://auth.company.com/.well-known/jwks.json

# ✅ Verify issuer matches your auth provider
BEARER_ISSUER=https://auth.company.com

# ✅ Use strong algorithms
BEARER_ALGORITHM=RS256
```

### Environment Security
- Use secrets management systems (AWS Secrets Manager, Azure Key Vault, etc.)
- Never commit production credentials to version control
- Use different credentials for each environment
- Implement credential rotation policies

## Configuration Troubleshooting

### Common Issues

**1. Connection Failed to FalkorDB**
```env
# Check these settings
FALKORDB_HOST=correct-hostname
FALKORDB_PORT=6379  # Verify port
```

**2. Authentication Errors**
```env
# Verify API key is set
MCP_API_KEY=your-actual-key

# For JWT: Check JWKS URL is accessible
BEARER_JWKS_URI=https://reachable-url/.well-known/jwks.json
```

**3. Multi-Tenancy Not Working**
```env
# Ensure all required settings are enabled
ENABLE_MULTI_TENANCY=true
MULTI_TENANT_AUTH_MODE=bearer
TENANT_GRAPH_PREFIX=true
```

### Configuration Validation Commands

```bash
# Test FalkorDB connection
curl -H "Authorization: Bearer $MCP_API_KEY" \
     http://localhost:3000/health

# Test JWT configuration (requires valid token)
curl -H "Authorization: Bearer $JWT_TOKEN" \
     http://localhost:3000/api/mcp/metadata

# Check graph prefixing (with multi-tenancy)
curl -X POST \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer $JWT_TOKEN" \
     -d '{"graphName": "test", "query": "RETURN 1"}' \
     http://localhost:3000/api/mcp/context
```

---

*This documentation was generated by Claude Code, an AI assistant, to provide comprehensive configuration guidance for the FalkorDB MCP Server.*