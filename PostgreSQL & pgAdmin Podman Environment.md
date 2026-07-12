# Complete Guide: Enterprise-Grade PostgreSQL & pgAdmin using Podman

---

### Purpose
This guide adapts our fail-safe PostgreSQL and pgAdmin architecture to run entirely on **Podman**. By migrating to Podman, we leverage a daemonless, rootless container engine for enhanced security, while maintaining full compatibility across Linux, macOS (Apple Silicon/Intel), and Windows (WSL2 via Podman Desktop/Machine). It retains automated user provisioning, baseline seeding, and the universal backup/restore utility.

---

### Design Decisions

*   **Rootless Execution by Default:** Podman runs containers as a non-root user, significantly reducing security risks. We rely on unprivileged ports (`5432` and `5050`) to avoid root-binding restrictions.
*   **SELinux Contexts (`:z` flags):** Linux distributions using SELinux (like Fedora, RHEL, CentOS) strictly block containers from accessing host directories by default. We append the `:z` flag to our bind mounts to automatically apply the correct SELinux labels, preventing silent permission failures.
*   **Named Volumes for Persistence:** Named volumes seamlessly handle the User Namespace (UserNS) UID/GID mapping that rootless Podman requires, completely sidestepping `chown` headaches on host file systems.
*   **Podman Compose Compatibility:** Podman natively supports Docker Compose specifications. We use standard `compose.yaml` syntax, allowing execution via `podman compose` or `podman-compose`.

---

### Architecture Diagram

```text
Host Machine (Mac/Win via podman machine, Linux native)
│
├── [ Podman Network: rootless internal bridge ]
│   │
│   ├── Container: production_postgres (Port 5432)
│   │   ├── Volume: postgres_data (Native DB Files mapped to UserNS)
│   │   ├── Mount: ./init-scripts (SELinux :z,ro labeled)
│   │   └── Mount: ./backups (SELinux :z labeled)
│   │
│   └── Container: production_pgadmin (Port 5050)
│       ├── Volume: pgadmin_data (Saves DB connections & UI settings)
│       └── Connects internally to 'production_postgres:5432'
│
└── [ Utility Scripts ]
    └── scripts/db-tool.sh (Triggers podman exec/run and S3 uploads)

```

---

### Step 1: Create the Directory Structure

Set up your project workspace.

```bash
mkdir -p my-postgres-env/{init-scripts,scripts,backups}
cd my-postgres-env
touch .env compose.yaml init-scripts/init-permissions.sql init-scripts/01-import-seed.sh scripts/db-tool.sh
chmod +x init-scripts/01-import-seed.sh scripts/db-tool.sh

```

---

### Step 2: Configure the Environment Variables

Open the `.env` file and define your settings.

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

### Step 3: Initialization Scripts

**1. Create `init-scripts/init-permissions.sql**`
Ensure your IDE saves this file with **LF** (Linux) line endings.

```sql
DO $$ BEGIN     IF NOT EXISTS (SELECT FROM pg_catalog.pg_roles WHERE rolname = current_setting('custom.reader_user', true)) THEN         EXECUTE format('CREATE USER \%I WITH PASSWORD \%L;', current_setting('custom.reader_user'), current_setting('custom.reader_password'));     END IF; END $$;

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

### Step 4: The Podman Compose File (`compose.yaml`)

This configuration includes the critical `:z` flags for SELinux environments.

```yaml
services:
  postgres_db:
    image: docker.io/library/postgres:${POSTGRES_VERSION:-17-alpine}
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
      # The ':z' flag tells Podman/SELinux to share this directory with the container
      - ./init-scripts:/docker-entrypoint-initdb.d:z,ro
      - ./backups:/backups:z
    command: >
      postgres
      -v READER_USER=${READER_USER}

  pgadmin:
    image: docker.io/dpage/pgadmin4:latest
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

### Step 5: The Podman Automation Tool (`scripts/db-tool.sh`)

Update the script to use `podman exec` and `podman run`.

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
        podman exec -t $CONTAINER_NAME pg_dump -U "$POSTGRES_USER" -d "$POSTGRES_DB" --clean --if-exists > "$INIT_SCRIPTS_DIR/seed.sql"
        echo "✅ Exported to $INIT_SCRIPTS_DIR/seed.sql"
        ;;
    backup-local)
        FILE="$LOCAL_BACKUP_DIR/backup_$TIMESTAMP.sql.gz"
        podman exec -t $CONTAINER_NAME pg_dump -U "$POSTGRES_USER" -d "$POSTGRES_DB" \vert{} gzip > "$FILE"
        echo "✅ Backup saved: $FILE"
        ;;
    backup-s3)
        if [ -z "$S3_BUCKET" ]; then echo "❌ S3_BUCKET missing in .env"; exit 1; fi
        FILE_NAME="backup_$TIMESTAMP.sql.gz"
        podman exec -t $CONTAINER_NAME pg_dump -U "$POSTGRES_USER" -d "$POSTGRES_DB" | gzip > "$LOCAL_BACKUP_DIR/$FILE_NAME"
        podman run --rm -v "$(pwd)/backups:/backups:z" -e AWS_ACCESS_KEY_ID="$AWS_ACCESS_KEY_ID" -e AWS_SECRET_ACCESS_KEY="$AWS_SECRET_ACCESS_KEY" -e AWS_DEFAULT_REGION="$AWS_DEFAULT_REGION" docker.io/amazon/aws-cli s3 cp "/backups/$FILE_NAME" "s3://$S3_BUCKET/backups/$FILE_NAME"
        echo "✅ Backup pushed to S3!"
        ;;
    restore-local)
        if [ -z "$2" ]; then echo "❌ Usage: $0 restore-local <file>"; exit 1; fi
        gunzip -c "$LOCAL_BACKUP_DIR/$2" \vert{} podman exec -i $CONTAINER_NAME psql -U "$POSTGRES_USER" -d "$POSTGRES_DB"
        echo "✅ Restore complete."
        ;;
    restore-s3)
        if [ -z "$2" ] \vert{}\vert{} [ -z "$S3_BUCKET" ]; then echo "❌ Usage: $0 restore-s3 <file>"; exit 1; fi
        podman run --rm -v "$(pwd)/backups:/backups:z" -e AWS_ACCESS_KEY_ID="$AWS_ACCESS_KEY_ID" -e AWS_SECRET_ACCESS_KEY="$AWS_SECRET_ACCESS_KEY" -e AWS_DEFAULT_REGION="$AWS_DEFAULT_REGION" docker.io/amazon/aws-cli s3 cp "s3://$S3_BUCKET/backups/$2" "/backups/$2"
        gunzip -c "$LOCAL_BACKUP_DIR/$2" \vert{} podman exec -i $CONTAINER_NAME psql -U "$POSTGRES_USER" -d "$POSTGRES_DB"
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
Depending on your Podman installation, run:

```bash
podman compose up -d

```

*(Note: If on Mac/Windows, ensure your Podman machine is running first using `podman machine start`).*

**2. Access pgAdmin**
Open a web browser and navigate to `http://localhost:5050`. Log in using your configured email and password.

**3. Register the Database Server**
Right-click on **Servers** > **Register** > **Server**. Enter the following in the Connection tab:

| Setting | Value |
| --- | --- |
| **Host name/address** | `production_postgres` |
| **Port** | `5432` |
| **Maintenance database** | `mydb` (or your POSTGRES_DB value) |
| **Username** | `db_admin` (or your POSTGRES_USER value) |
| **Password** | Your Admin Password |

Save the configuration. You now have a secure, rootless PostgreSQL setup running efficiently on Podman.

---

### Step 7: Everyday Operations & Maintenance

This section covers standard administrative tasks, user role management, and how to safely update or completely reset your Podman infrastructure without leaving orphaned volumes or encountering SELinux snags.

#### 1. Working with the Multi-User Roles

To maintain security, ensure your applications and team members connect using the appropriate credentials:

* **For Applications/DevOps (Admin Access):**
Use the `db_admin` credentials. This role has full privileges to run migrations, create tables, delete schemas, and alter configurations.
* **For Analytics/BI Tools (Data Reader Access):**
Provide external tools (like Metabase, PowerBI, or internal analytical scripts) with the `db_reader` credentials.

**Testing the Data Reader Restrictions**
You can verify that the reader user cannot alter data by dropping into a rootless interactive shell and attempting an insert statement:

```bash
# Log in as the read-only user
podman exec -it production_postgres psql -U db_reader -d mydb

# Try to create a table (This will fail with a permission denied error)
CREATE TABLE test_table (id SERIAL PRIMARY KEY);

```

---

#### 2. Common Command-Line Tasks

**Accessing the Postgres CLI Natively**
If you need to quickly drop into a Postgres shell on the admin account without opening pgAdmin:

```bash
podman exec -it production_postgres psql -U db_admin -d mydb

```

**Monitoring Real-Time Logs**
To inspect connection attempts, query issues, or running processes:

```bash
podman compose logs -f postgres_db

```

*(Alternatively, you can query the container directly: `podman logs -f production_postgres`)*

**Checking Database Disk Usage**
To see how much space your actual data volume is consuming inside your local user namespace:

```bash
podman system df -v

```

---

#### 3. Backup & Seed Workflows

**Daily/Weekly Maintenance (Local Backups)**
To run an ad-hoc local snapshot before executing risky structural changes or application updates:

```bash
./scripts/db-tool.sh backup-local

```

*Your file will be safely compressed and stamped inside the `./backups/` directory. Podman's `:z` label ensures the host can read the file created by the container.*

**Shipping to Production / Sharing a Baseline**
If you have modified your schema locally and want to "freeze" it so anyone else spinning up the project gets your latest database setup:

1. Run the seed export:
```bash
./scripts/db-tool.sh export-seed

```


2. Commit the new `init-scripts/seed.sql` file directly to your Git repository.

---

#### 4. Updating and Environment Resets

Because this architecture separates code from state using Podman named volumes, you can clean, upgrade, or destroy containers without accidentally blowing away your production data.

**Upgrading PostgreSQL or pgAdmin Versions**
When a new minor or major release comes out:

1. Update the version string inside your `.env` file (e.g., `POSTGRES_VERSION=17.2-alpine`).
2. Pull the new image and recreate the containers:
```bash
podman compose pull
podman compose up -d

```


*Your database files will cleanly transition to the new container variant via the immutable named volume.*

**Performing a "Hard Reset" (Wiping Data to Start Fresh)**
If your local development database gets corrupted or filled with bad mock data and you want to trigger the initialization scripts completely from scratch. **Be warned: this deletes the database completely.**

```bash
# Stop containers and violently purge the underlying named data volumes
podman compose down -v

# Boot everything back up cleanly (this triggers init-permissions.sql and seed.sql automatically)
podman compose up -d

```

