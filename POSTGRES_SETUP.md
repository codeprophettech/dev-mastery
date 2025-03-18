# PostgreSQL Backup Documentation

This document provides a complete guide for setting up, backing up, and restoring a PostgreSQL database running in Docker on Ubuntu Server.

## Table of Contents

1. [Database Setup](#database-setup)
2. [Backup System Overview](#backup-system-overview)
3. [Script Implementation](#script-implementation)
4. [Crontab Configuration](#crontab-configuration)
5. [Backup File Management](#backup-file-management)
6. [Emergency Restoration Procedure](#emergency-restoration-procedure)
7. [Backup Testing Schedule](#backup-testing-schedule)
8. [Contact Information](#contact-information)
9. [Changelog](#changelog)

## Database Setup

### Docker Compose Configuration

Create a `docker-compose.yml` file with the following configuration:

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:latest
    container_name: postgres-container
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: postgres
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./init-scripts:/docker-entrypoint-initdb.d
    networks:
      - postgres-network
    restart: always
    command: >
      postgres -c listen_addresses='*'
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 30s
      timeout: 10s
      retries: 5

volumes:
  postgres-data:

networks:
  postgres-network:
    driver: bridge
```

### Backup User Initialization

Create a directory for initialization scripts:

```bash
mkdir -p init-scripts
```

Create the file `init-scripts/01-create-backup-user.sql`:

```sql
-- File: init-scripts/01-create-backup-user.sql

-- Create a backup user with restricted permissions
CREATE USER backup WITH PASSWORD 'backup_password_here';

-- Grant connection privileges and read-only access
GRANT CONNECT ON DATABASE postgres TO backup;
GRANT USAGE ON SCHEMA public TO backup;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO backup;
GRANT SELECT ON ALL SEQUENCES IN SCHEMA public TO backup;

-- Allow the backup user to read future tables
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO backup;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON SEQUENCES TO backup;

-- Grant replication privileges for WAL archiving if needed
GRANT REPLICATION SLAVE ON DATABASE postgres TO backup;
```

### Starting the Database

Deploy the PostgreSQL container:

```bash
docker-compose up -d
```

## Backup System Overview

Our PostgreSQL backup system features:

- Daily automated backups at 2:00 AM
- 14-day retention policy
- Compressed backup files
- Offsite backup transfer
- Backup monitoring and alerts

## Script Implementation

### 1. Backup Script

Create the file `/usr/local/bin/postgres-backup.sh`:

```bash
#!/bin/bash

# Configuration
BACKUP_DIR="/var/backups/postgres"
DOCKER_CONTAINER="postgres-container"
DB_NAME="postgres"
BACKUP_USER="backup"
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
FILENAME="${DB_NAME}_${TIMESTAMP}.sql.gz"
RETENTION_DAYS=14

# Create backup directory if it doesn't exist
mkdir -p $BACKUP_DIR

# Run PostgreSQL dump inside the container and compress
docker exec -t $DOCKER_CONTAINER pg_dump -U $BACKUP_USER -d $DB_NAME | gzip > "$BACKUP_DIR/$FILENAME"

# Exit if backup failed
if [ $? -ne 0 ]; then
  echo "Database backup failed!"
  exit 1
fi

# Set proper permissions
chmod 600 "$BACKUP_DIR/$FILENAME"

# Delete backups older than RETENTION_DAYS
find $BACKUP_DIR -name "${DB_NAME}_*.sql.gz" -mtime +$RETENTION_DAYS -delete

echo "Backup completed: $FILENAME"
```

Make the script executable:

```bash
sudo chmod +x /usr/local/bin/postgres-backup.sh
```

### 2. Transfer Script

Create the file `/usr/local/bin/postgres-transfer.sh`:

```bash
#!/bin/bash

# Configuration
BACKUP_DIR="/var/backups/postgres"
REMOTE_USER="backup-user"
REMOTE_HOST="backup-server.example.com"
REMOTE_DIR="/backups/postgres"
DB_NAME="postgres"
REMOTE_PASSWORD="your_password_here"  # Hardcoding passwords is not recommended!

# Create log directory if it doesn't exist
mkdir -p /var/log

# Find the latest backup
LATEST_BACKUP=$(ls -t $BACKUP_DIR/${DB_NAME}_*.sql.gz | head -n1)

if [ -z "$LATEST_BACKUP" ]; then
  echo "No backups found to transfer!"
  exit 1
fi

# Transfer the latest backup using sshpass
sshpass -p "$REMOTE_PASSWORD" rsync -avz -e "ssh" $LATEST_BACKUP $REMOTE_USER@$REMOTE_HOST:$REMOTE_DIR/

if [ $? -ne 0 ]; then
  echo "Remote backup transfer failed!"
  exit 1
fi

echo "Transfer completed: $(basename $LATEST_BACKUP)"
```

Make the script executable:

```bash
sudo chmod +x /usr/local/bin/postgres-transfer.sh
```

### 3. Monitoring Script

Create the file `/usr/local/bin/postgres-monitor.sh`:

```bash
#!/bin/bash

# Configuration
BACKUP_DIR="/var/backups/postgres"
DB_NAME="postgres"
ADMIN_EMAIL="admin@example.com"
TODAY=$(date +"%Y%m%d")

# Create log directory if it doesn't exist
mkdir -p /var/log

# Check if backup exists for today
if ! ls $BACKUP_DIR/${DB_NAME}_${TODAY}*.sql.gz 1> /dev/null 2>&1; then
  echo "ALERT: No PostgreSQL backup found for today!" | mail -s "Backup Failed" $ADMIN_EMAIL
  echo "No backup found for today. Alert sent."
  exit 1
fi

# Check backup size
LATEST_BACKUP=$(ls -t $BACKUP_DIR/${DB_NAME}_${TODAY}*.sql.gz | head -n1)
BACKUP_SIZE=$(du -h "$LATEST_BACKUP" | cut -f1)

# Log success
echo "Backup monitoring completed. Latest backup: $(basename $LATEST_BACKUP), Size: $BACKUP_SIZE"
exit 0
```

Make the script executable:

```bash
sudo chmod +x /usr/local/bin/postgres-monitor.sh
```

### Install Required Tools

```bash
# For email alerts in the monitoring script
sudo apt-get update
sudo apt-get install -y mailutils rsync
```

## Crontab Configuration

Set up the backup schedule:

```bash
# Edit the crontab
sudo crontab -e
```

Add these lines:

```
# Run database backup daily at 2:00 AM
0 2 * * * /usr/local/bin/postgres-backup.sh >> /var/log/postgres-backup.log 2>&1

# Transfer backup to offsite location at 3:00 AM
0 3 * * * /usr/local/bin/postgres-transfer.sh >> /var/log/postgres-transfer.log 2>&1

# Check if today's backup exists at 8:00 AM
0 8 * * * /usr/local/bin/postgres-monitor.sh >> /var/log/postgres-monitor.log 2>&1
```

## Backup File Management

### Backup File Location

All database backups are stored in the following locations:

- **Primary location**: `/var/backups/postgres/` on the application server
- **Secondary location**: `/backups/postgres/` on the backup server (`backup-server.example.com`)

### Backup Naming Convention

Backup files follow this naming pattern:
```
postgres_YYYYMMDD_HHMMSS.sql.gz
```

Example: `postgres_20250301_020000.sql.gz`

### Setting Up Remote SSH Access

Generate an SSH key for the backup transfer:

```bash
sudo mkdir -p /root/.ssh
sudo ssh-keygen -t rsa -b 4096 -f /root/.ssh/backup_key -N ""
```

Copy the public key to the backup server:

```bash
sudo ssh-copy-id -i /root/.ssh/backup_key.pub backup-user@backup-server.example.com
```

## Emergency Restoration Procedure

In case of database failure, follow these steps to restore from backup:

### 1. Identify the appropriate backup file

```bash
ls -la /var/backups/postgres/
```

### 2. Stop the application containers

```bash
cd /path/to/docker-compose
docker-compose down
```

### 3. Start only the database container

```bash
docker-compose up -d postgres
```

### 4. Restore the database

```bash
# For a complete restore (replacing existing database)
gunzip -c /var/backups/postgres/postgres_YYYYMMDD_HHMMSS.sql.gz | docker exec -i postgres-container psql -U postgres
```

OR

```bash
# For restoring to a new database for verification
gunzip -c /var/backups/postgres/postgres_YYYYMMDD_HHMMSS.sql.gz | docker exec -i postgres-container psql -U postgres -c "CREATE DATABASE restored_db;" && docker exec -i postgres-container psql -U postgres -d restored_db
```

### 5. Verify the restoration

```bash
docker exec -i postgres-container psql -U postgres -d postgres -c "SELECT count(*) FROM your_main_table;"
```

### 6. Restart all application containers

```bash
cd /path/to/docker-compose
docker-compose up -d
```

## Backup Testing Schedule

The restoration process should be tested monthly following this procedure:

1. Create a test database
2. Restore the latest backup to the test database
3. Verify data integrity
4. Document the results in `/var/log/backup-tests.log`

Here's a script to automate the testing process:

```bash
#!/bin/bash
# /usr/local/bin/postgres-test-restore.sh

BACKUP_DIR="/var/backups/postgres"
DOCKER_CONTAINER="postgres-container"
DB_NAME="postgres"
TEST_DB="restore_test"
LOG_FILE="/var/log/backup-tests.log"
LATEST_BACKUP=$(ls -t $BACKUP_DIR/${DB_NAME}_*.sql.gz | head -n1)

echo "$(date): Starting backup restoration test" >> $LOG_FILE

# Create test database
docker exec -i $DOCKER_CONTAINER psql -U postgres -c "DROP DATABASE IF EXISTS $TEST_DB;"
docker exec -i $DOCKER_CONTAINER psql -U postgres -c "CREATE DATABASE $TEST_DB;"

if [ $? -ne 0 ]; then
  echo "$(date): Failed to create test database" >> $LOG_FILE
  exit 1
fi

# Restore backup to test database
gunzip -c $LATEST_BACKUP | docker exec -i $DOCKER_CONTAINER psql -U postgres -d $TEST_DB

if [ $? -ne 0 ]; then
  echo "$(date): Failed to restore backup to test database" >> $LOG_FILE
  exit 1
fi

# Verify data
TABLE_COUNT=$(docker exec -i $DOCKER_CONTAINER psql -U postgres -d $TEST_DB -t -c "SELECT count(*) FROM information_schema.tables WHERE table_schema = 'public';")

echo "$(date): Restoration test completed. Found $TABLE_COUNT tables in restored database." >> $LOG_FILE

# Clean up
docker exec -i $DOCKER_CONTAINER psql -U postgres -c "DROP DATABASE $TEST_DB;"

exit 0
```

Make the script executable:

```bash
sudo chmod +x /usr/local/bin/postgres-test-restore.sh
```

Add to crontab to run monthly:

```
# Run backup restoration test on the 1st of each month
0 1 1 * * /usr/local/bin/postgres-test-restore.sh
```

## Contact Information

For issues with the backup system, contact:

- **Primary**: DevOps Team Lead (devops-lead@example.com, +1-555-123-4567)
- **Secondary**: Database Administrator (dba@example.com, +1-555-765-4321)
