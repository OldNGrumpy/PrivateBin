# PrivateBin Runtime Writable Paths - Production Deployment Guide

## Executive Summary

This document identifies ALL files and directories that MUST have write access at runtime in a production PrivateBin deployment. Understanding these paths is critical for:

- Setting appropriate file permissions
- Configuring SELinux/AppArmor policies
- Planning backup strategies
- Implementing containerization or sandboxing
- Troubleshooting permission-related issues

## Overview of Runtime Write Operations

PrivateBin writes to storage in **four primary contexts**:

1. **Paste & Comment Storage** - Persistent user-created content
2. **Persistence Files** - Server operational state (salt, traffic limiter, purge limiter)
3. **Database Files** - SQLite database (optional, depends on configuration)
4. **Cache Control** - HTTP security headers (only for filesystem mode)

## Critical Directories & Files

### 1. Data Storage Directory

**Location:** Configured in `cfg/conf.php` under `[model_options] dir`  
**Default:** `data/` (relative to project root)  
**Absolute Path:** `PATH/data/` where PATH is defined in `index.php`

#### Structure & Contents

```
data/
├── .htaccess                           # Apache security header (auto-created)
├── salt.php                            # Server salt file (auto-created on first run)
├── traffic_limiter.php                 # Traffic rate-limiter cache (auto-created)
├── purge_limiter.php                   # Purge operation limiter (auto-created)
├── [2-char]/                           # First level of paste directory tree
│   └── [2-char]/                       # Second level of paste directory tree
│       ├── [16-char-paste-id].php      # Encrypted paste file
│       ├── [16-char-paste-id].discussion/
│       │   └── [id].[comment-id].[parent-id].php  # Comment files
│       └── ...
└── ...
```

#### Permission Requirements

- **Directory Permissions:** `0700` (rwx------)
  - Owner: web server process user (e.g., `www-data`, `apache`, `nobody`)
  - Group: read-only or none
  - Others: none
  
- **File Permissions:** Created as `0700` or with LOCK_EX
  - All files require read+write access
  - Files include protection line: `<?php http_response_code(403); /*`

#### What Gets Written

| File Type | Created | Frequency | Purpose | Size Impact |
|-----------|---------|-----------|---------|------------|
| Paste files (`.php`) | Per paste | Rare | Store encrypted paste data | Configurable max per paste |
| Comment files (`.php`) | Per comment | Rare | Store encrypted discussion comments | Configurable max per comment |
| `.htaccess` | First run | Once | Apache security (deny directory browsing) | ~20 bytes |
| `salt.php` | First run | Once | Server-unique salt (256 bytes hex) | ~520 bytes |
| `traffic_limiter.php` | Per violation check | Continuous | Cache IP rate-limits (persistent across requests) | Grows with unique IPs, ~1KB-100KB |
| `purge_limiter.php` | Per purge check | Periodic | Track last expiration sweep (5-300s intervals) | ~50 bytes |

#### Scalability Notes

High-traffic deployments may want to **deepen the directory structure**:
- Currently: `data/XX/YY/XXXXXXXXXXXXXXXX` (3 levels)
- High-traffic option: `data/XX/YY/ZZ/XXXXXXXXXXXXXXXX` (4 levels)
- This limits files per directory to prevent filesystem slowdowns

### 2. Persistence Files (Filesystem Backend Only)

When using the **Filesystem data model** (default), all persistence occurs in the `data/` directory above.

**Other backends are read-only from the app perspective:**
- **Database backend:** Writes to database (see Section 4)
- **S3/GCS backends:** Writes to external storage; local filesystem has zero write requirements

### 3. Database Files (Database Backend)

**Configuration:** `[model] class = Database` in `cfg/conf.php`

#### SQLite (Default if Database Model Selected)

**Location:** `data/db.sq3` (configured via `[model_options] dsn`)

```ini
[model]
class = Database
[model_options]
dsn = "sqlite:" PATH "data/db.sq3"
usr = null
pwd = null
opt[12] = true  ; PDO::ATTR_PERSISTENT
```

**Permission Requirements:**
- File: `0600` (rw-------)
- Directory: `0700` (rwx------)
- Write access: Continuous (insert/update on every paste/comment)
- Concurrent access: SQLite handles via file locking (LOCK_EX in PHP)

**Database Tables Created:**
- `paste` - Encrypted paste data, expiration dates, metadata
- `comment` - Comment data, associated paste ID, post dates
- `config` - System configuration (VERSION, traffic_limiter, salt migration)

**Automatic Initialization:**
- Database tables are auto-created on first connection if they don't exist
- Database schema is auto-upgraded if version mismatch detected
- No manual migration scripts required

**What Gets Written:**

| Operation | Frequency | Data |
|-----------|-----------|------|
| INSERT paste | Per paste creation | Encrypted payload + metadata |
| INSERT comment | Per comment creation | Encrypted comment + metadata |
| DELETE paste/comment | On expiration or user delete | N/A |
| UPDATE config | On traffic/purge/salt changes | traffic_limiter, purge_limiter, salt |
| Schema updates | On version upgrade | ALTER TABLE operations |

#### Other Database Backends

- **MySQL/MariaDB:** No local files (hosted on database server)
- **PostgreSQL:** No local files (hosted on database server)
- **S3/GCS:** No local SQLite file

### 4. Operational Files Generated at Runtime

#### `.htaccess` File (Apache Only)

**Location:** `data/.htaccess`

**Content:**
```
Require all denied
```

**Purpose:** Prevents web-accessible directory listing if data directory is exposed

**When Created:** First time `Filesystem::_storeString()` is called (lazy creation)

**Permissions:** `0644` (rw-r--r--) or auto-created with default umask

---

## Files That Require Write Access Summary

### Filesystem Backend (Default)

```
✓ data/                             [MUST BE WRITABLE - 0700]
  ✓ .htaccess                       [AUTO-CREATED]
  ✓ salt.php                        [AUTO-CREATED, 1x per installation]
  ✓ traffic_limiter.php             [CONTINUOUSLY WRITTEN]
  ✓ purge_limiter.php               [PERIODICALLY WRITTEN]
  ✓ [2-char]/[2-char]/*.php         [CONTINUOUSLY CREATED/DELETED]
  ✓ [2-char]/[2-char]/*.discussion/*.php [CONTINUOUSLY CREATED/DELETED]
```

### Database Backend (SQLite)

```
✓ data/                             [MUST EXIST, needs parent write for SQLite temp files]
  ✓ db.sq3                          [DATABASE FILE - 0600]
  ✓ db.sq3-wal                      [SQLITE WRITE-AHEAD LOG - auto-created, 0600]
  ✓ db.sq3-shm                      [SQLITE SHARED MEMORY - auto-created, 0600]
```

### Database Backend (MySQL/PostgreSQL/Remote)

```
✓ No local filesystem writes required
  (All writes go to remote database server)
```

### S3/Google Cloud Storage Backend

```
✓ No local filesystem writes required
  (All writes go to remote object storage)
✓ Optional: data/ directory
  (Only if you want to store local SQLite cache for config - not recommended)
```

---

## Directories That Can Remain Read-Only

### Safe to Make Read-Only

```
✓ tpl/                              [HTML templates - static]
✓ css/                              [Stylesheets - static]
✓ js/                               [Client-side scripts - static]
✓ i18n/                             [Translation files - static]
✓ lib/                              [PHP source code - static]
✓ vendor/                           [Composer dependencies - static]
✓ cfg/                              [Configuration files - static]
✓ doc/                              [Documentation - static]
✓ img/                              [Images - static]
✓ .htaccess (root)                  [Static]
✓ index.php                         [Entry point - static]
```

### Must Remain Readable (Not Write)

```
✓ All files above are read-only
✓ PHP source must always be readable by web server
```

---

## Error Logging & PHP Error Handling

### PHP error_log() Output

**Location:** Determined by `php.ini` configuration, NOT PrivateBin config

```ini
; php.ini
error_log = /var/log/php-errors.log
; OR
error_log = syslog
; OR
error_log = /dev/stderr  (container deployments)
```

**PrivateBin uses error_log() for:**
- Database/filesystem operation failures
- Configuration errors
- Traffic/purge limiter failures
- Proxy errors (URL shortener, YOURLS, Shlink)

**Note:** PrivateBin does NOT manage the error_log file itself. This is configured outside the application.

### No Session Directory Configuration

PrivateBin does NOT use server-side PHP sessions (`session_start()`). 

**Instead:**
- Template selection stored in browser cookies (optional)
- Language selection stored in browser cookies (optional)
- No session files are written to disk

---

## Backup & Archival Considerations

### What to Backup

- **Critical:** `data/` directory (contains all user pastes and metadata)
- **Critical:** Database files if using Database backend
- **Configuration:** `cfg/conf.php` (contains secrets and settings)

### What NOT to Backup (Cache/Temporary)

- `traffic_limiter.php` - Cache, regenerates on access
- `purge_limiter.php` - Cache, regenerates on access
- `.htaccess` - Auto-created, regenerates if deleted
- SQLite `-wal` and `-shm` files - Auto-managed by SQLite

### Backup Strategy

```bash
# Filesystem backend
rsync -av --delete /path/to/privatebin/data /backup/location/

# Database backend (MySQL example)
mysqldump -u user -p privatebin > /backup/privatebin-$(date +%Y%m%d).sql

# Configuration
cp /path/to/privatebin/cfg/conf.php /backup/conf.php.backup
```

---

## Production Deployment Checklist

### Pre-Deployment

- [ ] Decide on storage backend (Filesystem, SQLite, MySQL, PostgreSQL, S3, GCS)
- [ ] If using Filesystem: Create `data/` directory with `0700` permissions
- [ ] If using SQLite: Create `data/` directory with `0700` permissions, ensure parent is writable
- [ ] If using MySQL/PostgreSQL: Create database and user account
- [ ] Configure `cfg/conf.php` with chosen backend
- [ ] Set web server user ownership (e.g., `www-data`, `apache`, `nobody`)

### Filesystem Permissions (for Filesystem or SQLite)

```bash
# After deployment
chown -R www-data:www-data /var/www/privatebin/data
chmod 0700 /var/www/privatebin/data
chmod 0644 /var/www/privatebin/data/.htaccess  # if exists
chmod 0700 /var/www/privatebin/cfg              # protect config

# Verify
ls -ld /var/www/privatebin/data
# Expected: drwx------ www-data www-data
```

### SELinux Policy (if enabled)

```bash
# Allow httpd to write to data directory
semanage fcontext -a -t httpd_sys_rw_content_t "/var/www/privatebin/data(/.*)?"
restorecon -Rv /var/www/privatebin/data
```

### AppArmor Policy (if enabled)

```bash
# Add to /etc/apparmor.d/usr.sbin.apache2 (or similar)
/var/www/privatebin/data/ rw,
/var/www/privatebin/data/** rw,
```

### Docker / Container Deployment

```dockerfile
# Ensure runtime permissions
RUN mkdir -p /var/www/privatebin/data && \
    chown -R www-data:www-data /var/www/privatebin && \
    chmod 0700 /var/www/privatebin/data

# Volume mount for persistence
VOLUME ["/var/www/privatebin/data"]
```

### Systemd tmpfiles.d (Optional)

```ini
# /etc/tmpfiles.d/privatebin.conf
d /var/www/privatebin/data 0700 www-data www-data
```

---

## Common Permission Issues & Solutions

### Issue: "Error while trying to store data to the filesystem"

```
Cause: data/ directory not writable by web server
Solution:
  chown www-data:www-data /var/www/privatebin/data
  chmod 0700 /var/www/privatebin/data
```

### Issue: "failed to store the server salt"

```
Cause: data/ directory exists but not writable
Solution:
  # Check permissions
  ls -ld /var/www/privatebin/data
  # Should be drwx------ www-data www-data
```

### Issue: SQLite "database is locked" errors

```
Cause: SELinux/AppArmor blocking write access, or incorrect permissions
Solution:
  # Check SQLite WAL file
  ls -la /var/www/privatebin/data/db.sq3*
  
  # Should see:
  # -rw------- db.sq3
  # -rw------- db.sq3-wal
  # -rw------- db.sq3-shm
  
  # Fix permissions
  chmod 0600 /var/www/privatebin/data/db.sq3*
```

### Issue: "failed to store the traffic limiter"

```
Cause: traffic_limiter.php not writable
Solution:
  # Check file permissions
  ls -la /var/www/privatebin/data/traffic_limiter.php
  
  # Should be owned by www-data with 0700 or 0644
  chmod 0644 /var/www/privatebin/data/traffic_limiter.php
  chown www-data:www-data /var/www/privatebin/data/traffic_limiter.php
```

---

## External Storage Backends

### S3 Storage Backend

```ini
[model]
class = S3Storage
[model_options]
region = "eu-central-1"
version = "latest"
bucket = "my-bucket"
accesskey = "access_key_id"
secretkey = "secret_access_key"
```

**Local Filesystem Impact:** ZERO writes required  
**Credentials:** Stored in `cfg/conf.php` (MUST be read-protected)  
**Backup Strategy:** Use S3 bucket versioning and cross-region replication

### Google Cloud Storage Backend

```ini
[model]
class = GoogleCloudStorage
[model_options]
bucket = "my-private-bin"
prefix = "pastes"
uniformacl = false
```

**Local Filesystem Impact:** ZERO writes required  
**Credentials:** Via GOOGLE_APPLICATION_CREDENTIALS environment variable  
**Backup Strategy:** Use GCS bucket versioning

---

## Summary Table: Writable Paths by Backend

| Path | Filesystem | SQLite | MySQL/PgSQL | S3/GCS |
|------|:----------:|:------:|:-----------:|:------:|
| `data/` | ✓ RW | ✓ RW | ✗ None | ✗ None |
| `data/salt.php` | ✓ RW | ✓ RW | ✗ None | ✗ None |
| `data/traffic_limiter.php` | ✓ RW | ✓ RW | ✗ None | ✗ None |
| `data/purge_limiter.php` | ✓ RW | ✓ RW | ✗ None | ✗ None |
| `data/.htaccess` | ✓ RW | ✓ RW | ✗ None | ✗ None |
| `data/db.sq3` | ✗ None | ✓ RW | ✗ None | ✗ None |
| Remote DB | ✗ None | ✗ None | ✓ RW | ✗ None |
| S3/GCS | ✗ None | ✗ None | ✗ None | ✓ RW |
| `cfg/conf.php` | RO | RO | RO | RO |
| `lib/`, `js/`, `tpl/` | RO | RO | RO | RO |

---

## Security Recommendations

1. **Principle of Least Privilege:** Use `0700` for `data/` to prevent group/world access
2. **Web Server Isolation:** Run PHP-FPM or Apache under dedicated unprivileged user
3. **Configuration Protection:** Ensure `cfg/conf.php` is readable only by web server (`0640`)
4. **Database Credentials:** Store DB credentials in `cfg/conf.php` and restrict file permissions
5. **Backup Protection:** Backups include sensitive data (encryption keys, pastes); store securely
6. **Monitoring:** Monitor `data/` for unexpected writes or permission changes
7. **Logging:** Configure PHP error logging to syslog or secure log file (not web-accessible)

---

## References

- [PrivateBin Configuration Wiki](https://github.com/PrivateBin/PrivateBin/wiki/Configuration)
- [PrivateBin Installation Guide](https://github.com/PrivateBin/PrivateBin/wiki/Installation)
- [SQLite File Format](https://www.sqlite.org/fileformat.html)
- [PHP File Permissions](https://www.php.net/manual/en/function.chmod.php)

---

**Document Version:** 1.0  
**PrivateBin Version:** 2.0.5+  
**Last Updated:** August 2026
