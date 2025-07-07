# Architecture Overview

This document provides a comprehensive overview of the FalkorDB MCP Server architecture, including system design, component interactions, and data flow patterns.

## System Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client Layer                            │
├─────────────────────────────────────────────────────────────────┤
│  AI Models    │  Custom Apps  │  MCP Clients  │  Direct HTTP    │
│  • Claude     │  • Dashboards │  • FastMCP    │  • curl/Postman │
│  • GPT        │  • Analytics  │  • Custom     │  • SDKs         │
└─────────────────┬───────────────┬───────────────┬───────────────┘
                  │               │               │
                  ▼               ▼               ▼
┌─────────────────────────────────────────────────────────────────┐
│                     HTTP/REST Layer                            │
├─────────────────────────────────────────────────────────────────┤
│                    Express.js Router                           │
│  /health  │  /api/mcp/metadata  │  /api/mcp/context  │  /api/mcp/graphs │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Middleware Layer                            │
├─────────────────────────────────────────────────────────────────┤
│  Auth Middleware  │  Error Handler  │  Request Validator        │
│  • API Key Auth   │  • Structured   │  • Schema Validation      │
│  • Bearer Token   │    Responses    │  • Parameter Sanitization │
│  • Multi-Tenant   │  • HTTP Status  │  • Input Validation       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Controller Layer                             │
├─────────────────────────────────────────────────────────────────┤
│              MCP Controller                                     │
│  • Request Processing    │  • Response Formatting              │
│  • Business Logic       │  • Error Handling                   │
│  • Tenant Resolution    │  • Logging & Monitoring             │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Service Layer                               │
├─────────────────────────────────────────────────────────────────┤
│   FalkorDB Service    │         Tenant Graph Service           │
│   • Connection Mgmt   │         • Graph Name Resolution        │
│   • Query Execution   │         • Tenant Isolation            │
│   • Parameter Subst   │         • Prefix Management           │
│   • Error Recovery    │         • Multi-Tenant Logic          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Database Layer                              │
├─────────────────────────────────────────────────────────────────┤
│                       FalkorDB                                 │
│  • Graph Storage      │  • Cypher Engine    │  • Data Persistence │
│  • Query Processing   │  • Indexing         │  • Backup/Recovery   │
│  • Relationship Mgmt  │  • Performance Opt  │  • Clustering        │
└─────────────────────────────────────────────────────────────────┘
```

## Core Components

### 1. HTTP Server (Express.js)

**Location**: `src/index.ts`

**Responsibilities**:
- HTTP request handling and routing
- Middleware orchestration
- Server lifecycle management
- Graceful shutdown handling

**Key Features**:
```typescript
// Server setup with middleware
app.use(express.json({ limit: '10mb' }));
app.use(express.urlencoded({ extended: true }));
app.use('/api/mcp', authenticateMCP);
app.use('/api/mcp', mcpRouter);
```

### 2. Authentication Middleware

**Location**: `src/middleware/auth.middleware.ts`, `src/middleware/bearer.middleware.ts`

**Authentication Flow**:
```
Request → Auth Middleware → Validation → Tenant Extraction → Next()
```

**Supported Methods**:
- **API Key Authentication**: Simple bearer token validation
- **JWT Bearer Token**: Complete JWT validation with JWKS
- **Multi-Tenant Context**: Automatic tenant ID extraction

**Implementation**:
```typescript
// Dual authentication support
if (authMode === 'bearer' && enableMultiTenancy) {
  return bearerAuthMiddleware(req, res, next);
} else {
  return apiKeyAuth(req, res, next);
}
```

### 3. MCP Controller

**Location**: `src/controllers/mcp.controller.ts`

**Core Methods**:
- `processContextRequest()`: Execute graph queries
- `listGraphs()`: Retrieve available graphs
- `getMetadata()`: Server capability information
- `healthCheck()`: Service health status

**Request Processing Flow**:
```typescript
Request → Validation → Tenant Resolution → Query Execution → Response Formatting
```

### 4. FalkorDB Service

**Location**: `src/services/falkordb.service.ts`

**Core Responsibilities**:
- Database connection management
- Query execution with parameter substitution
- Connection pooling and retry logic
- Error handling and recovery

**Key Features**:
```typescript
// Singleton pattern with lazy initialization
class FalkorDBService {
  private static instance: FalkorDBService;
  private client: FalkorDB | null = null;
  private initPromise: Promise<void> | null = null;
}
```

**Parameter Substitution Workaround**:
```typescript
// Unicode-safe parameter substitution
private substituteParameters(query: string, params?: Record<string, any>): string {
  let substitutedQuery = query;
  for (const [key, value] of Object.entries(params)) {
    const escapedValue = this.escapeValue(value);
    substitutedQuery = substitutedQuery.replace(new RegExp(`\\$${key}\\b`, 'g'), escapedValue);
  }
  return substitutedQuery;
}
```

### 5. Tenant Graph Service

**Location**: `src/services/tenant-graph.service.ts`

**Multi-Tenant Graph Resolution**:
```typescript
// Graph name resolution logic
static resolveGraphName(graphName: string, tenantId?: string): string {
  if (!config.enableMultiTenancy || !config.tenantGraphPrefix || !tenantId) {
    return graphName; // No modification
  }
  return `${tenantId}_${graphName}`; // Tenant prefix
}
```

**Tenant Isolation**:
- **Single-tenant**: `memories` → `memories`
- **Multi-tenant**: `memories` → `tenant123_memories`
- **Shared graphs**: `shared_data` → `shared_data` (no prefix)

## Data Flow Architecture

### Request Processing Flow

```
1. HTTP Request
   ↓
2. Authentication Middleware
   ├── API Key Validation
   └── JWT Token Validation + Tenant Extraction
   ↓
3. Request Validation
   ├── Schema Validation
   ├── Parameter Sanitization
   └── Input Validation
   ↓
4. MCP Controller
   ├── Business Logic
   ├── Tenant Context Resolution
   └── Request Routing
   ↓
5. Service Layer
   ├── FalkorDB Service
   │   ├── Connection Management
   │   ├── Parameter Substitution
   │   └── Query Execution
   └── Tenant Graph Service
       ├── Graph Name Resolution
       └── Tenant Isolation
   ↓
6. Database Layer
   ├── FalkorDB Query Engine
   ├── Graph Storage
   └── Result Processing
   ↓
7. Response Processing
   ├── Result Formatting
   ├── Error Handling
   └── HTTP Response
```

### Multi-Tenant Request Flow

```
JWT Token → Tenant Extraction → Graph Resolution → Query Execution
     │              │                    │              │
     ▼              ▼                    ▼              ▼
"eyJ0eXAi..."  "tenant123"     "tenant123_memories"   MATCH (n) ...
```

## Configuration Architecture

### Environment-Based Configuration

**Location**: `src/config/index.ts`

```typescript
export interface Config {
  // Server Configuration
  port: number;
  nodeEnv: string;
  
  // FalkorDB Configuration
  falkorDB: {
    host: string;
    port: number;
    username?: string;
    password?: string;
  };
  
  // Authentication Configuration
  mcpApiKey: string;
  
  // Multi-Tenancy Configuration
  enableMultiTenancy: boolean;
  multiTenantAuthMode: 'api-key' | 'bearer';
  tenantGraphPrefix: boolean;
  
  // Bearer Token Configuration
  bearer: {
    jwksUri?: string;
    issuer?: string;
    algorithm: string;
    audience?: string;
  };
}
```

### Configuration Layers

```
Environment Variables → Default Values → Runtime Configuration
        │                     │                    │
        ▼                     ▼                    ▼
process.env.FALKORDB_HOST → 'localhost' → config.falkorDB.host
```

## Error Handling Architecture

### Structured Error Responses

```typescript
interface ErrorResponse {
  error: {
    type: string;
    message: string;
    code?: string;
    details?: any;
  };
  timestamp: string;
  path: string;
  tenant?: string;
}
```

### Error Categories

1. **Authentication Errors** (401)
   - Invalid API key
   - JWT validation failures
   - Missing authorization

2. **Validation Errors** (400)
   - Invalid request format
   - Missing required parameters
   - Parameter type mismatches

3. **Database Errors** (500)
   - Connection failures
   - Query execution errors
   - Timeout errors

4. **Multi-Tenancy Errors** (403)
   - Tenant access violations
   - Graph isolation breaches
   - Invalid tenant context

## Security Architecture

### Defense in Depth

```
┌─────────────────────────────────────────┐
│           Application Security          │
├─────────────────────────────────────────┤
│ Input Validation │ Output Sanitization  │
│ Parameter Escape │ Error Information    │
│ SQL Injection    │ Disclosure Control   │
│ Prevention       │                      │
└─────────────────────────────────────────┘
                    │
┌─────────────────────────────────────────┐
│          Authentication Security        │
├─────────────────────────────────────────┤
│ API Key Validation │ JWT Verification   │
│ Bearer Token Auth  │ JWKS Integration   │
│ Multi-Tenant      │ Token Expiration   │
│ Context           │ Validation         │
└─────────────────────────────────────────┘
                    │
┌─────────────────────────────────────────┐
│            Network Security             │
├─────────────────────────────────────────┤
│ HTTPS/TLS        │ Rate Limiting       │
│ CORS Protection  │ Request Size Limits │
│ Security Headers │ Timeout Protection  │
└─────────────────────────────────────────┘
```

### Tenant Isolation

```
Tenant A Requests → tenant_a_* graphs only
Tenant B Requests → tenant_b_* graphs only
Shared Requests   → shared_* graphs (if configured)
```

## Performance Architecture

### Connection Management

```typescript
// Singleton pattern with connection pooling
class FalkorDBService {
  private client: FalkorDB | null = null;
  
  // Lazy initialization
  private async ensureInitialized(): Promise<void> {
    if (!this.client) {
      await this.init();
    }
  }
}
```

### Query Optimization

- **Parameter Substitution**: Client-side parameter binding
- **Connection Reuse**: Persistent database connections
- **Timeout Management**: Configurable query timeouts
- **Error Recovery**: Automatic reconnection on failures

### Scalability Considerations

1. **Horizontal Scaling**: Stateless service design
2. **Database Scaling**: FalkorDB cluster support
3. **Load Balancing**: Standard HTTP load balancer compatibility
4. **Caching**: In-memory connection and configuration caching

## Testing Architecture

### Test Structure

```
src/tests/
├── unit/                 # Unit tests
│   ├── services/         # Service layer tests
│   ├── controllers/      # Controller tests
│   └── middleware/       # Middleware tests
├── integration/          # Integration tests
│   ├── api/             # API endpoint tests
│   ├── database/        # Database interaction tests
│   └── multi-tenant/    # Multi-tenancy tests
└── utils/               # Test utilities
    ├── helpers/         # Test helper functions
    ├── fixtures/        # Test data
    └── mocks/           # Mock implementations
```

### Test Categories

1. **Unit Tests**: Component isolation testing
2. **Integration Tests**: Component interaction testing
3. **Contract Tests**: API contract validation
4. **Multi-Tenancy Tests**: Tenant isolation verification
5. **Performance Tests**: Load and stress testing

## Monitoring Architecture

### Health Check Endpoints

```typescript
// Application health
GET /health → Service status

// Database health  
GET /api/mcp/metadata → Database connectivity

// Authentication health
GET /api/mcp/graphs → Auth pipeline validation
```

### Observability

- **Structured Logging**: JSON log format with correlation IDs
- **Error Tracking**: Detailed error context and stack traces
- **Performance Metrics**: Query execution times and throughput
- **Tenant Metrics**: Per-tenant usage and performance tracking

## Deployment Architecture

### Container Structure

```dockerfile
FROM node:18-alpine

# Application layer
COPY package*.json ./
RUN npm ci --only=production

# Source code layer
COPY dist/ ./dist/

# Runtime configuration
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

### Environment Support

- **Development**: Hot-reload, debug logging, test database
- **Staging**: Production configuration, test data
- **Production**: Optimized performance, secure configuration

---

*This documentation was generated by Claude Code, an AI assistant, to provide comprehensive technical documentation for the FalkorDB MCP Server architecture.*