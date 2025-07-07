# FalkorDB MCP Server Documentation

Welcome to the comprehensive documentation for FalkorDB Model Context Protocol (MCP) Server v1.1.0.

## 📚 Documentation Index

### Getting Started
- **[Quick Start Guide](quick-start.md)** - Get up and running in 5 minutes
- **[Installation](installation.md)** - Installation methods and requirements
- **[Configuration](configuration.md)** - Complete configuration reference

### Core Features
- **[Architecture Overview](architecture.md)** - System design and components
- **[API Reference](api-reference.md)** - REST API endpoints and MCP protocol
- **[Multi-Tenancy](multi-tenancy.md)** - Multi-tenant setup and management
- **[Authentication](authentication.md)** - API key and Bearer token authentication

### Deployment & Operations
- **[Docker Deployment](deployment.md)** - Production deployment with Docker
- **[Development Setup](development.md)** - Local development environment
- **[Monitoring & Health](monitoring.md)** - Health checks and observability
- **[Troubleshooting](troubleshooting.md)** - Common issues and solutions

### Advanced Topics
- **[Security](security.md)** - Security considerations and best practices
- **[Performance](performance.md)** - Performance tuning and optimization
- **[Migration Guide](migration.md)** - Upgrading from previous versions
- **[Contributing](../CONTRIBUTING.md)** - Development and contribution guidelines

### Examples & Tutorials
- **[Basic Usage Examples](examples/basic-usage.md)** - Simple API usage patterns
- **[Multi-Tenant Setup](examples/multi-tenant-setup.md)** - Complete multi-tenant configuration
- **[Docker Compose Examples](examples/docker-compose.md)** - Production deployment examples
- **[Integration Patterns](examples/integration-patterns.md)** - Common integration scenarios

## 🚀 What is FalkorDB MCP Server?

FalkorDB MCP Server is a production-ready Model Context Protocol server that provides AI models with seamless access to FalkorDB graph databases. It features:

- **Graph Database Integration**: Direct connection to FalkorDB instances
- **Multi-Tenant Architecture**: Tenant isolation with prefix-based graph naming
- **Flexible Authentication**: API key and JWT bearer token support
- **Production Ready**: Comprehensive testing, monitoring, and deployment tools
- **Zero Breaking Changes**: Fully backward compatible with existing deployments

## 🏗️ Architecture Overview

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   AI Models     │    │  MCP Server     │    │   FalkorDB      │
│                 │    │                 │    │                 │
│ • Claude        │◄──►│ • Authentication│◄──►│ • Graph Storage │
│ • Custom LLMs   │    │ • Multi-Tenancy │    │ • Query Engine  │
│ • Applications  │    │ • API Gateway   │    │ • Data Management│
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

## 🔧 Key Features

### Core Functionality
- **RESTful API**: Standard HTTP endpoints for graph operations
- **Cypher Query Support**: Full Cypher query language support
- **Parameter Binding**: Safe parameter substitution with Unicode support
- **Connection Pooling**: Efficient database connection management

### Multi-Tenancy
- **Tenant Isolation**: Complete data separation between tenants
- **Flexible Authentication**: JWT token-based tenant identification
- **Graph Prefixing**: Automatic tenant prefix for graph names
- **Backward Compatibility**: Optional feature with zero breaking changes

### Security & Reliability
- **Input Validation**: Comprehensive request validation
- **Error Handling**: Structured error responses with proper HTTP status codes
- **Health Monitoring**: Built-in health check endpoints
- **Connection Resilience**: Automatic reconnection and retry logic

### Development & Operations
- **Docker Support**: Production-ready containerization
- **CI/CD Integration**: Automated testing and deployment
- **Comprehensive Testing**: 120+ tests with 99% success rate
- **Monitoring Ready**: Structured logging and observability

## 🎯 Use Cases

### AI Model Integration
- **Memory Systems**: Persistent graph-based memory for AI conversations
- **Knowledge Graphs**: Complex relationship modeling for AI reasoning
- **Context Management**: Multi-session context storage and retrieval

### Multi-Tenant Platforms
- **SaaS Applications**: Tenant-isolated graph data for multi-customer platforms
- **Enterprise Deployments**: Department or team-based data separation
- **Development Environments**: Isolated testing and development graphs

### Data Management
- **Graph Analytics**: Complex relationship analysis and traversal
- **Real-time Queries**: Fast graph queries for live applications
- **Data Integration**: Graph-based data consolidation and analysis

## 🚦 Quick Start

1. **Install with Docker**:
   ```bash
   docker run -d \
     --name falkordb-mcp \
     -p 3000:3000 \
     -e FALKORDB_HOST=your-falkordb-host \
     -e MCP_API_KEY=your-api-key \
     ghcr.io/dragonatorul/falkordb-mcpserver:1.1.0
   ```

2. **Test the connection**:
   ```bash
   curl -H "Authorization: Bearer your-api-key" \
        http://localhost:3000/api/mcp/metadata
   ```

3. **Execute a query**:
   ```bash
   curl -X POST \
        -H "Content-Type: application/json" \
        -H "Authorization: Bearer your-api-key" \
        -d '{"graphName": "test", "query": "CREATE (n:Node {name: \"test\"}) RETURN n"}' \
        http://localhost:3000/api/mcp/context
   ```

For detailed setup instructions, see the [Quick Start Guide](quick-start.md).

## 📖 Version Information

- **Current Version**: v1.1.0
- **Release Date**: 2025-01-07
- **Compatibility**: Node.js 16.x, 18.x, 20.x
- **Container Images**: `ghcr.io/dragonatorul/falkordb-mcpserver:1.1.0`

## 🤝 Support & Community

- **Documentation**: Complete guides and API reference
- **Issues**: [GitHub Issues](https://github.com/Dragonatorul/FalkorDB-MCPServer/issues)
- **Discussions**: [GitHub Discussions](https://github.com/Dragonatorul/FalkorDB-MCPServer/discussions)
- **Contributing**: See [Contributing Guide](../CONTRIBUTING.md)

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](../LICENSE) file for details.

---

For specific implementation details, configuration options, and deployment scenarios, please refer to the individual documentation sections linked above.