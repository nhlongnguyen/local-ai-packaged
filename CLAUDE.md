# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Architecture Overview

This is a **local AI development stack** that provides a complete self-hosted AI environment using Docker Compose. The stack combines multiple AI services with workflow automation and supporting infrastructure.

### Core Services Architecture

The architecture follows a microservices pattern with these main components:

- **n8n**: Workflow automation and orchestration hub - the primary interface for creating AI agents and workflows
- **Supabase**: Complete backend-as-a-service with PostgreSQL database, authentication, and real-time features
- **Open WebUI**: ChatGPT-like interface for interacting with local LLMs and n8n agents
- **Ollama**: Local LLM inference server supporting CPU and GPU acceleration
- **Vector Storage**: Both Qdrant (high-performance) and Supabase (integrated option) for RAG applications
- **Neo4j**: Graph database for knowledge graphs and advanced RAG patterns (GraphRAG, LightRAG)
- **Caddy**: Reverse proxy with automatic HTTPS/TLS for production deployments
- **Langfuse**: LLM observability and monitoring platform
- **SearXNG**: Privacy-focused metasearch engine for web search capabilities

### Key Integration Points

- **n8n ↔ Open WebUI**: Via `n8n_pipe.py` function that enables chat interface to trigger n8n workflows
- **n8n ↔ Supabase**: Database operations, authentication, and data storage for AI agents
- **n8n ↔ Ollama**: Local LLM inference for all AI operations
- **Caddy**: Routes all external traffic to appropriate services with domain-based routing

## Development Commands

### Starting Services

Use the Python orchestration script with hardware-specific profiles:

```bash
# For Nvidia GPU users
python start_services.py --profile gpu-nvidia

# For AMD GPU users (Linux only)
python start_services.py --profile gpu-amd

# For CPU-only or Mac users
python start_services.py --profile cpu

# For Mac users with local Ollama (recommended)
python start_services.py --profile none
```

### Environment Configurations

```bash
# Private environment (default) - all ports accessible locally
python start_services.py --profile gpu-nvidia --environment private

# Public environment - only ports 80/443 exposed for production
python start_services.py --profile gpu-nvidia --environment public

# Rebuild containers to pick up environment variable changes
python start_services.py --profile gpu-nvidia --rebuild

# Combine rebuild with environment
python start_services.py --profile gpu-nvidia --environment public --rebuild
```

### Service Management

```bash
# Stop all services
docker compose -p localai -f docker-compose.yml --profile <your-profile> down

# Update all containers to latest versions
docker compose -p localai -f docker-compose.yml --profile <your-profile> pull

# View logs for specific service
docker compose -p localai logs -f <service-name>

# Access service shell
docker compose -p localai exec <service-name> sh
```

### Manual Supabase Operations

When working with Supabase directly:

```bash
# Navigate to Supabase directory
cd supabase/docker

# Start only Supabase services
docker compose -p localai up -d

# Access Supabase Studio
# http://localhost:8000 (or configured domain)
```

## Service Access Points

### Local Development URLs
- **n8n**: http://localhost:5678 - Workflow designer and automation platform
- **Open WebUI**: http://localhost:3000 - Chat interface for LLMs and agents  
- **Supabase Studio**: http://localhost:8000 - Database management and backend tools
- **Flowise**: http://localhost:3001 - No-code AI agent builder
- **Langfuse**: http://localhost:3002 - LLM observability dashboard
- **Neo4j Browser**: http://localhost:7474 - Graph database interface
- **SearXNG**: http://localhost:8006 - Private search engine

### Production Domain Configuration
Configure in `.env` file for public deployment with automatic HTTPS:
- `N8N_HOSTNAME=n8n.yourdomain.com`
- `WEBUI_HOSTNAME=openwebui.yourdomain.com`
- `SUPABASE_HOSTNAME=supabase.yourdomain.com`
- Plus additional service hostnames

## Essential Configuration

### Environment Setup

1. **Copy environment template**: `cp .env .env` (already exists but verify values)

2. **Required secrets in `.env`**:
   - `N8N_ENCRYPTION_KEY` and `N8N_USER_MANAGEMENT_JWT_SECRET` (use `openssl rand -hex 32`)
   - `POSTGRES_PASSWORD`, `JWT_SECRET`, `ANON_KEY`, `SERVICE_ROLE_KEY` (Supabase keys)
   - `NEO4J_AUTH=neo4j/your_password`
   - `LANGFUSE_SALT`, `NEXTAUTH_SECRET`, `ENCRYPTION_KEY` (Langfuse secrets)

3. **Service credentials setup in n8n**:
   - Ollama URL: `http://ollama:11434`
   - Postgres Host: `db` (not localhost - Docker service name)
   - Qdrant URL: `http://qdrant:6333`

### Mac Users Special Configuration

If running Ollama locally on Mac (outside Docker):
1. Update n8n environment: `OLLAMA_HOST=host.docker.internal:11434`
2. In n8n credentials, set Ollama base URL to: `http://host.docker.internal:11434/`

## File System Organization

### Shared Data Directory
- **Path in containers**: `/data/shared`
- **Local path**: `./shared/` 
- Use this path in n8n nodes that interact with local filesystem

### Important Volume Mounts
- **n8n workflows**: `./n8n/backup/` → `/backup` (auto-imported on startup)
- **Neo4j data**: `./neo4j/` → various Neo4j directories
- **SearXNG config**: `./searxng/` → `/etc/searxng`

### Workflow Templates
Pre-configured n8n workflows in `./n8n/backup/workflows/`:
- `V1_Local_RAG_AI_Agent.json` - Basic RAG implementation
- `V2_Local_Supabase_RAG_AI_Agent.json` - RAG with Supabase integration  
- `V3_Local_Agentic_RAG_AI_Agent.json` - Advanced agentic RAG patterns

## Development Patterns

### n8n Workflow Development
- Use the workflow designer at http://localhost:5678
- Workflows auto-import from `./n8n/backup/workflows/` on container start
- Test webhooks using the "Test workflow" feature before activating
- Copy production webhook URLs for Open WebUI integration

### Open WebUI Integration
- Add the `n8n_pipe.py` function in Workspace → Functions
- Configure `n8n_url` to point to your n8n webhook endpoint
- Function appears as selectable model in chat interface

### Vector Database Choice
- **Qdrant**: Higher performance, dedicated vector operations
- **Supabase pgvector**: Integrated with your data, relational + vector queries
- Both available simultaneously - choose based on use case

### Graph Database Usage (Neo4j)
- Ideal for knowledge graphs, entity relationships
- Supports advanced RAG patterns like GraphRAG
- Access via Neo4j Browser or programmatically through n8n

## Troubleshooting

### Common Issues
- **Supabase pooler restarting**: Check Postgres password doesn't contain `@` symbol
- **SearXNG container restarting**: Run `chmod 755 searxng` for permissions
- **GPU not detected**: Follow Ollama Docker GPU setup for your platform
- **Port conflicts**: Use `docker ps` to check for conflicting services

### Container Health Checks
All services include health checks. Monitor with:
```bash
docker compose -p localai ps
```

### Log Access
```bash
# All services
docker compose -p localai logs -f

# Specific service
docker compose -p localai logs -f n8n
```

## Security Notes

- Default configuration is for local development
- Use `--environment public` for production deployments
- Caddy handles automatic HTTPS with Let's Encrypt in production
- Never commit actual secrets to version control
- SearXNG includes privacy-focused search without tracking