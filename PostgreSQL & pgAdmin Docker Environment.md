# Complete Guide: Enterprise-Grade PostgreSQL & pgAdmin Docker Environment

---

### Purpose

This guide provides a robust, cross-platform Docker configuration for PostgreSQL and pgAdmin. It is designed to be completely fail-safe across Linux, macOS (Apple Silicon/Intel), and Windows (WSL2). It automates the creation of administrative and read-only users, handles database seeding on the first boot, and includes a comprehensive utility script for local and S3-based backups and restores.

---

### Design Decisions

* **Multi-Architecture Support:** Utilizes `BUILDX_ARCH` to allow native execution on ARM64 (Apple M-Series/AWS Graviton) and AMD64, preventing emulation overhead and crashes.
* **Named Volumes for Persistence:** Bind mounts often cause permission errors (`chmod`/`chown`) across different host operating systems. Named volumes (`postgres_data` and `pgadmin_data`) allow Docker to natively manage internal Linux permissions, ensuring stability during rebuilds and restarts.
* **Automated User Provisioning:** Uses PostgreSQL's native `/docker-entrypoint-initdb.d/` feature to automatically generate a restricted `db_reader` role. It utilizes `ALTER DEFAULT PRIVILEGES` to ensure the reader automatically gains access to future tables created by the admin.
* **Air-gapped Scripting:** The backup/restore script (`db-tool.sh`) uses a lightweight AWS CLI Docker container for S3 operations, removing the need to install Python or the AWS CLI natively on the host machine.

---

### Architecture Diagram

```text
Host Machine (Mac / Win WSL / Linux)
│
├── [ Docker Network: default bridge ]
│   │
│   ├── Container: production_postgres (Port 5432)
│   │   ├── Volume: postgres_data (Native DB Files)
│   │   ├── Mount: ./init-scripts (Auto-executes SQL/SH on first boot)
│   │   └── Mount: ./backups (Target for dumps/restores)
│   │
│   └── Container: production_pgadmin (Port 5050)
│       ├── Volume: pgadmin_data (Saves DB connections & UI settings)
│       └── Connects internally to 'production_postgres:5432'
│
└── [ Utility Scripts ]
    └── scripts/db-tool.sh (Triggers pg_dump/psql and S3 uploads)

```

---

### Step 1: Create the Directory Structure

Set up your project workspace by creating the necessary folders.

**Command Line Instructions:**

```bash
mkdir -p my-postgres-env/{init-scripts,scripts,backups}
cd my-postgres-env
touch .env docker-compose.yml init-scripts/init-permissions.sql init-scripts/01-import-seed.sh scripts/db-tool.sh
chmod +x init-scripts/01-import-seed.sh scripts/db-tool.sh

```

---

### Step 2: Configure the Environment Variables

Open the `.env` file and paste the following configuration. Modify passwords and S3 buckets as needed.

```env
# Database Core
POSTGRES_VERSION=17-alpine
POSTGRES_DB=mydb
BUILDX_ARCH=amd64

# Admin User
POSTGRES_USER=db_admin
POSTGRES_PASSWORD=SuperSecureAdminPassword123

# Reader User
READER_USER=db_reader
READER_PASSWORD=SecureReaderPassword123

# pgAdmin UI
PGADMIN_EMAIL=admin@mycompany.local
PGADMIN_PASSWORD=SuperSecurePgAdmin123
PGADMIN_PORT=5050

# S3 Backup (Optional)
AWS_ACCESS_KEY_ID=your_aws_access_key
AWS_SECRET_ACCESS_KEY=your_aws_secret_key
AWS_DEFAULT_REGION=us-east-1
S3_BUCKET=my-postgres-backups-bucket

```

---

### Step 3: Initialization and Seed Scripts

PostgreSQL will automatically run files in `init-scripts` in alphabetical order on the very first boot (when the data volume is empty).

**1. Create `init-scripts/init-permissions.sql**`
Ensure your IDE saves this file with **LF** (Linux) line endings, not CRLF.

```sql
DO $$
BEGIN
    IF NOT EXISTS (SELECT FROM pg_catalog.pg_roles WHERE rolname = current_setting('custom.reader_user', true)) THEN
        EXECUTE format('CREATE USER %I WITH PASSWORD %L;', current_setting('custom.reader_user'), current_setting('custom.reader_password'));
    END IF;
END
$$;

GRANT CONNECT ON DATABASE postgres TO :READER_USER;
GRANT USAGE ON SCHEMA public TO :READER_USER;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO :READER_USER;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO :READER_USER;

```

**2. Create `init-scripts/01-import-seed.sh**`

```bash
#!/bin/bash
set -e

SEED_FILE="/docker-entrypoint-initdb.d/seed.sql"

if [ -f "$SEED_FILE" ]; then
    echo "🌱 Seed file found. Importing baseline data into ${POSTGRES_DB}..."
    psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" --dbname "$POSTGRES_DB" -f "$SEED_FILE"
else
    echo "ℹ️ No seed.sql found. Skipping baseline data import."
fi

```

---

### Step 4: The Docker Compose File

Open `docker-compose.yml` and paste the following configuration. This glues the services, networks, and volumes together.

```yaml
services:
  postgres_db:
    image: postgres:${POSTGRES_VERSION:-17-alpine}
    platform: linux/${BUILDX_ARCH:-amd64}
    container_name: production_postgres
    restart: always
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      PGOPTIONS: "-c custom.reader_user=${READER_USER} -c custom.reader_password=${READER_PASSWORD}"
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init-scripts:/docker-entrypoint-initdb.d:ro
      - ./backups:/backups
    command: >
      postgres
      -v READER_USER=${READER_USER}

  pgadmin:
    image: dpage/pgadmin4:latest
    container_name: production_pgadmin
    restart: always
    environment:
      PGADMIN_DEFAULT_EMAIL: ${PGADMIN_EMAIL}
      PGADMIN_DEFAULT_PASSWORD: ${PGADMIN_PASSWORD}
      PGADMIN_CONFIG_ENHANCED_COOKIE_PROTECTION: "False" 
    ports:
      - "${PGADMIN_PORT}:80"
    volumes:
      - pgadmin_data:/var/lib/pgadmin
    depends_on:
      - postgres_db

volumes:
  postgres_data:
    driver: local
  pgadmin_data:
    driver: local

```

---

### Step 5: The Database Automation Tool

Open `scripts/db-tool.sh` and add the management logic.

```bash
#!/bin/bash

if [ -f .env ]; then
    export $(grep -v '^#' .env | xargs)
else
    echo "❌ Error: .env file not found."
    exit 1
fi

CONTAINER_NAME="production_postgres"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
LOCAL_BACKUP_DIR="./backups"
INIT_SCRIPTS_DIR="./init-scripts"

mkdir -p "$LOCAL_BACKUP_DIR"

case "$1" in
    export-seed)
        echo "📦 Exporting database structure and data..."
        docker exec -t $CONTAINER_NAME pg_dump -U "$POSTGRES_USER" -d "$POSTGRES_DB" --clean --if-exists > "$INIT_SCRIPTS_DIR/seed.sql"
        echo "✅ Exported to $INIT_SCRIPTS_DIR/seed.sql"
        ;;
    backup-local)
        FILE="$LOCAL_BACKUP_DIR/backup_$TIMESTAMP.sql.gz"
        docker exec -t $CONTAINER_NAME pg_dump -U "$POSTGRES_USER" -d "$POSTGRES_DB" | gzip > "$FILE"
        echo "✅ Backup saved: $FILE"
        ;;
    backup-s3)
        if [ -z "$S3_BUCKET" ]; then echo "❌ S3_BUCKET missing in .env"; exit 1; fi
        FILE_NAME="backup_$TIMESTAMP.sql.gz"
        docker exec -t $CONTAINER_NAME pg_dump -U "$POSTGRES_USER" -d "$POSTGRES_DB" | gzip > "$LOCAL_BACKUP_DIR/$FILE_NAME"
        docker run --rm -v "$(pwd)/backups:/backups" -e AWS_ACCESS_KEY_ID="$AWS_ACCESS_KEY_ID" -e AWS_SECRET_ACCESS_KEY="$AWS_SECRET_ACCESS_KEY" -e AWS_DEFAULT_REGION="$AWS_DEFAULT_REGION" amazon/aws-cli s3 cp "/backups/$FILE_NAME" "s3://$S3_BUCKET/backups/$FILE_NAME"
        echo "✅ Backup pushed to S3!"
        ;;
    restore-local)
        if [ -z "$2" ]; then echo "❌ Usage: $0 restore-local <file>"; exit 1; fi
        gunzip -c "$LOCAL_BACKUP_DIR/$2" | docker exec -i $CONTAINER_NAME psql -U "$POSTGRES_USER" -d "$POSTGRES_DB"
        echo "✅ Restore complete."
        ;;
    restore-s3)
        if [ -z "$2" ] || [ -z "$S3_BUCKET" ]; then echo "❌ Usage: $0 restore-s3 <file>"; exit 1; fi
        docker run --rm -v "$(pwd)/backups:/backups" -e AWS_ACCESS_KEY_ID="$AWS_ACCESS_KEY_ID" -e AWS_SECRET_ACCESS_KEY="$AWS_SECRET_ACCESS_KEY" -e AWS_DEFAULT_REGION="$AWS_DEFAULT_REGION" amazon/aws-cli s3 cp "s3://$S3_BUCKET/backups/$2" "/backups/$2"
        gunzip -c "$LOCAL_BACKUP_DIR/$2" | docker exec -i $CONTAINER_NAME psql -U "$POSTGRES_USER" -d "$POSTGRES_DB"
        echo "✅ Restore from S3 complete."
        ;;
    *)
        echo "Usage: $0 {export-seed|backup-local|backup-s3|restore-local <file>|restore-s3 <file>}"
        exit 1
        ;;
esac

```

---

### Step 6: Launch and Connect

**1. Start the Environment**
Run the following command in the root directory:

```bash
docker-compose up -d

```

**2. Access pgAdmin**
Open a web browser and navigate to `http://localhost:5050`. Log in using the email and password defined in your `.env` file.

**3. Register the Database Server**
Right-click on **Servers** > **Register** > **Server**. Use the following exact settings in the Connection tab:

| Setting | Value |
| --- | --- 
| **Host name/address** | `production_postgres` |
| **Port** | `5432` |
| **Maintenance database** | `mydb` (or your POSTGRES_DB value) |
| **Username** | `db_admin` (or your POSTGRES_USER value) |
| **Password** | Your Admin Password |

Save the configuration. You now have a resilient, multi-user, backup-ready Postgres environment.Here is the expanded section to add to your documentation, tailored for everyday operations, maintenance, and handling environment resets.

---

### Section 7: Everyday Operations & Maintenance

This section covers common day-to-day scenarios, managing the dual-user environment, and safely updating or resetting your infrastructure.

### 1. Working with the Multi-User Roles

To maintain security, ensure your applications and team members connect using the appropriate credentials:

* **For Applications/DevOps (Admin Access):**
Use the `db_admin` credentials. This role has full privileges to run migrations, create tables, delete schemas, and alter configurations.
* **For Analytics/BI Tools (Data Reader Access):**
Provide external tools (like Metabase, PowerBI, or internal analytical scripts) with the `db_reader` credentials.

#### Testing the Data Reader Restrictions

You can verify that the reader user cannot alter data by attempting an insert statement:

```bash
# Log in as the read-only user
docker exec -it production_postgres psql -U db_reader -d mydb

# Try to create a table (This will fail with a permission denied error)
CREATE TABLE test_table (id SERIAL PRIMARY KEY);

```

---

### 2. Common Command-Line Tasks

#### Accessing the Postgres CLI Natively

If you need to quickly drop into a Postgres shell on the admin account without installing local tools:

```bash
docker exec -it production_postgres psql -U db_admin -d mydb

```

#### Monitoring Real-Time Logs

To inspect connection attempts, query issues, or running processes:

```bash
docker compose logs -f postgres_db

```

#### Checking Database Disk Usage

To see how much space your actual data volume is consuming on the host disk:

```bash
docker system df -v

```

---

### 3. Backup & Seed Workflows

#### Daily/Weekly Maintenance (Local Backups)

To run an ad-hoc local snapshot before executing risky structural changes or updates:

```bash
./scripts/db-tool.sh backup-local

```

*Your file will be safely compressed and stamped inside the `./backups/` directory.*

#### Shipping to Production / Sharing a Baseline

If you have modified your schema locally and want to "freeze" it so anyone else spinning up the project gets your latest database setup:

1. Run the seed export:
```bash
./scripts/db-tool.sh export-seed

```


2. Commit the new `init-scripts/seed.sql` file directly to your Git repository.

---

### 4. Updating and Environment Resets

Because this architecture separates code from state using named Docker volumes, you can clean, upgrade, or destroy containers without accidentally blowing away your production data.

#### Upgrading PostgreSQL or pgAdmin Versions

When a new minor or major release comes out:

1. Update the version string inside your `.env` file (e.g., `POSTGRES_VERSION=17.2-alpine`).
2. Pull the new image and recreate the containers:
```bash
docker compose pull
docker compose up -d

```


*Your database files will cleanly transition to the new container variant via the immutable named volume.*

#### Performing a "Hard Reset" (Wiping Data to Start Fresh)

If your local development database gets corrupted or filled with bad mock data and you want to trigger the initialization scripts completely from scratch:

```bash
# Stop containers and violently purge the underlying named data volumes
docker compose down -v

# Boot everything back up cleanly (this triggers init-permissions.sql and seed.sql automatically)
docker compose up -d

```
