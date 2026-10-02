# Claude.md - Project Documentation for AI Assistants

## Project Overview

Docker-based lab environment for **Okta On-Premises Provisioning (OPP) Agent** with JDBC connectivity. Provides containerized setup for testing and developing Okta provisioning integrations with on-premises databases.

**Project:** okta-lab-onprem-jdbc-legacy | **Author:** Fabio Grasso <iam@fabiograsso.net>
**Quick Start:** [QUICKSTART.md](QUICKSTART.md)

## Architecture

Four Docker services with separated OPP Agent and SCIM Server containers:

**1. MariaDB Database (`db`)**
- Image: `mariadb:11`, Port: 3306 (internal)
- Data: `./data/mysql`, SQL init: `./sql/` → `/docker-entrypoint-initdb.d/`
- Health checks enabled

**2. DBGate (`dbgate`)**
- Image: `dbgate/dbgate:latest`, Port: 8090 → 3000
- Access: http://localhost:8090, pre-configured for MariaDB
- Depends on: `db` health

**3. Okta OPP Agent (`okta-opp`)**
- Image: `quay.io/centos/centos:stream9-minimal` (linux/amd64)
- Components: Okta Provisioning Agent, Oracle JDK (bundled)
- Volumes: `./data/okta-opp/{conf,logs,security}`, `./docker/okta-opp/packages` (read-only)
- Depends on: `okta-scim` health

**4. Okta SCIM Server (`okta-scim`)**
- Image: `quay.io/centos/centos:stream9-minimal` (linux/amd64)
- Components: Okta On-Prem SCIM Server, OpenJDK 25, MySQL Connector/J 9.6.0 (auto-downloaded)
- Volumes: `./data/okta-scim/{logs,conf,certs}`, `./docker/okta-scim/packages` (read-only)
- Depends on: `db` health

## Directory Structure

```
.
├── .env                          # Environment variables (gitignored)
├── .env-sample                   # Sample environment configuration
├── docker-compose.yml            # Docker services definition
├── Makefile                      # Build and deployment commands
├── README.md                     # User-facing documentation
├── CLAUDE.md                     # This file (AI assistant documentation)
├── data/                         # Persistent data (gitignored)
│   ├── mysql/                    # MariaDB data files
│   ├── okta-opp/                 # OPP Agent data
│   │   ├── conf/                 # Agent configuration files
│   │   ├── logs/                 # Agent logs
│   │   └── security/             # Keystores and certificates
│   └── okta-scim/                # SCIM Server data
│       ├── conf/                 # SCIM Server configuration
│       ├── logs/                 # SCIM Server logs
│       └── certs/                # SCIM Server certificates and keystores
├── docker/                       # Docker build contexts
│   ├── okta-opp/                 # OPP Agent container
│   │   ├── Dockerfile            # OPP Agent image definition
│   │   ├── entrypoint.sh         # OPP Agent startup script
│   │   └── packages/             # OPP Agent packages (not in git)
│   │       ├── OktaProvisioningAgent*.rpm
│   │       └── *.pem/*.crt       # Optional certificates
│   └── okta-scim/                # SCIM Server container
│       ├── Dockerfile            # SCIM Server image definition
│       ├── entrypoint.sh         # SCIM Server startup script
│       └── packages/             # SCIM Server packages (not in git)
│           ├── OktaOnPremScimServer*.rpm
│           ├── *.jar             # JDBC drivers
│           └── *.pem/*.crt       # Optional certificates
└── sql/                          # Database initialization scripts
    ├── init.sql                  # Schema and test data
    └── stored_proc.sql           # SCIM stored procedures
```

## Required Files (Not in Repository)

**OPP Agent Files** (`./docker/okta-opp/packages/`):
1. **OktaProvisioningAgent-*.rpm** (Required) - [Download](https://help.okta.com/oie/en-us/content/topics/provisioning/opp/opp-install-agent.htm)
2. **CA Certificates** (Optional) - `*.pem` or `*.crt` for custom VPN (e.g., Prisma Access, GlobalProtect)

**SCIM Server Files** (`./docker/okta-scim/packages/`):
1. **OktaOnPremScimServer-*.rpm** (Required) - [Download](https://help.okta.com/oie/en-us/content/topics/provisioning/opp/on-prem-scim-install.htm)
2. **JDBC Drivers** (Optional) - `*.jar` files. MySQL Connector/J 9.6.0 auto-downloaded. Additional drivers for MariaDB, PostgreSQL, Oracle, SQL Server supported. All jars copied to `/opt/OktaOnPremScimServer/userlib/`. See [Generic DB Connector docs](https://help.okta.com/oie/en-us/content/topics/provisioning/opc/connectors/on-prem-connector-generic-db.htm).
3. **CA Certificates** (Optional) - `*.pem` or `*.crt` for custom VPN

## Database Initialization

MariaDB auto-executes SQL scripts from `./sql/` on first startup via `/docker-entrypoint-initdb.d/`.

### Schema and Test Data (`sql/init.sql`)

**USERS Table** (25 fields):
- **Required (5):** USER_ID (PK), USERNAME, FIRSTNAME, LASTNAME, EMAIL
- **Optional (20):** Identity (MIDDLENAME, DISPLAYNAME, NICKNAME), Contact (MOBILEPHONE, STREETADDRESS, CITY, STATE, ZIPCODE, COUNTRYCODE, TIMEZONE), Work (TITLE, ORGANIZATION, DEPARTMENT, EMPLOYEENUMBER, MANAGER, MANAGERID), Dates (HIREDATE, TERMINATIONDATE), Security (PASSWORD_HASH, IS_ACTIVE)

**ENTITLEMENTS Table:**
- ENT_ID (INT PK), ENT_NAME (VARCHAR 100 UNIQUE), ENT_DESCRIPTION (VARCHAR 255)

**USERENTITLEMENTS Table:**
- USERENTITLEMENT_ID (UUID PK), USER_ID (FK), ENT_ID (FK), ASSIGNEDDATE, Unique constraint (USER_ID, ENT_ID)

**Test Data:**
- 15 Star Wars characters with extended profiles (Luke, Leia, Han, Obi-Wan, Yoda, etc.)
- Organizations: Jedi, Resistance, Empire, Droid with manager hierarchies
- 10 entitlements: VPN Access, GitHub Admin, AWS Console, Jira Admin, Confluence Edit, Database Read/Write, Slack Admin, Office 365, Salesforce
- Realistic role-based assignments

**Views (6):** V_USERENTITLEMENTS, V_INACTIVE_USERENTITLEMENTS, V_ACTIVE_USERS, V_INACTIVE_USERS, V_ENTITLEMENT_USAGE, V_USER_HIERARCHY

### Stored Procedures (`sql/stored_proc.sql`)

Based on Generic DB Connector Appendix A, adapted for MySQL/MariaDB:

1. **GET_ACTIVEUSERS()** - Returns all active users (all 25 fields)
2. **GET_ALL_ENTITLEMENTS()** - Returns all entitlements (ENT_ID, ENT_NAME, ENT_DESCRIPTION)
3. **GET_USER_BY_ID(p_user_id)** - Returns specific user (all 25 fields)
4. **GET_USER_ENTITLEMENT(p_user_id)** - Returns user's entitlements with details
5. **CREATE_USER(...)** - Creates user with 24 params (5 mandatory: user_id, username, firstname, lastname, email; 19 optional). Sets IS_ACTIVE=1.
6. **UPDATE_USER(...)** - Updates user with 24 params (5 mandatory, 19 optional)
7. **ACTIVATE_USER(p_user_id)** - Sets IS_ACTIVE=1
8. **DEACTIVATE_USER(p_user_id)** - Sets IS_ACTIVE=0
9. **ADD_ENTITLEMENT_TO_USER(p_user_id, p_ent_id)** - Assigns entitlement
10. **REMOVE_ENTITLEMENT_FROM_USER(p_user_id, p_ent_id)** - Revokes entitlement

**See also:** [Okta_Provisioning_Configuration.md](doc/Okta_Provisioning_Configuration.md), [Okta_SCIM_Server.md](doc/Okta_SCIM_Server.md)

## Configuration

**Environment Variables (.env):**
```bash
MARIADB_PORT=3306
MARIADB_ROOT_PASSWORD=oktademo
MARIADB_USER=oktademo
MARIADB_PASSWORD=oktademo
MARIADB_DATABASE=oktademo
```

**OPP Agent** (`./data/okta-opp/conf/`):
Waits for configuration files before starting:
1. **OktaProvisioningAgent.conf** - Must contain: orgUrl, agentId, keystoreKey, keyPassword, env, subdomain, agentKey
2. **settings.conf** - Auto-configured: `JAVA_OPTS="-Xmx4096m -Dhttps.protocols=TLSv1.2"`
3. **Keystore** - `./data/okta-opp/security/OktaProvisioningKeystore.p12`

**SCIM Server** (`./data/okta-scim/`):
Auto-generates configuration on first startup:
- **config-${CUSTOMER_ID}.properties** - SCIM server config including bearer token
- **customer-id.conf** - Customer ID
- **jvm.conf** - Java memory and GC settings
- **Certificates** - 4096-bit RSA cert/key/p12 keystore (10-year validity, self-signed)
- **Logs** - `/var/log/OktaOnPremScimServer/` → `./data/okta-scim/logs/`

**CRITICAL:** Bearer token in Okta Admin Console MUST include `Bearer ` prefix: `Bearer <token-value>`

## Build and Deployment Commands (Makefile)

```bash
make help            # Display all commands
make check-prereqs   # Run prerequisite checks
make build           # Build images (with prereq checks)
make rebuild         # Force rebuild (no cache)
make start           # Start services (detached)
make start-live      # Start with live logs
make start-logs      # Start detached + follow logs
make stop            # Stop and remove containers
make restart         # Stop then start
make restart-logs    # Restart + follow logs
make logs            # Follow logs (last 500 lines)
make kill            # Kill containers + remove orphans
make configure       # Interactive agent config
```

**Prerequisite Checks** (auto-run before start/build):
Verifies OPP/SCIM RPMs (required), JDBC jars (info only), certs (warning only), .env file, MARIADB_* vars. Exits on critical failures.

**Interactive Configure:** `make configure` runs agent setup script (requires running container) or manually place files in `./data/okta-opp/conf/`.

## Setup Workflow

1. **Prepare packages:**
   ```bash
   mkdir -p docker/okta-{opp,scim}/packages
   # Copy RPMs (required) and certificates (optional) to respective directories
   ```

2. **Configure environment:** `cp .env-sample .env` (edit if needed)

3. **Build:** `make build` (prereq checks run automatically)

4. **Start:** `make start-logs` (follows logs, initializes DB)

5. **Configure OPP Agent:**
   - Wait for "⏳ Waiting for configuration files" message
   - Run `make configure` OR place config files in `./data/okta-opp/conf/`
   - Agent auto-starts when config detected

6. **Retrieve SCIM Credentials:**
   ```bash
   cat ./data/okta-scim/conf/config-*.properties | grep scim.security.bearer.token
   cat ./data/okta-scim/certs/OktaOnPremScimServer-*.crt
   ```
   Need: hostname=`okta-scim`, token=`Bearer <value>`, cert=.crt file

7. **Verify:** `docker compose exec db mariadb -u oktademo -poktademo oktademo -e "SELECT COUNT(*) FROM USERS;"` (expect 15)

8. **Access DBGate:** http://localhost:8090 (root/oktademo)

## Container Behavior

**OPP Agent** (`./docker/okta-opp/entrypoint.sh`):
1. Display Okta logo
2. Create dirs: security, conf, logs
3. Configure Java opts: -Xmx4096m, TLSv1.2
4. Set ownership: provisioningagent:provisioningagent
5. Wait for config files (polls every 10s): OktaProvisioningAgent.conf with required keys, keystore
6. Start agent in background

**SCIM Server** (`./docker/okta-scim/entrypoint.sh`):
1. Display Okta logo
2. Create cert symlinks
3. Generate CUSTOMER_ID (if not exists)
4. Generate 4096-bit RSA cert/key/p12 (10yr validity)
5. Create Spring Boot config (port 1443, TLS, bearer token, logging)
6. Create JVM config
7. Copy JDBC jars to userlib
8. Display credentials
9. Health check: validates `/ws/rest/jdbc_on_prem/scim/v2/Status` every 5 min
10. Start SCIM server (foreground)

All config/certs/logs persist to host, survive restarts.

## Database Access

**DBGate:** http://localhost:8090 (oktademo/oktademo)
**CLI:** `docker-compose exec db mariadb -u oktademo -poktademo oktademo`
**JDBC:** host=db, port=3306, db=oktademo, user/pass=oktademo

## Troubleshooting

**Status & Logs:**
```bash
docker-compose ps
make logs                                    # All services
docker compose logs -f {okta-opp|okta-scim|db}
tail -f ./data/okta-{opp|scim}/logs/*.log
```

**Agent not starting:**
1. Check `./data/okta-opp/conf/OktaProvisioningAgent.conf` exists
2. Verify required keys: orgUrl, agentId, keystoreKey, keyPassword, env, subdomain, agentKey
3. Verify keystore: `./data/okta-opp/security/OktaProvisioningKeystore.p12`
4. Check permissions (provisioningagent user)
5. Check logs: `tail -f ./data/okta-opp/logs/agent.log`

**SCIM Server issues:**
1. Check logs: `./data/okta-scim/logs/`
2. Verify JDBC drivers: `./docker/okta-scim/packages/*.jar`
3. Verify bearer token: `cat ./data/okta-scim/conf/config-*.properties | grep bearer.token`
4. Test endpoint:
   ```bash
   curl -i https://localhost:1443/ws/rest/jdbc_on_prem/scim/v2/ServiceProviderConfig \
     -H "Authorization: Bearer YOUR_TOKEN" --insecure
   ```

**Database issues:**
```bash
docker-compose ps db
docker-compose exec db mariadb-admin ping
# Check .env variables
```

**Build failures:**
```bash
ls -la docker/okta-{opp,scim}/packages/  # Verify packages exist
make rebuild                              # Rebuild without cache
```

**Prereq failures:**
- Missing RPMs: Download from Okta docs (see error message)
- Missing .env: `cp .env-sample .env`
- JDBC drivers: Optional (MySQL auto-downloaded)
- Certificates: Optional (warning only, non-blocking)

**Configure command fails:**
- Start container first: `make start`
- Or manually add config to `./data/okta-opp/conf/`

### Database Query Logging

Debug SQL queries by enabling MariaDB's general query log:

**Enable (temporary):**
```bash
docker compose exec db mariadb -u root -poktademo -e "SET GLOBAL general_log = 'ON';"
docker compose exec db mariadb -u root -poktademo -e "SET GLOBAL general_log_file = '/var/log/mysql/general.log';"
```

**Enable (permanent):** Add to docker-compose.yml db service:
```yaml
command:
  - --general-log=1
  - --general-log-file=/var/log/mysql/general.log
```
Then `make restart`

**View logs:**
```bash
docker compose exec db tail -f /var/log/mysql/general.log
docker compose exec db grep "CALL" /var/log/mysql/general.log
```

**Disable:** `docker compose exec db mariadb -u root -poktademo -e "SET GLOBAL general_log = 'OFF';"`

**Persist logs:** Add volume `./data/mysql-logs:/var/log/mysql` to docker-compose.yml, then `mkdir -p ./data/mysql-logs && make restart`

**Warning:** High I/O impact. Enable only for debugging. Shows all SQL with parameters.

## Development

**Modify containers:**
1. Edit `docker/okta-{opp|scim}/{Dockerfile|entrypoint.sh}`
2. `make rebuild && make restart-logs`

**Add JDBC drivers:**
1. Place `.jar` in `./docker/okta-scim/packages/`
2. `make rebuild`
3. Verify: `docker compose exec okta-scim ls -la /opt/OktaOnPremScimServer/userlib/`

**Certificate management:**
- Auto-updated on start from `./docker/okta-{opp|scim}/packages/*.{pem|crt}` to `/etc/pki/ca-trust/source/anchors/`

**Schema changes:**
1. Edit `sql/init.sql` or `sql/stored_proc.sql`
2. `docker compose down && rm -rf ./data/mysql`
3. `make start-logs`
Note: SQL scripts run only on first init. For existing DBs, apply changes manually.

## Security & Known Issues

**Security:**
- Lab passwords only (not production-ready)
- .env/keystores/certs gitignored
- OPP agent runs as provisioningagent (non-root)

**Known Issues:**
- Platform: linux/amd64 (macOS compat, possible ARM perf impact)
- Ports: 8090 (DBGate), 3306 (MariaDB internal)
- Check conflicts: `lsof -i :8090`

## References & Versions

**Documentation:**
- [OPP Agent Install](https://help.okta.com/oie/en-us/content/topics/provisioning/opp/opp-install-agent.htm)
- [SCIM Server Install](https://help.okta.com/oie/en-us/content/topics/provisioning/opp/on-prem-scim-install.htm)
- [Generic DB Connector](https://help.okta.com/oie/en-us/content/topics/provisioning/opc/connectors/on-prem-connector-generic-db.htm)

**Versions:**
- MariaDB: 11 | DBGate: Latest | CentOS: Stream 9 Minimal
- OpenJDK: 25 Headless | Oracle JDK: (RPM bundled)
- MySQL Connector/J: 9.6.0 (auto-downloaded)
- OPP Agent & SCIM Server: From RPMs (varies)

## Key Configuration Notes

**Bearer Token Format:**
When configuring Generic Database Connector in Okta Admin Console, bearer token MUST include `Bearer ` prefix:
- ✅ Correct: `Bearer da655feabd8ec0c3f89c1fb6e9f0ad39`
- ❌ Incorrect: `da655feabd8ec0c3f89c1fb6e9f0ad39` (fails authentication)

Token from: `./data/okta-scim/conf/config-*.properties` (scim.security.bearer.token)

**SCIM Base URL:**
Use container hostname `okta-scim` (containers on same Docker network)

**SQL Initialization:**
Scripts in `./sql/` auto-run alphabetically on first startup:
1. **init.sql** - Schema with 25-field USERS, test data (15 users, 10 entitlements), 6 views
2. **stored_proc.sql** - 10 procedures for SCIM operations

Reset: `docker compose down && rm -rf ./data/mysql && make start-logs`

## Additional Documentation

- **QUICKSTART.md** - Fast setup (minimal steps, essential commands)
- **README.md** - User guide (comprehensive with diagrams, badges, architecture)
- **doc/Okta_Provisioning_Configuration.md** - Admin Console config (procedures, parameters, screenshots, testing)
- **doc/Okta_SCIM_Server.md** - SCIM internals (Spring Boot 3.5.0, SCIM 2.0 endpoints, auth, config, performance - educational only, not official)
- **CLAUDE.md** (this file) - Technical reference for AI assistants (implementation, architecture, schema, troubleshooting)
