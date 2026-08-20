# PrivateBin Writable Paths - Visual Summary

## Directory Structure & Write Access Map

### Complete Directory Tree with Permissions

```
/var/www/privatebin/
│
├─ [RO] index.php                    Entry point (read-only)
├─ [RO] README.md, CHANGELOG.md      Documentation
│
├─ [RO] lib/                         PHP application code
│   ├─ [RO] Data/
│   │   ├─ AbstractData.php
│   │   ├─ Filesystem.php            ← Defines writable paths
│   │   ├─ Database.php              ← Database backend logic
│   │   ├─ S3Storage.php
│   │   └─ GoogleCloudStorage.php
│   ├─ [RO] Persistence/
│   │   ├─ ServerSalt.php            ← Generates salt on first run
│   │   ├─ TrafficLimiter.php        ← Rate limit cache logic
│   │   └─ PurgeLimiter.php          ← Purge frequency limiter logic
│   ├─ [RO] Controller.php           ← Main entry point logic
│   ├─ [RO] Configuration.php        ← Config loading logic
│   └─ [RO] ... (other libraries)
│
├─ [RO] vendor/                      Composer dependencies
│   └─ [RO] ... (external packages)
│
├─ [RO] js/                          Client-side JavaScript
│   ├─ [RO] privatebin.js            Main app logic
│   ├─ [RO] legacy.js                Browser compatibility
│   └─ [RO] ... (libraries)
│
├─ [RO] css/                         Stylesheets
│   ├─ [RO] bootstrap/
│   ├─ [RO] bootstrap5/
│   └─ [RO] prettify/
│
├─ [RO] tpl/                         HTML templates
│   ├─ [RO] bootstrap5.php
│   └─ [RO] ...
│
├─ [RO] i18n/                        Translations
│   ├─ [RO] en.json
│   ├─ [RO] fr.json
│   └─ [RO] ... (50+ languages)
│
├─ [RO] cfg/                         Configuration
│   ├─ [RO] conf.sample.php          Template (read-only)
│   └─ [WR] conf.php                 Active config (read-protected)
│
├─ [RW] data/                        **CRITICAL: MUST BE 0700**
│   │
│   ├─ [AUTO] salt.php               Created on first run
│   │                                 Content: PHP comment with 256-byte salt
│   │                                 Size: ~520 bytes
│   │                                 Frequency: 1x per installation
│   │
│   ├─ [CONT] traffic_limiter.php    Continuously updated
│   │                                 Content: var_export() of IP rate limits
│   │                                 Size: 1KB - 100KB depending on unique IPs
│   │                                 Frequency: Every request (via getValue/setValue)
│   │                                 Expires: Not automatic (persists until cleared)
│   │
│   ├─ [PERI] purge_limiter.php      Periodically updated
│   │                                 Content: Last purge timestamp
│   │                                 Size: ~50 bytes
│   │                                 Frequency: Every purge check (5-300s interval)
│   │
│   ├─ [AUTO] .htaccess              Created on first write to data/
│   │                                 Content: "Require all denied"
│   │                                 Size: ~20 bytes
│   │                                 Purpose: Apache security (deny browsing)
│   │
│   ├─ [RW] aa/                      First level of paste tree
│   │   ├─ [RW] bb/                  Second level of paste tree
│   │   │   ├─ [RW] aaaaaaaaaaaaaaaa.php            Paste #1
│   │   │   ├─ [RW] aaaaaaaaaaaaaaaa.discussion/   Comments for paste #1
│   │   │   │   ├─ [RW] aaaa.cccc.pppp.php         Comment 1
│   │   │   │   └─ [RW] aaaa.cccc2.pppp.php        Comment 2
│   │   │   │
│   │   │   ├─ [RW] bbbbbbbbbbbbbbbb.php            Paste #2
│   │   │   └─ [RW] bbbbbbbbbbbbbbbb.discussion/   Comments for paste #2
│   │   │
│   │   └─ [RW] cc/ ... [RW] ff/    Other first-level dirs
│   │
│   ├─ [IF-SQLITE] db.sq3            SQLite database file (if model=Database)
│   │                                 Permissions: 0600
│   │                                 Created: First connection
│   │                                 Contains: paste, comment, config tables
│   │
│   ├─ [IF-SQLITE] db.sq3-wal        SQLite write-ahead log (auto-managed)
│   │                                 Created: When WAL mode enabled
│   │                                 Managed: Automatically by SQLite
│   │
│   └─ [IF-SQLITE] db.sq3-shm        SQLite shared memory (auto-managed)
│                                     Created: When shared cache needed
│                                     Managed: Automatically by SQLite
│
├─ [RO] doc/                         Documentation
│   └─ [RO] ... (markdown guides)
│
├─ [RO] img/                         Images
│   └─ [RO] ... (static images)
│
└─ [RO] bin/                         Utility scripts
    ├─ [RO] administration           Admin tools
    ├─ [RO] configuration-test-generator
    └─ [RO] ... (utility scripts)

Legend:
  [RO] = Read-only
  [RW] = Read-write required
  [AUTO] = Auto-created on first run
  [CONT] = Continuously updated
  [PERI] = Periodically updated
  [IF-SQLITE] = Only if using SQLite backend
```

---

## Write Operations Timeline

### Application Startup

```
1. index.php loaded (read-only)
   ↓
2. Controller initialized
   ↓
3. Configuration loaded from cfg/conf.php
   ↓
4. Data model instantiated
   │
   ├─ If Filesystem:
   │  └─ [WRITES] data/ directory created if missing (0700)
   │
   ├─ If SQLite:
   │  └─ [WRITES] data/db.sq3 auto-created + schema
   │
   └─ If MySQL/PostgreSQL/S3/GCS:
      └─ (no local writes)
   ↓
5. Persistence store initialized
   │
   ├─ ServerSalt::get()
   │  ├─ Read: data/salt.php
   │  └─ [WRITE] Create if missing (1x per installation)
   │
   ├─ TrafficLimiter initialized
   │  ├─ Read: data/traffic_limiter.php
   │  └─ (writes on first traffic violation from new IP)
   │
   └─ PurgeLimiter initialized
      ├─ Read: data/purge_limiter.php
      └─ (writes on periodic purge check)
   ↓
6. Ready to serve requests
```

### During Request (Paste Creation)

```
User submits paste
   ↓
POST /api/create
   ↓
Controller::_create()
   ↓
Model::create()
   │
   ├─ Filesystem backend:
   │  ├─ Generate paste ID (aa/bb/aaaaaaaaaaaaaaaa format)
   │  ├─ [WRITE] data/aa/bb/ directory (mkdir 0700)
   │  └─ [WRITE] data/aa/bb/aaaaaaaaaaaaaaaa.php
   │
   ├─ SQLite backend:
   │  ├─ [WRITE] INSERT INTO paste (data, expire_date, meta)
   │  └─ SQLite auto-manages WAL/SHM files
   │
   └─ MySQL/S3/GCS:
      └─ (remote write, no local filesystem)
   ↓
TrafficLimiter::limit()
   ├─ Read: data/traffic_limiter.php
   └─ [WRITE] Update if rate limit needs tracking
   ↓
200 OK returned to client
```

### During Request (Paste Read)

```
User visits paste (e.g., GET /?abc123)
   ↓
Controller::_read()
   ↓
Model::read()
   │
   ├─ Filesystem backend:
   │  ├─ Read: data/aa/bb/aaaaaaaaaaaaaaaa.php
   │  └─ Check: data/aa/bb/aaaaaaaaaaaaaaaa (legacy format)
   │     ├─ If exists: [WRITE] Rename to .php extension
   │     └─ (file format migration)
   │
   └─ Database backend:
      └─ SELECT * FROM paste WHERE dataid = 'abc123'
   ↓
Check expiration
   ├─ If expired:
   │  └─ [WRITE] DELETE
   │     (and delete associated comments)
   │
   └─ If not expired:
      └─ Decrypt in browser, display
   ↓
TrafficLimiter::limit()
   └─ [WRITE] Update rate limit cache
   ↓
200 OK returned
```

### Periodic Operations (Background)

```
During normal requests, triggered ~once per second:

PurgeLimiter::canPurge()
   ├─ Check: data/purge_limiter.php
   └─ Every 300 seconds (configurable):
      ├─ [WRITE] Update purge_limiter.php timestamp
      └─ Call Model::delete() on all expired pastes
         ├─ Filesystem: [WRITE] unlink() paste & comment files
         ├─ Database:   [WRITE] DELETE FROM paste WHERE expire_date < now()
         └─ S3/GCS:     (remote deletion)
```

---

## Backend Comparison: Write Requirements

### Filesystem Backend

```
Writes:
  - data/salt.php (1x)
  - data/traffic_limiter.php (continuous)
  - data/purge_limiter.php (periodic)
  - data/.htaccess (1x)
  - data/[tree]/*.php (per paste/comment)

Total local writes: HIGH (every paste/comment)
Scalability: Good with deep directory trees
Backup: Entire data/ directory
```

### SQLite Backend

```
Writes:
  - data/db.sq3 (every paste/comment)
  - data/db.sq3-wal (auto, managed by SQLite)
  - data/db.sq3-shm (auto, managed by SQLite)
  - data/salt.php (via config table)
  - data/traffic_limiter.php (via config table) [optional]

Total local writes: HIGH (every write goes to db.sq3)
Scalability: Limited by single-file database (1000s of pastes OK)
Backup: Single db.sq3 file
Concurrency: SQLite handles via file locking
```

### MySQL/PostgreSQL Backend

```
Writes:
  - Remote database server only
  - No local filesystem writes

Local disk space: Minimal (~KB for config only)
Scalability: Excellent (can handle millions of pastes)
Backup: Database replication/backup tools
```

### S3/GCS Backend

```
Writes:
  - S3/GCS object storage only
  - No local filesystem writes

Local disk space: Minimal (~KB for config only)
Scalability: Unlimited (cloud storage)
Backup: Bucket versioning, cross-region replication
Consistency: Eventually consistent (GCS) or strong (S3)
```

---

## Permission Scenarios

### Scenario 1: Apache + Filesystem Backend

```bash
# Directory structure
/var/www/privatebin/
  ├─ (owned by root)
  ├─ lib/ (owned by root, 0755)
  ├─ js/ (owned by root, 0755)
  ├─ vendor/ (owned by root, 0755)
  ├─ cfg/
  │  ├─ conf.sample.php (0644)
  │  └─ conf.php (0640 - www-data must read)
  │
  └─ data/ (owned by www-data:www-data, 0700) ← CRITICAL
      ├─ salt.php (auto 0644 or 0700)
      ├─ traffic_limiter.php (auto 0644 or 0700)
      └─ ...

# Setup commands
sudo chown www-data:www-data /var/www/privatebin/data
sudo chmod 0700 /var/www/privatebin/data
sudo chmod 0640 /var/www/privatebin/cfg/conf.php
```

### Scenario 2: PHP-FPM + SQLite

```bash
# Directory structure (same as above)
/var/www/privatebin/
  └─ data/ (owned by www-data:www-data, 0700)
      ├─ db.sq3 (auto-created, 0600)
      ├─ db.sq3-wal (auto-created, 0600)
      └─ db.sq3-shm (auto-created, 0600)

# Troubleshooting SQLite locked error
sudo chmod 0600 /var/www/privatebin/data/db.sq3*
sudo chown www-data:www-data /var/www/privatebin/data/db.sq3*
```

### Scenario 3: Container + S3

```dockerfile
FROM php:8.1-apache

# S3 requires zero local filesystem writes
# Configure via environment variables
ENV AWS_S3_BUCKET=privatebin
ENV AWS_S3_REGION=eu-central-1

# AWS credentials via:
# 1. Environment variables (AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY)
# 2. IAM role (if running on EC2)
# 3. ~/.aws/credentials (if mounting home directory)

# No data/ directory needed for write access
# (cfg/conf.php is still read-only)
```

---

## Permission Check Commands

```bash
# Check data directory
ls -ld /var/www/privatebin/data
# Expected: drwx------ www-data www-data

# Check paste files
ls -la /var/www/privatebin/data/aa/bb/
# Expected: -rw------- www-data www-data (or -rw-r--r--)

# Check SQLite files
ls -la /var/www/privatebin/data/db.sq3*
# Expected: -rw------- www-data www-data

# Check persistence files
ls -la /var/www/privatebin/data/*.php
# Expected: -rw------- or -rw-r--r-- www-data www-data

# Verify config is protected
ls -la /var/www/privatebin/cfg/conf.php
# Expected: -rw-r----- www-data root (or similar, NOT world-readable)

# Verify read-only areas
ls -ld /var/www/privatebin/lib
ls -ld /var/www/privatebin/vendor
ls -ld /var/www/privatebin/js
# Expected: all owned by root, not writable by www-data
```

---

## Summary: What Needs Write Access?

✓ **MUST be writable:**
- `data/` directory (permissions 0700)
- Any files created within `data/` by PrivateBin

✗ **Must NOT be writable:**
- `lib/`, `js/`, `css/`, `tpl/`, `i18n/`, `vendor/`
- `cfg/conf.php` (readable only)
- `index.php` and all root-level PHP files

~ **Depends on backend:**
- Remote database (not local filesystem)
- S3/GCS buckets (not local filesystem)
- Error logs (configured in php.ini, not by PrivateBin)

---

## File Size Predictions

### Typical Installation (1000 pastes)

```
data/
├─ .htaccess                    20 bytes
├─ salt.php                     520 bytes
├─ traffic_limiter.php          10 KB (varies with unique IPs)
├─ purge_limiter.php            50 bytes
│
└─ [paste tree]                 ~5-100 MB
   (depends on paste size limit and content)
   
Total: ~5-100 MB for 1000 pastes
Database (SQLite): ~1-50 MB for equivalent data
```

### High-Traffic Installation (1,000,000 pastes)

```
Filesystem backend: 5-100 GB
SQLite backend: 1-50 GB
MySQL backend: 2-100 GB
S3/GCS: Unlimited, pay per usage
```

---

## Monitoring & Alerts

### Recommended Monitoring

```bash
# Check data directory size
du -sh /var/www/privatebin/data

# Monitor for permission changes
stat /var/www/privatebin/data
# Alert if permissions != 0700

# Monitor database lock contention (SQLite)
lsof | grep db.sq3
# Alert if many processes have file open

# Monitor error log for write failures
tail -f /var/log/php-errors.log | grep -i "store\|write\|permission"
```

### Alert Triggers

- `data/` permission changes
- `db.sq3` locked errors
- "Error while trying to store data to the filesystem"
- "failed to store the server salt"
- "failed to store the traffic limiter"
- Disk space usage exceeds threshold
