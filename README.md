# Open Web UI, Postgres, Qdrant Docker Compose

[![Docs](https://img.shields.io/badge/Docs-Reference-blue.svg)](https://github.com/open-webui/open-webui)
[![Docker](https://img.shields.io/badge/Docker-Ready-2496ED?logo=docker&logoColor=white)](https://hub.docker.com/r/openwebui/open-webui)
[![OpenWebUI](https://img.shields.io/badge/OpenWebUI-Latest-green.svg)](https://openwebui.com/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-15-336791?logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![Backups](https://img.shields.io/badge/Backups-Automated-yellow.svg)](https://www.postgresql.org/docs/current/backup.html)

![alt text](opq.jpg)

A short guide to setting up the "OPQ Stack"*

*( Note: to the best of my knowledge, nobody actually calls it this..)*

## Date

Tech moves fast and AI at lightning pace. Given the high probability that these docs will become rapidly outdated and soon after obsolete, they were drafted on March 12, 2025. Accuracy at any point after that date cannot be vouched for...

# Deploying Open Web UI With Postgres And Qdrant 

Note: I added Redis to my stack also. If you don't want that, remove from the code samples.

## Standard Docker Compose

For the standard (official) OpenWebUI Docker Compose, see [here](https://github.com/open-webui/open-webui/blob/main/docker-compose.yaml) and their documentation.

## Modification 1: SQLite to Postgres

Refer to the project's (long!) list of environment variables [here](https://docs.openwebui.com/getting-started/env-configuration/).

Open Web UI is provided as a single container which - while convenient - makes it a little harder to understand what's "under the hood." Database-wise, at the time of writing, the answer is SQLite. 

If you prefer Postgres, then you'll need to do a few things:

1) Set up a PostgreSQL instance

You can do this as part of the stack *or* you can use a remote PostgreSQL instance. Some options include:

- Using the containerized PostgreSQL as shown in this guide
- Using a non-dockerized PostgreSQL installed directly on your host machine
- Using a managed PostgreSQL service (like AWS RDS, Azure Database for PostgreSQL, etc.)

Just remember: data in containers is not persistent until it's made to be persistent. So you'll need to ensure that your PostgreSQL container has a persistent volume in order for your data to survive reboots.

2) Set the `DATABASE_URL` environment variable

```yaml
`version: '3.8'
services:
  openwebui:
    image: ghcr.io/open-webui/open-webui:latest
    container_name: openwebui
    hostname: openwebui
    restart: unless-stopped
    depends_on:
      - postgres
    environment:
      - DATABASE_URL=postgresql://postgres:some_random_password@postgres:5432/openwebui
      - PORT=${OPENWEBUI_PORT:-8080}
      - REDIS_URL=redis://default:${REDIS_PASSWORD:-some_random_redis_password}@redis:6379 # Ensure redis is available in your infra (docker-compose, k8s, etc)
      - ENABLE_WEBSOCKET_SUPPORT=true
      - WEBSOCKET_MANAGER=redis
      - WEBSOCKET_REDIS_URL=redis://default:${REDIS_PASSWORD:-some_random_redis_password}@redis:6379
      - DATABASE_POOL_SIZE=20
      - DATABASE_POOL_MAX_OVERFLOW=10
      - DATABASE_POOL_TIMEOUT=30
      - DATABASE_POOL_RECYCLE=1800
      - DEFAULT_USER_EMAIL=user@example.com
      - DEFAULT_USER_PASSWORD=some_random_admin_password
      - DEFAULT_USER_FIRST_NAME=User
      - DEFAULT_USER_LAST_NAME=Name
      - DEFAULT_USER_ROLE=admin
    volumes:
      - openwebui_data:/app/backend/data
    networks:
      - my_network
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  postgres:
    image: postgres:15-alpine
    container_name: postgres
    hostname: postgres
    restart: unless-stopped
    command: postgres -c shared_buffers=256MB -c work_mem=16MB -c maintenance_work_mem=128MB -c effective_cache_size=
```    

## Migrating from SQLite to PostgreSQL

**Important Note**: Simply changing the `DATABASE_URL` environment variable from SQLite to PostgreSQL will not automatically migrate your existing data!

You'll need to perform a migration to transfer your data from SQLite to PostgreSQL.

For migrating your existing OpenWebUI data from SQLite to PostgreSQL, you can use this helpful migration tool:

[open-webui-postgres-migration](https://github.com/taylorwilsdon/open-webui-postgres-migration)

This Python script automates the process of:
1. Extracting data from your SQLite database
2. Transforming it to be compatible with PostgreSQL
3. Loading it into your new PostgreSQL database

The migration process preserves your:
- User accounts and settings
- Conversation history
- Model configurations
- RAG documents and collections

Follow the instructions in the repository to perform the migration before switching your OpenWebUI instance to use PostgreSQL. It's highly recommended to do all database operations when the data is "cold" (ie, the containers reading and writing to them are down; leave some buffer time for residual read/write operations to wind up).

## Modification 2: ChromaDB To Qdrant

See the RAG section of the docs for these variables.

Most importantly, you'll want to define a value for `VECTOR_DB` which - if not provided - will mean that the instance uses its default ChromaDB (that comes along for the ride!).

Currently supported alternatives (again, at the time of writing) are:

-  Milvus  
-  Qdrant  
-  OpenSearch
-  PG Vector 

## Modification 3: Redis for WebSocket Support

Redis is used for WebSocket support in OpenWebUI, which enables real-time communication. The configuration includes setting up a Redis instance and configuring OpenWebUI to use it for WebSocket management.

This is particularly important for features that require real-time updates, such as streaming responses from AI models.

## Docker Compose Snippet

```yaml
version: '3.8'
services:
  openwebui:
    image: ghcr.io/open-webui/open-webui:latest
    container_name: openwebui
    hostname: openwebui
    restart: unless-stopped
    depends_on:
      - postgres
      - qdrant
    environment:
      - DATABASE_URL=postgresql://postgres:${POSTGRES_PASSWORD:-some_random_postgres_password}@postgres:5432/openwebui
      - QDRANT_URI=http://qdrant:${QDRANT_PORT:-6333}
      - VECTOR_DB=${VECTOR_DB:-qdrant}
      - QDRANT_API_KEY=${QDRANT_API_KEY:-some_random_qdrant_api_key}
      - PORT=${OPENWEBUI_PORT:-8080}
      - REDIS_URL=redis://default:${REDIS_PASSWORD:-some_random_redis_password}@redis:6379 
      - ENABLE_WEBSOCKET_SUPPORT=true
      - WEBSOCKET_MANAGER=redis
      - WEBSOCKET_REDIS_URL=redis://default:${REDIS_PASSWORD:-some_random_redis_password}@redis:6379
      - DATABASE_POOL_SIZE=20
      - DATABASE_POOL_MAX_OVERFLOW=10
      - DATABASE_POOL_TIMEOUT=30
      - DATABASE_POOL_RECYCLE=1800
      - DEFAULT_USER_EMAIL=user@example.com
      - DEFAULT_USER_PASSWORD=some_random_admin_password
      - DEFAULT_USER_FIRST_NAME=User
      - DEFAULT_USER_LAST_NAME=Name
      - DEFAULT_USER_ROLE=admin
    volumes:
      - openwebui_data:/app/backend/data
    networks:
      - my_network
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
  postgres:
    image: postgres:15-alpine
    container_name: postgres
    hostname: postgres
    restart: unless-stopped
    command: postgres -c shared_buffers=256MB -c work_mem=16MB -c maintenance_work_mem=128MB -c effective_cache_size=512MB -c max_connections=100
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=some_random_postgres_password
      - POSTGRES_DB=postgres
      - POSTGRES_MULTIPLE_DATABASES=langflow,linkwarden,n8n
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./scripts/postgres-init:/docker-entrypoint-initdb.d
    networks:
      - my_network
  qdrant:
    image: qdrant/qdrant:latest
    container_name: qdrant
    hostname: qdrant
    environment:
      - QDRANT__SERVICE__API_KEY=${QDRANT_API_KEY:-some_random_qdrant_api_key}
      - QDRANT__SERVICE__ENABLE_API_KEY_AUTHORIZATION=true
    restart: unless-stopped
    volumes:
      - qdrant_data:/qdrant/storage
    networks:
      - my_network
 redis:
    image: redis:alpine
    container_name: redis
    hostname: redis
    restart: unless-stopped
    command: redis-server --requirepass ${REDIS_PASSWORD:-some_random_redis_password}
    volumes:
      - redis_data:/data
    networks:
      - my_network

networks:
  my_network:
    external: true
    name: my_network
volumes:
  openwebui_data:
  postgres_data:
  qdrant_data:
  redis_data:
```

## Multiple Databases in PostgreSQL

The `POSTGRES_MULTIPLE_DATABASES` environment variable is used to create multiple databases in the PostgreSQL container during initialization. This is useful if you want to use the same PostgreSQL instance for multiple applications.

 This requires a custom initialization script in the `./scripts/postgres-init` directory. A sample script is provided in this repository:

```bash
# scripts/postgres-init/create-multiple-databases.sh
#!/bin/bash

set -e
set -u

function create_user_and_database() {
    local database=$1
    echo "  Creating user and database '$database'"
    psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" <<-EOSQL
        CREATE USER $database;
        CREATE DATABASE $database;
        GRANT ALL PRIVILEGES ON DATABASE $database TO $database;
EOSQL
}

if [ -n "$POSTGRES_MULTIPLE_DATABASES" ]; then
    echo "Multiple database creation requested: $POSTGRES_MULTIPLE_DATABASES"
    for db in $(echo $POSTGRES_MULTIPLE_DATABASES | tr ',' ' '); do
        create_user_and_database $db
    done
    echo "Multiple databases created"
fi

# Create openwebui database if it doesn't exist in the list
if [[ ! "$POSTGRES_MULTIPLE_DATABASES" =~ "openwebui" ]]; then
    create_user_and_database "openwebui"
    echo "OpenWebUI database created"
fi
```

Make sure to make this script executable:

```bash
chmod +x scripts/postgres-init/create-multiple-databases.sh
```

## Network Configuration

The `my_network` network is defined as external, which means it should be created before running this docker-compose file. This allows multiple docker-compose stacks to communicate with each other on the same network.

You can create the network with the following command:
```bash
docker network create my_network
```
## .Env

Then, make sure that you have the corresponding .env.

```txt

# PostgreSQL Configuration
POSTGRES_USER=postgres
POSTGRES_PASSWORD=some_random_postgres_password
POSTGRES_DB=postgres

# Qdrant Configuration
QDRANT_API_KEY=some_random_qdrant_api_key
QDRANT_PORT=6333

# OpenWebUI Configuration
OPENWEBUI_PORT=8080  # You can change this if you want to expose OpenWebUI on a different port
REDIS_PASSWORD=some_random_redis_password
```

## Running the Stack

To run the stack, follow these steps:

1. Create the external network (if it doesn't exist already):
   ```bash
   docker network create my_network
   ```

2. Copy the `.env.example` file to `.env` and customize the values as needed.

3. Start the stack:
   ```bash
   docker-compose up -d
   ```

4. Access OpenWebUI at `http://localhost:8080` (or the port you specified in the `.env` file).

## PostgreSQL Backup Advantages

One of the major advantages of switching from SQLite to PostgreSQL is the robust backup ecosystem. PostgreSQL offers several significant benefits for data protection.
 
### Automated Backup Script (Illustrative)

Here's a simple script you can use to automate daily backups of your PostgreSQL database:

```bash
#!/bin/bash

# Configuration
BACKUP_DIR="/path/to/backups"
DB_NAME="openwebui"
TIMESTAMP=$(date +%Y-%m-%d_%H-%M-%S)
BACKUP_FILE="$BACKUP_DIR/${DB_NAME}_backup_$TIMESTAMP.sql"

# Create backup directory if it doesn't exist
mkdir -p $BACKUP_DIR

# For dockerized PostgreSQL
docker exec -t postgres pg_dump -U postgres -d $DB_NAME > $BACKUP_FILE

# Compress the backup
gzip $BACKUP_FILE

# Keep only the last 7 backups
ls -t $BACKUP_DIR/${DB_NAME}_backup_*.sql.gz | tail -n +8 | xargs -r rm

echo "Backup completed: $BACKUP_FILE.gz"
```

For a non-dockerized PostgreSQL instance, simply replace the backup command with:

```bash
# For non-dockerized PostgreSQL
PGPASSWORD=your_password pg_dump -h localhost -U postgres -d $DB_NAME > $BACKUP_FILE
```

To run this script daily, save it as `backup_postgres.sh`, make it executable with `chmod +x backup_postgres.sh`, and set up a cron job:

```bash
# Add to crontab (run 'crontab -e')
0 2 * * * /path/to/backup_postgres.sh
```

## AI Prompt

YAML is a finnicky language and there's nothing more annoying than failing to deploy a cool tech stack because you messed up one single line of indentation. 

Fortunately, LLMs are pretty good at fixing and drafting YAML. Ask it for help!

You can try something like this:

`Generate a docker compose that will bring up a stack that includes: OpenWebUI, Postgres, Qdrant`

