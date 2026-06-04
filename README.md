# Proyecto DBT - SQL Server Integration

This project sets up a **dbt (data build tool)** environment with **SQL Server 2022** running in Docker containers. 

## Overview

The architecture consists of:
- **SQL Server 2022** container with ODBC drivers
- **dbt-core** container with dbt-sqlserver adapter
- Network communication between containers for seamless dbt operations

## Project Structure

```
proyecto-dbt/
├── docker-compose.yml          # Docker container orchestration
├── Dockerfile.dbt              # dbt container image build
├── .env                        # Environment variables (SQL password)
├── dbt/                        # dbt project directory
│   ├── dbt_project.yml         # dbt project configuration
│   ├── profiles.yml            # dbt profiles & adapter configuration
│   ├── .env                    # dbt-specific environment variables
│   ├── .user.yml               # dbt user settings
│   ├── models/                 # SQL transformation models (user-created)
│   ├── seeds/                  # CSV seed data (user-created)
│   ├── tests/                  # Data quality tests (user-created)
│   └── target/                 # Generated dbt artifacts (auto-created)
└── README.md                   # Project documentation
```

## Setup Instructions

### Prerequisites

- Docker & Docker Compose installed
- Windows PowerShell or WSL2 terminal
- Git (optional, included in dbt container)

### 1. Clone or Download Project

Place the project files in a workspace directory:
```bash
cd ~/Desktop/proyecto-dbt
```

### 2. Configure Environment

The project includes a `.env` file with the SQL Server password:

**File: `.env`**
```env
SQL_PASSWORD=TuPasswordSuperSeguroYComplejo2026!
```

This password is:
- Used by SQL Server during initialization
- Referenced by dbt profiles for authentication
- Passed to containers via environment variables

### 3. Build and Start Containers

```bash
cd /c/Users/ruben.moreno/Desktop/proyecto-dbt
docker-compose up --build -d
```

**Output:**
```
[+] Building 2.0s
[+] up 2/2
 ✔ Network proyecto-dbt_default Created
 ✔ Container sqlserver_local Started
 ✔ Container dbt_core_local Started
```

This command:
- Builds the dbt Docker image from `Dockerfile.dbt`
- Starts both SQL Server and dbt containers
- Creates a Docker network for inter-container communication
- Mounts the `./dbt` directory to `/usr/app/dbt_project` in the dbt container

### 4. Verify Configuration with dbt debug

```bash
docker-compose exec -T dbt dbt debug
```

**Successful Output:**
```
Running with dbt=1.11.11
dbt version: 1.11.11
python version: 3.11.13
os info: Linux-6.6.114.1-microsoft-standard-WSL2-x86_64-with-glibc2.31

Using profiles dir at /usr/app/dbt_project
Using profiles.yml file at /usr/app/dbt_project/profiles.yml
Using dbt_project.yml file at /usr/app/dbt_project/dbt_project.yml

adapter type: sqlserver
adapter version: 1.9.2

Configuration:
  profiles.yml file [OK found and valid]
  dbt_project.yml file [OK found and valid]

Required dependencies:
  - git [OK found]

Connection:
  server: sqlserver
  port: 1433
  database: master
  schema: dbo
  UID: sa
  authentication: sql
  retries: 3
  login_timeout: 0
  query_timeout: 0
  trace_flag: False
  encrypt: True
  trust_cert: True

Registered adapter: sqlserver=1.9.2
  Connection test: [OK connection ok]

All checks passed!
```

## Docker Configuration Details

### docker-compose.yml

**SQL Server Service:**
- Image: `mcr.microsoft.com/mssql/server:2022-latest`
- Container: `sqlserver_local`
- Port: `1433:1433` (exposed to host)
- Volume: `sqlserver_data` (persistent storage)
- Environment: `ACCEPT_EULA=Y`, `MSSQL_SA_PASSWORD` (from .env)

**dbt Service:**
- Build: `Dockerfile.dbt`
- Container: `dbt_core_local`
- Volume: `./dbt:/usr/app/dbt_project` (mount local dbt directory)
- Environment:
  - `DBT_PROFILES_DIR=/usr/app/dbt_project` (profiles location)
  - `DBT_ENV_SECRET_PASSWORD` (from .env for database auth)
- Depends on: `sqlserver` (waits for SQL Server to be ready)
- Command: `tail -f /dev/null` (keeps container alive)

### Dockerfile.dbt

Multi-stage build that:
1. Starts with Python 3.11 slim image
2. Installs system dependencies: curl, gnupg, git, build-essential
3. Adds Microsoft ODBC drivers (required for SQL Server connections)
4. Installs dbt-core and dbt-sqlserver adapter
5. Sets working directory to `/usr/app/dbt_project`

**Key installations:**
- `dbt-core` - Core dbt framework
- `dbt-sqlserver` - SQL Server adapter
- `msodbcsql18` - Microsoft ODBC Driver 18 for SQL Server
- `git` - Version control & dbt dependency management

### dbt Configuration Files

**dbt_project.yml**
```yaml
name: 'proyecto_dbt'
version: '1.0.0'
config-version: 2
profile: 'default'

models:
  proyecto_dbt:
    materialized: table
```

**profiles.yml**
```yaml
default:
  outputs:
    dev:
      type: sqlserver
      driver: 'ODBC Driver 18 for SQL Server'
      host: sqlserver          # Docker container name
      port: 1433
      user: sa
      password: "{{ env_var('DBT_ENV_SECRET_PASSWORD') }}"
      database: master
      schema: dbo
      encrypt: true
      trust_cert: true
  target: dev
```

**Connection Details:**
- Host: `sqlserver` (Docker container name, resolved via DNS)
- User: `sa` (SQL Server system administrator)
- Password: Injected from environment variable
- Database: `master` (default system database, can be changed)
- Schema: `dbo` (default schema)
- Encryption: Enabled with self-signed certificate trust

## Common dbt Commands

Once `dbt debug` shows all checks passed, you can run:

```bash
# Load seed data (CSV files)
docker-compose exec -T dbt dbt seed

# Run models (create/update tables)
docker-compose exec -T dbt dbt run

# Test data quality
docker-compose exec -T dbt dbt test

# Generate documentation
docker-compose exec -T dbt dbt docs generate

# Compile without running
docker-compose exec -T dbt dbt compile

# Preview SQL without executing
docker-compose exec -T dbt dbt compile --select model_name
```

## Container Management

```bash
# View running containers
docker-compose ps

# View container logs
docker-compose logs sqlserver
docker-compose logs dbt

# Stop containers
docker-compose down

# Remove volumes (data deletion)
docker-compose down -v

# Interactive shell in dbt container
docker-compose exec dbt bash

# Database query (via dbt container)
docker-compose exec -T dbt sqlcmd -S sqlserver -U sa -P <password> -Q "SELECT @@VERSION"
```

## Troubleshooting

### Container not starting
```bash
# Check logs for errors
docker-compose logs -f

# Verify .env file exists with SQL_PASSWORD
cat .env
```

### dbt debug shows connection errors
- Ensure both containers are running: `docker-compose ps`
- Wait 30 seconds after first startup for SQL Server initialization
- Verify `profiles.yml` has correct host (should be `sqlserver`, not localhost)

### Port conflicts
If port 1433 is already in use, modify `docker-compose.yml`:
```yaml
ports:
  - "1434:1433"  # Change host port to 1434
```

### Permission denied errors
Ensure dbt project directory is readable:
```bash
chmod 755 dbt/
```

## Next Steps

After confirming `dbt debug` passes all checks:

1. **Create models** - Add SQL transformation files in `dbt/models/`
2. **Add seeds** - Place CSV data files in `dbt/seeds/`
3. **Define tests** - Create data quality tests in `dbt/schema.yml`
4. **Run transformations** - Execute `dbt run` to build tables
5. **Validate quality** - Run `dbt test` to ensure data integrity

## References

- [dbt Documentation](https://docs.getdbt.com)
- [dbt SQL Server Adapter](https://docs.getdbt.com/reference/warehouse-setups/sqlserver-setup)
- [SQL Server Docker Image](https://hub.docker.com/_/microsoft-mssql-server)
- [Docker Compose Documentation](https://docs.docker.com/compose/)

## Notes

- This setup uses the `dbo` schema (SQL Server default)
- SQL Server is initialized with `master` database
- dbt profiles reference the container name `sqlserver` directly (no port remapping needed within container network)
- All credentials are environment-based for security
- The dbt container includes git for managing dbt packages and dependencies
