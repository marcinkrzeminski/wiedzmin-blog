# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Project Overview

This is a complete WordPress Docker environment using nginx, MariaDB 11.2, PHP 8.4-FPM, and WP-CLI. The setup is designed for both local development and production deployment via Coolify.

## Architecture

### Multi-Container Setup
- **nginx (1.25-alpine)**: Reverse proxy handling HTTP requests, serving static files, and security headers
- **wordpress (PHP 8.4-FPM)**: WordPress application layer with WP-CLI pre-installed
- **db (MariaDB 11.2)**: Database server with optimized configuration

### Data Flow
1. nginx receives HTTP requests on port 80 (exposed)
2. Static files served directly by nginx from shared volume
3. PHP requests forwarded to wordpress:9000 via FastCGI
4. WordPress connects to db:3306 for database operations

### Volume Architecture
- `wordpress_data`: Shared between nginx (read-only) and wordpress (read-write) for WordPress files
- `db_data`: Persistent database storage
- `nginx_logs`: Persistent nginx logs
- Configuration files mounted read-only from `docker/` subdirectories

### Key Configuration Files
- `docker/wordpress/Dockerfile`: Custom WordPress image with WP-CLI
- `docker/nginx/default.conf`: nginx server block with WordPress-specific rules
- `docker/php/custom.ini`: PHP runtime configuration (512M memory, 300s timeout, OPcache)
- `docker/php/php-fpm.conf`: PHP-FPM pool settings
- `docker/mariadb/my.cnf`: MariaDB optimizations (512M buffer pool, 200 connections)

## Essential Commands

### Initial Setup
```bash
# Local development - copy and configure environment
cp .env.example .env
nano .env  # Set strong passwords

# Start all containers
docker-compose up -d

# Check status
docker-compose ps
```

### WordPress Installation via WP-CLI
```bash
# Install WordPress core
docker-compose exec -u www-data wordpress wp core install \
  --url="http://localhost:8080" \
  --title="Site Name" \
  --admin_user="admin" \
  --admin_password="secure_password" \
  --admin_email="admin@example.com"

# Configure permalinks
docker-compose exec -u www-data wordpress wp rewrite structure '/%postname%/'

# Install and activate theme
docker-compose exec -u www-data wordpress wp theme install astra --activate

# Install and activate plugin
docker-compose exec -u www-data wordpress wp plugin install contact-form-7 --activate
```

### Database Operations
```bash
# Export database backup
docker-compose exec -u www-data wordpress wp db export backup.sql

# Import database
docker-compose exec -u www-data wordpress wp db import backup.sql

# Search and replace URLs (e.g., after domain change)
docker-compose exec -u www-data wordpress wp search-replace 'oldurl.com' 'newurl.com'

# Access MariaDB CLI
docker-compose exec db mysql -u wordpress -p
```

### Common WordPress Tasks
```bash
# List all plugins with status
docker-compose exec -u www-data wordpress wp plugin list

# List all themes
docker-compose exec -u www-data wordpress wp theme list

# Update WordPress core
docker-compose exec -u www-data wordpress wp core update

# Flush WordPress cache
docker-compose exec -u www-data wordpress wp cache flush

# Regenerate image thumbnails
docker-compose exec -u www-data wordpress wp media regenerate --yes

# Reset admin password
docker-compose exec -u www-data wordpress wp user update 1 --user_pass=new_password
```

### Container Management
```bash
# View logs (all containers)
docker-compose logs -f

# View logs (specific service)
docker-compose logs -f nginx
docker-compose logs -f wordpress

# Restart specific service
docker-compose restart nginx
docker-compose restart wordpress

# Stop all containers (preserves data)
docker-compose stop

# Start stopped containers
docker-compose start

# Rebuild after configuration changes
docker-compose down
docker-compose up -d --force-recreate

# Remove everything including volumes (DESTRUCTIVE)
docker-compose down -v
```

### Debugging
```bash
# Access WordPress container shell
docker-compose exec wordpress bash

# Access nginx container shell
docker-compose exec nginx sh

# View nginx error logs
docker-compose exec nginx tail -f /var/log/nginx/wordpress-error.log

# View PHP error logs
docker-compose exec wordpress tail -f /var/log/php_errors.log

# View MariaDB slow query log
docker-compose exec db tail -f /var/log/mysql/slow.log

# Check PHP version and modules
docker-compose exec wordpress php -v
docker-compose exec wordpress php -m

# Test nginx configuration
docker-compose exec nginx nginx -t
```

### Backup and Restore
```bash
# Backup WordPress files
docker run --rm --volumes-from wiedzmin_wordpress \
  -v $(pwd):/backup alpine tar czf /backup/wordpress_files.tar.gz -C /var/www html

# Backup database
docker-compose exec -u www-data wordpress wp db export - > wordpress_db.sql

# Restore WordPress files
docker run --rm --volumes-from wiedzmin_wordpress \
  -v $(pwd):/backup alpine tar xzf /backup/wordpress_files.tar.gz -C /var/www

# Restore database
docker-compose exec -T -u www-data wordpress wp db import < wordpress_db.sql
```

### Permission Fixes
```bash
# Fix WordPress file permissions
docker-compose exec wordpress chown -R www-data:www-data /var/www/html

# Fix wp-content permissions (for plugin/theme uploads)
docker-compose exec wordpress chown -R www-data:www-data /var/www/html/wp-content
```

## Important Notes

### Environment Variables
- **Local**: Create `.env` from `.env.example` and set `MYSQL_ROOT_PASSWORD` and `MYSQL_PASSWORD`
- **Coolify**: Environment variables auto-generated; no `.env` file needed

### PHP Version
WordPress 6.7+ has beta support for PHP 8.4. For production stability, consider PHP 8.2 or 8.3 by changing the base image in `docker/wordpress/Dockerfile` from `wordpress:php8.4-fpm` to `wordpress:php8.2-fpm` or `wordpress:php8.3-fpm`.

### WP-CLI User Context
Always run WP-CLI commands as `www-data` user using `-u www-data` flag to avoid permission issues.

### Networking
- **Local**: WordPress accessible at http://localhost:8080
- **Coolify**: nginx exposed on port 80; Coolify handles SSL/TLS and domain routing

### Security Configurations
nginx blocks access to:
- `.ht` files
- `.git` directories
- `xmlrpc.php` (prevents brute force attacks)
- `wp-config.php`
- PHP files in uploads directory
- readme/license/changelog files

### Development vs Production
For development, enable debug mode in `docker-compose.yml`:
```yaml
WORDPRESS_DEBUG: 1
```

For production (default), debug is disabled (`WORDPRESS_DEBUG: 0`).

## Performance Tuning

### PHP Settings (docker/php/custom.ini)
- Memory limit: 512M
- Max execution time: 300s
- Upload limit: 512M
- OPcache enabled with 128M memory

### MariaDB Settings (docker/mariadb/my.cnf)
- InnoDB buffer pool: 512M
- Max connections: 200
- Max packet size: 256M

### nginx Static File Caching
Static assets (images, CSS, JS, fonts) cached for 365 days with immutable flag.
