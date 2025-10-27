# Agent Instructions for Charon Repository

This repository provides Docker Compose override configurations for extending Snapchain Farcaster Hub nodes with API and data infrastructure services. It is designed to work alongside a base Snapchain installation.

## Repository Purpose

**Charon** extends Snapchain nodes with:
- PostgreSQL database for persistent Farcaster data storage
- Redis for caching and job queuing
- Waypoint API service for populating postresql from redis
- Backfill services for historical data synchronization
- PostGraphile for instant GraphQL APIs

## Repository Structure

```
charon/
├── docker-compose.override.yml   # Main override file extending base snapchain
├── .env                          # Environment configuration (not in git)
├── README.md                     # User-facing documentation
└── AGENTS.md                     # This file - AI agent instructions
```

## Key Conventions

### Docker Compose Architecture

**Two-layer composition pattern:**

1. **Base layer** (`<snapchain-base-dir>/docker-compose.yml`):
   - Managed by `snapchain.sh` installer script
   - Contains: snapchain node, statsd, grafana
   - Network: `snapchain` (bridge)
   - Auto-updated by upstream scripts

2. **Override layer** (`./docker-compose.override.yml`):
   - Defined in this repository
   - Contains: postgres, redis, waypoint, backfill workers
   - Joins `snapchain` network to communicate with base services
   - Manually maintained and versioned

### Service Naming Conventions

- `postgres`: PostgreSQL database with pgvector extension
- `redis`: Redis cache/queue
- `waypoint`: Main API service connecting hub → database
- `backfill-queue`: Job queue populator (profile: `backfill`)
- `backfill-worker`: Job processor (profile: `backfill`)
- `postgraphile`: GraphQL API generator

### Environment Variable Patterns

Variables use double-underscore notation for nested configuration:

```bash
# Pattern: SERVICE_CATEGORY__SETTING
WAYPOINT_DATABASE__URL            # waypoint.database.url
WAYPOINT_HUB__RETRY_MAX_ATTEMPTS  # waypoint.hub.retry_max_attempts
WAYPOINT_MCP__ENABLED             # waypoint.mcp.enabled
```

### Network Architecture

All services join the `snapchain` network (defined in base layer) to enable:
- Waypoint ↔ Snapchain node communication (port 3383)
- Database and Redis internal service discovery
- Isolated from external networks by default

## When Making Changes

### 1. Modifying docker-compose.override.yml

**Always validate syntax before committing:**

```bash
cd <snapchain-base-dir>
docker compose -f docker-compose.yml -f <path-to-charon>/docker-compose.override.yml config
```

**Preserve these critical patterns:**
- All services must join `networks: - snapchain`
- Database services must have `healthcheck` sections
- Dependent services use `depends_on: condition: service_healthy`
- Use environment variable substitution with defaults: `${VAR:-default}`
- Ports use variable expansion: `"${PORT:-8080}:${PORT:-8080}"`

**Testing changes:**

```bash
# 1. Stop existing services
docker compose down

# 2. Start with override
docker compose -f docker-compose.yml -f <path-to-charon>/docker-compose.override.yml up -d

# 3. Verify services are healthy
docker compose ps
docker compose logs -f waypoint
```

### 2. Adding New Services

When adding services (e.g., PostGraphile):

1. **Define service in docker-compose.override.yml**:
   ```yaml
   postgraphile:
     image: graphile/postgraphile:latest
     restart: unless-stopped
     ports:
       - "${POSTGRAPHILE_PORT:-5000}:5000"
     environment:
       # Use connection string format with variable substitution
     depends_on:
       postgres:
         condition: service_healthy
     networks:
       - snapchain
   ```

2. **Add environment variables to README.md** (Configuration section)

3. **Document the service** (Services section with ports, purpose, features)

4. **Update API Endpoints section** if the service exposes HTTP/GraphQL endpoints

5. **Add troubleshooting guidance** for common issues

### 4. Environment Configuration

**Never commit `.env` files** - they contain secrets and instance-specific config.

**When adding new environment variables:**
1. Add to `docker-compose.override.yml` with sensible defaults
2. Document in README.md "Configure environment variables" section
3. Use the `${VAR:-default}` pattern for all optional settings
4. Use `${VAR}` (no default) only for required secrets

**Example:**
```yaml
environment:
  # Required secret (no default)
  POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}

  # Optional with default
  POSTGRES_USER: ${POSTGRES_USER:-postgres}

  # Feature flag with default
  WAYPOINT_MCP__ENABLED: ${WAYPOINT_MCP__ENABLED:-true}
```

## Common Tasks

### Debugging Service Issues

**Service won't start:**

1. Check logs: `docker compose logs <service>`
2. Verify dependencies: `docker compose ps` (ensure postgres/redis healthy)
3. Test network: `docker compose exec waypoint ping postgres`
4. Validate env vars: `docker compose config | grep -A 20 'service_name'`

**Database connection errors:**

1. Check health: `docker compose exec postgres pg_isready`
2. Verify credentials: Ensure `.env` matches connection strings
3. Test connection: `docker compose exec postgres psql -U postgres -d waypoint -c '\dt'`
4. Review pool settings: Reduce `MAX_CONNECTIONS` if hitting limits

**Hub connection issues:**

1. Verify base snapchain running: `docker compose ps snapchain`
2. Test hub connectivity: `docker compose exec waypoint curl -v snapchain:3383`
3. Check shard configuration: Review `WAYPOINT_HUB__SHARD_INDICES` setting
4. Inspect waypoint logs: `docker compose logs waypoint | grep -i hub`

### Scaling Backfill Workers

To increase historical data sync speed:

```bash
# Start multiple workers
docker compose --profile backfill up -d --scale backfill-worker=5

# Monitor queue depth
docker compose exec redis redis-cli LLEN backfill_queue

# Watch worker progress
docker compose logs -f backfill-worker | grep -i "processed"
```

### Updating Services

**Important:** This repository does NOT update the base snapchain installation. Only update services defined in `docker-compose.override.yml`.

```bash
# Update waypoint and related images
docker compose pull waypoint backfill-queue backfill-worker
docker compose up -d waypoint backfill-queue backfill-worker

# Update database (requires downtime)
docker compose stop postgres
docker compose pull postgres
docker compose up -d postgres
```

## Service-Specific Notes

### PostgreSQL (pgvector/pgvector:pg17)

- **Extension support**: Includes pgvector for embeddings (AI/ML use cases)
- **Initialization**: Runs `./migrations/*.sql` on first startup only
- **Persistence**: Data stored in `postgres-data` volume
- **Access**: Internal only (5432 exposed for local debugging)
- **Health check**: `pg_isready` every 10s, 5 retries, 5s start period

**Common operations:**

```bash
# Connect to database
docker compose exec postgres psql -U postgres -d waypoint

# Backup database
docker compose exec postgres pg_dump -U postgres waypoint > backup.sql

# Restore database
cat backup.sql | docker compose exec -T postgres psql -U postgres -d waypoint

# Monitor connections
docker compose exec postgres psql -U postgres -d waypoint -c \
  "SELECT count(*) FROM pg_stat_activity WHERE datname='waypoint';"
```

### Redis (redis:7)

- **Persistence**: AOF (Append-Only File) enabled
- **Purpose**: Job queue for backfill, future caching layer
- **Data location**: `redis-data` volume
- **Config**: Default settings (no redis.conf override)

**Queue management:**

```bash
# Check queue length
docker compose exec redis redis-cli LLEN backfill_queue

# Peek at queue items
docker compose exec redis redis-cli LRANGE backfill_queue 0 10

# Clear queue (if needed)
docker compose exec redis redis-cli DEL backfill_queue

# Monitor operations
docker compose exec redis redis-cli MONITOR
```

### Waypoint (officialunofficial/waypoint)

- **Purpose**: Bridge between Snapchain hub and PostgreSQL
- **Features**: REST API, GraphQL, MCP (Model Context Protocol)
- **Configuration**: Environment variable driven (see docker-compose.override.yml)
- **Logging**: JSON format by default (`WAYPOINT_LOG_FORMAT=json`)

**Key configuration areas:**

- **Database**: Connection pooling, timeout, message storage
- **Hub**: Shard subscription, retry logic, authentication
- **API**: Host binding, ports
- **MCP**: AI agent integration protocol (port 8000)

**Monitoring:**

```bash
# Watch real-time logs
docker compose logs -f waypoint

# Check API health
curl http://localhost:8080/health

# Monitor database connections from waypoint
docker compose exec postgres psql -U postgres -d waypoint -c \
  "SELECT * FROM pg_stat_activity WHERE application_name LIKE '%waypoint%';"
```

### Backfill Services

Two complementary services under `backfill` profile:

1. **backfill-queue**: Reads FID range, queues jobs to Redis
2. **backfill-worker**: Processes jobs, fetches historical data from hub

**Workflow:**
```
backfill-queue → Redis queue → backfill-worker(s) → PostgreSQL
```

**Scaling pattern:**
- Run ONE queue instance (avoids duplicate work)
- Scale workers horizontally: `--scale backfill-worker=N`
- Tune `BACKFILL_CONCURRENCY` for per-worker parallelism

**Start backfill:**

```bash
# Start both queue and workers
docker compose --profile backfill up -d

# Scale workers for faster processing
docker compose --profile backfill up -d --scale backfill-worker=3

# Stop backfill (preserves queue state)
docker compose stop backfill-queue backfill-worker
```

## Relationship to Base Snapchain

**Base snapchain installation** (`<snapchain-base-dir>/`):

- Managed by `snapchain.sh` script (auto-updating)
- Contains official Farcaster Snapchain node
- Defines `snapchain` network used by override services
- Should NOT be modified by this repository

**Override integration points:**

- **Network**: Join existing `snapchain` network (don't redefine)
- **Hub access**: Waypoint connects to `snapchain:3383` for gRPC
- **Monitoring**: Can leverage existing statsd/grafana (not yet configured)
- **Updates**: Base layer updates independently; test compatibility

**Deployment pattern:**

```bash
# Option 1: Explicit file specification
cd <snapchain-base-dir>
docker compose -f docker-compose.yml -f <path-to-charon>/docker-compose.override.yml up -d

# Option 2: Symlink override into base directory (compose auto-discovers)
ln -s <path-to-charon>/docker-compose.override.yml <snapchain-base-dir>/
cd <snapchain-base-dir>
docker compose up -d  # Automatically merges override
```

## Best Practices

### For AI Agents Working with This Repository

1. **Always validate compose syntax**: Run `docker compose config` before committing
2. **Preserve network isolation**: All services must join `snapchain` network
3. **Use health checks**: Dependencies should wait for `condition: service_healthy`
4. **Environment variable defaults**: Use `${VAR:-default}` pattern consistently
5. **Document all changes**: Update README.md when adding services or variables
6. **Test incrementally**: Start new services individually before full deployment
7. **Respect base layer**: Never modify the base Snapchain installation directory directly
8. **Migration safety**: Never edit existing migration files, only append new ones

### Security Considerations

- **Secrets**: Never commit `.env` files or hardcode passwords
- **Port exposure**: Only expose ports necessary for external access
- **Network isolation**: Keep internal services (postgres, redis) off exposed ports in production
- **TLS**: Consider adding TLS termination for production deployments
- **Resource limits**: Add CPU/memory limits to prevent resource exhaustion

### Performance Tuning

**Database:**
- `WAYPOINT_DATABASE__MAX_CONNECTIONS`: Match to concurrent load (default: 60)
- `WAYPOINT_DATABASE__TIMEOUT_SECONDS`: Balance between patience and failure detection

**Redis:**
- `WAYPOINT_REDIS__POOL_SIZE`: Increase for high job throughput (default: 100)
- `WAYPOINT_REDIS__IDLE_TIMEOUT_SECS`: Tune based on usage patterns

**Backfill:**
- `BACKFILL_CONCURRENCY`: Per-worker parallelism (default: 40)
- `BACKFILL_BATCH_SIZE`: Queue batch size (default: 50)
- Worker count: Scale horizontally with `--scale backfill-worker=N`

**Hub connection:**
- `WAYPOINT_HUB__RETRY_MAX_ATTEMPTS`: Increase for unreliable networks
- `WAYPOINT_HUB__CONN_TIMEOUT_MS`: Adjust for network latency

## Future Additions

Services mentioned in documentation but not yet implemented:

### PostGraphile

- **Purpose**: Auto-generate GraphQL API from PostgreSQL schema
- **Port**: 5000 
- **Features**: GraphiQL interface, real-time subscriptions, schema introspection
- **Configuration**: See README.md "Development > Adding PostGraphile"

### Integration with Base Monitoring

- **Statsd**: Waypoint already configured, but could integrate with base statsd
- **Grafana**: Could add Charon-specific dashboards to base Grafana
- **Metrics**: Expose database, Redis, and Waypoint metrics

## Troubleshooting Checklist

When debugging issues, work through this checklist:

- [ ] Base snapchain services running: `docker compose ps snapchain`
- [ ] Override syntax valid: `docker compose config`
- [ ] Networks exist: `docker network ls | grep snapchain`
- [ ] Health checks passing: `docker compose ps` (look for "healthy" status)
- [ ] Logs show errors: `docker compose logs <service>`
- [ ] Environment variables loaded: `docker compose config | grep -A 5 'environment'`
- [ ] Volumes persisted: `docker volume ls | grep postgres-data`
- [ ] Connectivity: `docker compose exec waypoint ping postgres`
- [ ] Port conflicts: `sudo netstat -tlnp | grep -E '(5432|6379|8080)'`

## Resources

- **Snapchain**: https://github.com/farcasterxyz/snapchain
- **Waypoint**: https://github.com/officialunofficial/waypoint
- **Docker Compose**: https://docs.docker.com/compose/
- **PostgreSQL**: https://www.postgresql.org/docs/
- **Redis**: https://redis.io/documentation
- **PostGraphile**: https://www.graphile.org/postgraphile/

## Contributing Workflow

When working with this repository:

1. **Read README.md first** - understand user-facing documentation
2. **Review existing override file** - understand current service definitions
3. **Test changes locally** - use `docker compose config` and staging deployment
4. **Update documentation** - keep README.md and this file in sync
5. **Commit atomically** - one logical change per commit
6. **Document breaking changes** - note any backwards-incompatible modifications

Remember: This repository provides infrastructure extensions. Changes should enhance functionality while maintaining compatibility with the base Snapchain installation.
