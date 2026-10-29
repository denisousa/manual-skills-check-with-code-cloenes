<!-- Alex-65/tdcweb:.claude/skills/tdc-database/SKILL.md | https://github.com/Alex-65/tdcweb/blob/HEAD/.claude/skills/tdc-database/SKILL.md -->

---
name: tdc-database
description: Manage MySQL database for The Dreamer's Cave. Use for schema, migrations, queries, backup/restore.
---

# TDC Database Administrator

Expert agent for managing the MySQL database for The Dreamer's Cave virtual music club website.

## Trigger

Use this skill when:
- User asks about database schema or tables
- User says "/database", "/tdc-database", "/db", "/schema", or "/migration"
- User wants to create, modify, or query database tables
- User needs to run migrations or manage database
- User asks about backup/restore procedures

## Project Context

**The Dreamer's Cave** - Website for a virtual music club in Second Life.

### Database Credentials

| Type | User | Password | Database |
|------|------|----------|----------|
| Admin | admin | rutt1n0 | - |
| App | tdcweb | tdcweb | tdcweb |

### Database Standards

| Standard | Value |
|----------|-------|
| Engine | InnoDB |
| Charset | utf8mb4 |
| Collation | utf8mb4_unicode_ci |
| Naming | snake_case |
| Timestamps | created_at, updated_at on ALL tables |
| Soft Delete | is_active BOOLEAN flag |
| Flexible Data | JSON columns |

### File Structure

```
sql/
├── schema.sql              # COMPLETE current schema (ALWAYS keep updated!)
├── migrations/
│   ├── 001_initial.sql     # Initial schema
│   ├── 002_add_xxx.sql     # Subsequent migrations
│   └── ...
├── seeds/
│   ├── languages.sql       # Initial supported languages
│   ├── locations.sql       # Sample locations
│   └── admin_user.sql      # Initial admin user
└── scripts/
    ├── init_db.sh          # Initialize database from scratch
    ├── migrate.sh          # Apply pending migrations
    ├── backup.sh           # Create backup
    └── restore.sh          # Restore from backup
```

## Instructions

### Phase 1: Database Tables Overview

**28 tables total:**

| Category | Tables |
|----------|--------|
| **Auth** | users, oauth_accounts, password_reset_tokens |
| **Locations** | locations, location_translations |
| **Artists** | artists, artist_translations |
| **Staff** | staff, staff_translations |
| **Events** | events, event_translations, event_artists |
| **Blog** | blog_posts, blog_post_translations, blog_categories, blog_category_translations, blog_post_categories |
| **Media** | media |
| **Patreon** | patreon_tiers, patreon_supporters, exclusive_content, exclusive_content_translations |
| **Notifications** | user_notification_preferences, notification_queue |
| **System** | integration_settings, integration_logs, site_settings, supported_languages |

### Phase 2: Translation Tables Pattern

All content tables use the `*_translations` pattern:

```sql
-- Main table (language-independent data)
CREATE TABLE locations (
    id INT AUTO_INCREMENT PRIMARY KEY,
    slug VARCHAR(100) NOT NULL UNIQUE,
    name VARCHAR(255) NOT NULL,           -- Default/fallback name
    capacity INT DEFAULT 100,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Translation table (language-specific data)
CREATE TABLE location_translations (
    id INT AUTO_INCREMENT PRIMARY KEY,
    location_id INT NOT NULL,
    language VARCHAR(10) NOT NULL,        -- 'en', 'it', 'fr', 'es'
    name VARCHAR(255),                    -- NULL = use main table name
    tagline VARCHAR(500),
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    FOREIGN KEY (location_id) REFERENCES locations(id) ON DELETE CASCADE,
    UNIQUE KEY unique_location_lang (location_id, language),
    INDEX idx_language (language)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

**Query pattern with translation:**

```sql
-- Get location with translation (fallback to main name)
SELECT
    l.*,
    COALESCE(lt.name, l.name) AS name,
    lt.tagline,
    lt.description
FROM locations l
LEFT JOIN location_translations lt
    ON l.id = lt.location_id AND lt.language = 'it'
WHERE l.slug = 'dreamerscave' AND l.is_active = TRUE;
```

### Phase 3: Migration System

#### Migration File Format

```sql
-- migrations/002_add_location_images.sql
-- Description: Add gallery images support to locations
-- Date: 2025-01-15
-- Author: Claude

-- ============================================
-- UP Migration
-- ============================================

-- Add location_images table
CREATE TABLE location_images (
    id INT AUTO_INCREMENT PRIMARY KEY,
    location_id INT NOT NULL,
    image_url VARCHAR(500) NOT NULL,
    thumbnail_url VARCHAR(500),
    caption VARCHAR(255),
    sort_order INT DEFAULT 0,
    is_primary BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY (location_id) REFERENCES locations(id) ON DELETE CASCADE,
    INDEX idx_location (location_id),
    INDEX idx_primary (is_primary)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Record migration
INSERT INTO schema_migrations (version, description, applied_at)
VALUES ('002', 'add_location_images', NOW());

-- ============================================
-- DOWN Migration (for rollback - keep commented)
-- ============================================
-- DROP TABLE IF EXISTS location_images;
-- DELETE FROM schema_migrations WHERE version = '002';
```

#### Migration Tracking Table

```sql
-- Add to initial migration
CREATE TABLE schema_migrations (
    id INT AUTO_INCREMENT PRIMARY KEY,
    version VARCHAR(10) NOT NULL UNIQUE,
    description VARCHAR(255),
    applied_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_version (version)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### Phase 4: Init Script (scripts/init_db.sh)

```bash
#!/bin/bash
# Initialize tdcweb database from scratch
# Usage: ./init_db.sh [--drop-existing]

set -e

# Configuration
DB_ROOT_USER="admin"
DB_ROOT_PASS="rutt1n0"
DB_NAME="tdcweb"
DB_USER="tdcweb"
DB_PASS="tdcweb"

SQL_DIR="$(dirname "$0")/.."

# Colors
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

echo -e "${GREEN}=== TDC Database Initialization ===${NC}"

# Check for --drop-existing flag
if [ "$1" == "--drop-existing" ]; then
    echo -e "${YELLOW}WARNING: This will DROP the existing database!${NC}"
    read -p "Are you sure? (yes/no): " confirm
    if [ "$confirm" != "yes" ]; then
        echo "Aborted."
        exit 1
    fi

    echo "Dropping existing database..."
    mysql -u"$DB_ROOT_USER" -p"$DB_ROOT_PASS" -e "DROP DATABASE IF EXISTS $DB_NAME;"
fi

# Create database
echo "Creating database '$DB_NAME'..."
mysql -u"$DB_ROOT_USER" -p"$DB_ROOT_PASS" <<EOF
CREATE DATABASE IF NOT EXISTS $DB_NAME
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;
EOF

# Create user (ignore if exists)
echo "Creating user '$DB_USER'..."
mysql -u"$DB_ROOT_USER" -p"$DB_ROOT_PASS" <<EOF
CREATE USER IF NOT EXISTS '$DB_USER'@'localhost' IDENTIFIED BY '$DB_PASS';
GRANT ALL PRIVILEGES ON $DB_NAME.* TO '$DB_USER'@'localhost';
FLUSH PRIVILEGES;
EOF

# Apply schema
echo "Applying schema..."
mysql -u"$DB_USER" -p"$DB_PASS" "$DB_NAME" < "$SQL_DIR/schema.sql"

# Apply seeds
echo "Applying seed data..."
for seed_file in "$SQL_DIR"/seeds/*.sql; do
    if [ -f "$seed_file" ]; then
        echo "  - $(basename "$seed_file")"
        mysql -u"$DB_USER" -p"$DB_PASS" "$DB_NAME" < "$seed_file"
    fi
done

echo -e "${GREEN}Database initialized successfully!${NC}"
echo ""
echo "Connection info:"
echo "  Host: localhost"
echo "  Database: $DB_NAME"
echo "  User: $DB_USER"
echo "  Password: $DB_PASS"
```

### Phase 5: Migration Script (scripts/migrate.sh)

```bash
#!/bin/bash
# Apply pending database migrations
# Usage: ./migrate.sh [--dry-run]

set -e

# Configuration
DB_USER="tdcweb"
DB_PASS="tdcweb"
DB_NAME="tdcweb"

SQL_DIR="$(dirname "$0")/.."
MIGRATIONS_DIR="$SQL_DIR/migrations"

# Colors
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

DRY_RUN=false
if [ "$1" == "--dry-run" ]; then
    DRY_RUN=true
    echo -e "${YELLOW}DRY RUN - No changes will be applied${NC}"
fi

echo -e "${GREEN}=== TDC Database Migrations ===${NC}"

# Get applied migrations
applied=$(mysql -u"$DB_USER" -p"$DB_PASS" "$DB_NAME" -N -e \
    "SELECT version FROM schema_migrations ORDER BY version;" 2>/dev/null || echo "")

# Find pending migrations
pending=()
for migration_file in "$MIGRATIONS_DIR"/*.sql; do
    if [ -f "$migration_file" ]; then
        filename=$(basename "$migration_file")
        version=$(echo "$filename" | grep -oP '^\d+')

        if ! echo "$applied" | grep -q "^$version$"; then
            pending+=("$migration_file")
        fi
    fi
done

if [ ${#pending[@]} -eq 0 ]; then
    echo "No pending migrations."
    exit 0
fi

echo "Pending migrations: ${#pending[@]}"
echo ""

# Apply pending migrations
for migration_file in "${pending[@]}"; do
    filename=$(basename "$migration_file")
    echo -e "${YELLOW}Applying: $filename${NC}"

    if [ "$DRY_RUN" = false ]; then
        mysql -u"$DB_USER" -p"$DB_PASS" "$DB_NAME" < "$migration_file"
        echo -e "${GREEN}  ✓ Applied${NC}"
    else
        echo "  [DRY RUN] Would apply $filename"
    fi
done

echo ""
echo -e "${GREEN}Migrations complete!${NC}"
```

### Phase 6: Backup Script (scripts/backup.sh)

```bash
#!/bin/bash
# Create database backup
# Usage: ./backup.sh [backup_name]

set -e

# Configuration
DB_USER="tdcweb"
DB_PASS="tdcweb"
DB_NAME="tdcweb"

BACKUP_DIR="/data1/tdcweb/backups"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_NAME="${1:-tdcweb_${TIMESTAMP}}"

# Colors
GREEN='\033[0;32m'
NC='\033[0m'

# Create backup directory
mkdir -p "$BACKUP_DIR"

echo -e "${GREEN}=== TDC Database Backup ===${NC}"
echo "Creating backup: ${BACKUP_NAME}.sql.gz"

# Create backup with mysqldump
mysqldump -u"$DB_USER" -p"$DB_PASS" \
    --single-transaction \
    --routines \
    --triggers \
    --add-drop-table \
    "$DB_NAME" | gzip > "$BACKUP_DIR/${BACKUP_NAME}.sql.gz"

# Show backup info
BACKUP_SIZE=$(ls -lh "$BACKUP_DIR/${BACKUP_NAME}.sql.gz" | awk '{print $5}')
echo ""
echo -e "${GREEN}Backup complete!${NC}"
echo "  File: $BACKUP_DIR/${BACKUP_NAME}.sql.gz"
echo "  Size: $BACKUP_SIZE"

# Cleanup old backups (keep last 10)
echo ""
echo "Cleaning up old backups (keeping last 10)..."
ls -t "$BACKUP_DIR"/*.sql.gz 2>/dev/null | tail -n +11 | xargs -r rm -f

# List recent backups
echo ""
echo "Recent backups:"
ls -lht "$BACKUP_DIR"/*.sql.gz 2>/dev/null | head -5
```

### Phase 7: Restore Script (scripts/restore.sh)

```bash
#!/bin/bash
# Restore database from backup
# Usage: ./restore.sh <backup_file>

set -e

# Configuration
DB_ROOT_USER="admin"
DB_ROOT_PASS="rutt1n0"
DB_NAME="tdcweb"
DB_USER="tdcweb"
DB_PASS="tdcweb"

# Colors
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

if [ -z "$1" ]; then
    echo "Usage: $0 <backup_file.sql.gz>"
    echo ""
    echo "Available backups:"
    ls -lht /data1/tdcweb/backups/*.sql.gz 2>/dev/null | head -10
    exit 1
fi

BACKUP_FILE="$1"

if [ ! -f "$BACKUP_FILE" ]; then
    echo -e "${RED}Error: Backup file not found: $BACKUP_FILE${NC}"
    exit 1
fi

echo -e "${GREEN}=== TDC Database Restore ===${NC}"
echo "Backup file: $BACKUP_FILE"
echo ""
echo -e "${YELLOW}WARNING: This will REPLACE all data in database '$DB_NAME'!${NC}"
read -p "Are you sure? (yes/no): " confirm

if [ "$confirm" != "yes" ]; then
    echo "Aborted."
    exit 1
fi

# Drop and recreate database
echo "Recreating database..."
mysql -u"$DB_ROOT_USER" -p"$DB_ROOT_PASS" <<EOF
DROP DATABASE IF EXISTS $DB_NAME;
CREATE DATABASE $DB_NAME
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;
GRANT ALL PRIVILEGES ON $DB_NAME.* TO '$DB_USER'@'localhost';
EOF

# Restore from backup
echo "Restoring data..."
gunzip -c "$BACKUP_FILE" | mysql -u"$DB_USER" -p"$DB_PASS" "$DB_NAME"

echo ""
echo -e "${GREEN}Restore complete!${NC}"

# Show table counts
echo ""
echo "Table row counts:"
mysql -u"$DB_USER" -p"$DB_PASS" "$DB_NAME" -e "
SELECT table_name, table_rows
FROM information_schema.tables
WHERE table_schema = '$DB_NAME'
ORDER BY table_name;"
```

### Phase 8: Common Query Patterns

#### CRUD with Translations

```sql
-- ============================================
-- CREATE with translation
-- ============================================
-- Step 1: Insert main record
INSERT INTO locations (slug, name, capacity, mood_category, primary_color)
VALUES ('new-location', 'New Location', 150, 'cosmic_tech', '#06b6d4');

-- Step 2: Get the ID
SET @location_id = LAST_INSERT_ID();

-- Step 3: Insert translations
INSERT INTO location_translations (location_id, language, name, tagline, description)
VALUES
    (@location_id, 'en', 'New Location', 'English tagline', 'English description'),
    (@location_id, 'it', 'Nuova Location', 'Tagline italiano', 'Descrizione italiana');


-- ============================================
-- READ with translation (single)
-- ============================================
SELECT
    l.id,
    l.slug,
    COALESCE(lt.name, l.name) AS name,
    l.capacity,
    l.mood_category,
    l.primary_color,
    l.secondary_color,
    l.accent_color,
    l.css_gradient,
    lt.tagline,
    lt.description,
    lt.architecture_notes,
    lt.atmosphere_notes
FROM locations l
LEFT JOIN location_translations lt
    ON l.id = lt.location_id AND lt.language = %s
WHERE l.slug = %s AND l.is_active = TRUE;


-- ============================================
-- READ with translation (list)
-- ============================================
SELECT
    l.id,
    l.slug,
    COALESCE(lt.name, l.name) AS name,
    l.capacity,
    l.mood_category,
    lt.tagline
FROM locations l
LEFT JOIN location_translations lt
    ON l.id = lt.location_id AND lt.language = %s
WHERE l.is_active = TRUE
ORDER BY l.sort_order, l.name;


-- ============================================
-- UPDATE main record
-- ============================================
UPDATE locations
SET
    name = %s,
    capacity = %s,
    primary_color = %s,
    updated_at = NOW()
WHERE id = %s;


-- ============================================
-- UPDATE or INSERT translation (UPSERT)
-- ============================================
INSERT INTO location_translations
    (location_id, language, name, tagline, description)
VALUES (%s, %s, %s, %s, %s)
ON DUPLICATE KEY UPDATE
    name = VALUES(name),
    tagline = VALUES(tagline),
    description = VALUES(description),
    updated_at = NOW();


-- ============================================
-- SOFT DELETE (set inactive)
-- ============================================
UPDATE locations SET is_active = FALSE WHERE id = %s;

-- HARD DELETE (cascades to translations)
DELETE FROM locations WHERE id = %s;
```

#### Pagination

```sql
-- ============================================
-- Paginated list with total count
-- ============================================

-- Get total count
SELECT COUNT(*) AS total
FROM events e
WHERE e.is_published = TRUE
  AND e.start_time >= NOW();

-- Get paginated results (page 2, 20 per page)
SELECT
    e.id,
    e.slug,
    et.title,
    e.event_type,
    e.start_time,
    e.end_time,
    l.name AS location_name,
    e.poster_image_url
FROM events e
LEFT JOIN event_translations et
    ON e.id = et.event_id AND et.language = %s
LEFT JOIN locations l ON e.location_id = l.id
WHERE e.is_published = TRUE
  AND e.start_time >= NOW()
ORDER BY e.start_time ASC
LIMIT 20 OFFSET 20;  -- Page 2: OFFSET = (page - 1) * per_page
```

#### Complex JOIN Queries

```sql
-- ============================================
-- Event with location, artists, and translations
-- ============================================
SELECT
    e.id,
    e.slug,
    et.title,
    et.description,
    e.event_type,
    e.start_time,
    e.end_time,
    e.timezone,
    e.poster_image_url,
    -- Location info
    l.slug AS location_slug,
    COALESCE(lt.name, l.name) AS location_name,
    l.primary_color AS location_color,
    l.slurl AS location_slurl,
    -- Artists (as JSON array)
    (
        SELECT JSON_ARRAYAGG(
            JSON_OBJECT(
                'id', a.id,
                'slug', a.slug,
                'name', a.name,
                'performance_type', a.performance_type,
                'profile_image_url', a.profile_image_url
            )
        )
        FROM event_artists ea
        JOIN artists a ON ea.artist_id = a.id
        WHERE ea.event_id = e.id
        ORDER BY ea.performance_order
    ) AS artists
FROM events e
LEFT JOIN event_translations et
    ON e.id = et.event_id AND et.language = %s
LEFT JOIN locations l ON e.location_id = l.id
LEFT JOIN location_translations lt
    ON l.id = lt.location_id AND lt.language = %s
WHERE e.slug = %s AND e.is_published = TRUE;


-- ============================================
-- User's favorite events (based on preferences)
-- ============================================
SELECT DISTINCT e.*, et.title
FROM events e
JOIN event_translations et ON e.id = et.event_id AND et.language = %s
LEFT JOIN event_artists ea ON e.id = ea.event_id
JOIN user_notification_preferences unp ON unp.user_id = %s
WHERE e.is_published = TRUE
  AND e.start_time >= NOW()
  AND (
      -- Favorite location
      JSON_CONTAINS(unp.favorite_locations, CAST(e.location_id AS JSON))
      OR
      -- Favorite artist
      JSON_CONTAINS(unp.favorite_artists, CAST(ea.artist_id AS JSON))
      OR
      -- Favorite event type
      JSON_CONTAINS(unp.favorite_event_types, JSON_QUOTE(e.event_type))
  )
ORDER BY e.start_time
LIMIT 10;
```

#### Full-Text Search (Blog)

```sql
-- ============================================
-- Search blog posts
-- ============================================
SELECT
    bp.id,
    bp.slug,
    bpt.title,
    bpt.excerpt,
    bp.featured_image_url,
    bp.published_at,
    u.username AS author_name,
    -- Relevance score
    MATCH(bpt.title, bpt.content) AGAINST (%s IN NATURAL LANGUAGE MODE) AS relevance
FROM blog_posts bp
JOIN blog_post_translations bpt
    ON bp.id = bpt.post_id AND bpt.language = %s
LEFT JOIN users u ON bp.author_id = u.id
WHERE bp.status = 'published'
  AND MATCH(bpt.title, bpt.content) AGAINST (%s IN NATURAL LANGUAGE MODE)
ORDER BY relevance DESC
LIMIT 20;
```

### Phase 9: Useful MySQL Commands

```bash
# ============================================
# Connection
# ============================================

# Connect as app user
mysql -u tdcweb -ptdcweb tdcweb

# Connect as admin
mysql -u admin -prutt1n0

# ============================================
# Database Info
# ============================================

# Show tables
SHOW TABLES;

# Show table structure
DESCRIBE users;
SHOW CREATE TABLE users;

# Show indexes
SHOW INDEX FROM users;

# Table sizes
SELECT
    table_name,
    table_rows,
    ROUND(data_length / 1024 / 1024, 2) AS 'Data (MB)',
    ROUND(index_length / 1024 / 1024, 2) AS 'Index (MB)'
FROM information_schema.tables
WHERE table_schema = 'tdcweb'
ORDER BY data_length DESC;

# ============================================
# Debug Queries
# ============================================

# Show running queries
SHOW PROCESSLIST;

# Explain query plan
EXPLAIN SELECT * FROM events WHERE start_time > NOW();
EXPLAIN ANALYZE SELECT * FROM events WHERE start_time > NOW();

# Show last queries (if slow query log enabled)
SELECT * FROM mysql.slow_log ORDER BY start_time DESC LIMIT 10;

# ============================================
# User Management
# ============================================

# List users
SELECT user, host FROM mysql.user;

# Check grants
SHOW GRANTS FOR 'tdcweb'@'localhost';

# Reset app user password
ALTER USER 'tdcweb'@'localhost' IDENTIFIED BY 'new_password';

# ============================================
# Maintenance
# ============================================

# Check table integrity
CHECK TABLE users;

# Optimize table (reclaim space)
OPTIMIZE TABLE notification_queue;

# Analyze table (update statistics)
ANALYZE TABLE events;

# ============================================
# Export/Import
# ============================================

# Export single table
mysqldump -u tdcweb -ptdcweb tdcweb locations > locations.sql

# Export with data only (no schema)
mysqldump -u tdcweb -ptdcweb tdcweb --no-create-info locations > locations_data.sql

# Import SQL file
mysql -u tdcweb -ptdcweb tdcweb < locations.sql

# ============================================
# JSON Operations
# ============================================

# Query JSON field
SELECT * FROM artists WHERE JSON_CONTAINS(social_links, '"facebook.com"', '$.facebook');

# Extract from JSON
SELECT name, JSON_EXTRACT(social_links, '$.facebook') AS facebook FROM artists;

# Update JSON field
UPDATE artists
SET social_links = JSON_SET(social_links, '$.twitter', 'https://twitter.com/artist')
WHERE id = 1;
```

### Phase 10: Schema Maintenance Rules

**CRITICAL: Always keep schema.sql updated!**

After any migration:

1. **Update schema.sql** to reflect current state:
   ```bash
   # Generate fresh schema
   mysqldump -u tdcweb -ptdcweb --no-data tdcweb > sql/schema.sql

   # Or manually add the changes to schema.sql
   ```

2. **Add seed data** if introducing new required records

3. **Test migration** on a fresh database:
   ```bash
   # Create test database
   mysql -u admin -prutt1n0 -e "CREATE DATABASE tdcweb_test;"

   # Apply schema
   mysql -u tdcweb -ptdcweb tdcweb_test < sql/schema.sql

   # Verify
   mysql -u tdcweb -ptdcweb tdcweb_test -e "SHOW TABLES;"

   # Cleanup
   mysql -u admin -prutt1n0 -e "DROP DATABASE tdcweb_test;"
   ```

### Phase 11: Git Workflow

After database changes:

1. **Update files:**
   - `sql/schema.sql` - Complete current schema
   - `sql/migrations/NNN_xxx.sql` - New migration file
   - `sql/seeds/*.sql` - Any new seed data

2. **Test locally:**
   ```bash
   ./sql/scripts/migrate.sh --dry-run
   ./sql/scripts/migrate.sh
   ```

3. **Commit:**
   ```bash
   git add sql/
   git commit -m "db: [description]

   - Migration: NNN_xxx.sql
   - Changes: ...

   🤖 Generated with [Claude Code](https://claude.com/claude-code)

   Co-Authored-By: Claude <noreply@anthropic.com>"
   ```

## Important Notes

### Security

- NEVER commit real passwords (use .env.example patterns)
- NEVER use string concatenation in queries - always use parameterized queries
- Backup before any destructive operation
- Test migrations on staging before production

### Performance

- Add indexes for frequently queried columns
- Use EXPLAIN to analyze slow queries
- Paginate large result sets
- Use appropriate column types (INT vs BIGINT, VARCHAR length)

### Data Integrity

- Always use foreign keys with appropriate ON DELETE actions
- Use transactions for multi-step operations
- Validate data before insertion
- Use ENUM for fixed value sets

### Translation Tables

- Main table stores language-independent data
- Translation table stores language-specific content
- Always LEFT JOIN to allow missing translations
- Use COALESCE for fallback to default name
- Supported languages: en (default), it, fr, es

### Naming Conventions

- Tables: plural, snake_case (users, blog_posts)
- Columns: snake_case (created_at, user_id)
- Foreign keys: {referenced_table_singular}_id (user_id, location_id)
- Indexes: idx_{column} or idx_{columns} (idx_email, idx_user_status)
- Unique keys: unique_{description} (unique_provider_account)
