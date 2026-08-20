# PrivateBin Writable Paths - Quick Reference

## TL;DR: Essential Writable Paths

### Default Filesystem Backend

```
✓ data/                         MUST be 0700, writable by web server
  ├── salt.php                  Auto-created on first run
  ├── traffic_limiter.php       Continuously updated
  ├── purge_limiter.php         Periodically updated
  ├── .htaccess                 Auto-created for Apache
  └── [2-char]/[2-char]/        Paste storage tree (auto-created)
```

### SQLite Backend  

```
✓ data/                         MUST be 0700
  └── db.sq3                    Database file (0600)
      db.sq3-wal                Write-ahead log (auto-managed)
      db.sq3-shm                Shared memory (auto-managed)
```

### Remote Backend (MySQL/PostgreSQL/S3/GCS)

```
✓ No local filesystem writes required
  (Configuration stored in cfg/conf.php)
```

---

## Quick Setup

### Filesystem Backend (Default)

```bash
mkdir -p /var/www/privatebin/data
chown www-data:www-data /var/www/privatebin/data
chmod 0700 /var/www/privatebin/data
```

### SQLite Backend

```bash
mkdir -p /var/www/privatebin/data
chown www-data:www-data /var/www/privatebin/data
chmod 0700 /var/www/privatebin/data
# Database auto-creates on first request
```

### MySQL Backend

```ini
; cfg/conf.php
[model]
class = Database
[model_options]
dsn = "mysql:host=db.example.com;dbname=privatebin;charset=UTF8"
tbl = "privatebin_"
usr = "privatebin"
pwd = "password"
opt[12] = true
```

### S3 Backend

```ini
; cfg/conf.php
[model]
class = S3Storage
[model_options]
region = "eu-central-1"
bucket = "privatebin-bucket"
accesskey = "AKIA..."
secretkey = "..."
```

---

## Permission Requirements

| Path | Owner | Permissions | Purpose |
|------|-------|-------------|---------|
| `data/` | www-data | 0700 | Paste & metadata storage |
| `data/salt.php` | www-data | auto | Server salt (created 1x) |
| `data/traffic_limiter.php` | www-data | auto | Rate limit cache |
| `data/purge_limiter.php` | www-data | auto | Expiration sweep cache |
| `data/db.sq3` | www-data | 0600 | SQLite database (if used) |
| `cfg/conf.php` | www-data | 0640 | Configuration (read-only) |

---

## Common Issues

| Issue | Cause | Fix |
|-------|-------|-----|
| "Error while trying to store data" | `data/` not writable | `chmod 0700 data/` |
| "failed to store the server salt" | Permission denied | `chown www-data:www-data data/` |
| "database is locked" (SQLite) | Permission issue or SELinux | `chmod 0600 data/db.sq3*` |
| "failed to store the traffic limiter" | Wrong file permissions | `chmod 0644 data/traffic_limiter.php` |

---

## What NOT to Worry About

```
✓ vendor/ directory         - Dependencies, read-only
✓ lib/ directory            - Source code, read-only
✓ js/ directory             - Client scripts, read-only
✓ tpl/ directory            - Templates, read-only
✓ css/ directory            - Stylesheets, read-only
✓ i18n/ directory           - Translations, read-only
✓ PHP error logging         - Configured outside app (php.ini)
✓ PHP sessions              - Not used by PrivateBin
✓ Cache headers             - Generated in-memory, not written to disk
```

---

## Scaling Notes

For high-traffic deployments:

**Default structure:**
```
data/AA/BB/XXXXXXXXXXXXXXXX.php
```

**Deeper structure** (edit Filesystem.php):
```
data/AA/BB/CC/XXXXXXXXXXXXXXXX.php  (4 levels instead of 3)
```

This reduces files per directory and improves filesystem performance.

---

## Backup

```bash
# What to backup
- data/ directory
- cfg/conf.php
- (Optional: database if remote)

# What NOT to backup (auto-regenerates)
- traffic_limiter.php
- purge_limiter.php
- .htaccess
- db.sq3-wal, db.sq3-shm (SQLite temporary files)
```

---

## Docker Example

```dockerfile
FROM php:8.1-apache

# ... setup code ...

# Create data directory with proper permissions
RUN mkdir -p /var/www/html/data && \
    chown -R www-data:www-data /var/www/html && \
    chmod 0700 /var/www/html/data

# Make data volume for persistence
VOLUME ["/var/www/html/data"]

EXPOSE 80
CMD ["apache2-foreground"]
```

---

## SELinux (if enabled)

```bash
semanage fcontext -a -t httpd_sys_rw_content_t "/var/www/privatebin/data(/.*)?"
restorecon -Rv /var/www/privatebin/data
```

---

## One-Line Deployment Test

```bash
curl -X POST http://localhost/api/create -d '{"data":"Hello"}' && \
  test -f /var/www/privatebin/data/*//*/*.php && \
  echo "✓ Writable paths working"
```
