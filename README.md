# WordPress Docker Setup

Kompletne środowisko WordPress z nginx, MariaDB, PHP-FPM i WP-CLI.

## Struktura katalogów

```
blog/
├── docker-compose.yml
├── docker/
│   ├── nginx/
│   │   ├── nginx.conf
│   │   └── default.conf
│   ├── php/
│   │   ├── custom.ini
│   │   └── php-fpm.conf
│   └── mariadb/
│       └── my.cnf
└── README.md
```

## Konfiguracja

### Zmień hasła przed uruchomieniem

#### Dla lokalnego deploymentu:

```bash
# Skopiuj plik przykładowy
cp .env.example .env

# Edytuj .env i ustaw hasła
nano .env
```

#### Dla Coolify:

Coolify automatycznie wygeneruje i wstrzyknie zmienne środowiskowe:
- `MYSQL_ROOT_PASSWORD` - hasło root dla MariaDB
- `MYSQL_PASSWORD` - hasło użytkownika WordPress do bazy danych

Nie musisz tworzyć pliku `.env` - Coolify to zrobi za Ciebie.

### Główne ustawienia PHP

- Wersja: PHP 8.4 (najnowsza)
- Memory limit: 512M
- Max execution time: 300s
- Upload max filesize: 512M
- Post max size: 512M
- OPcache włączony
- Timezone: Europe/Warsaw

**Uwaga**: WordPress 6.7+ ma wsparcie beta dla PHP 8.4. Dla maksymalnej stabilności produkcyjnej możesz użyć PHP 8.2 lub 8.3 zmieniając `FROM wordpress:php8.4-fpm` na `FROM wordpress:php8.2-fpm` w `docker/wordpress/Dockerfile`.

### Główne ustawienia MariaDB

- Character set: utf8mb4
- InnoDB buffer pool: 512M
- Max connections: 200
- Max packet: 256M

## Uruchomienie

```bash
# Uruchom wszystkie kontenery
docker-compose up -d

# Sprawdź status
docker-compose ps

# Logi
docker-compose logs -f
```

### Lokalnie
WordPress będzie dostępny pod: http://localhost:8080

### Coolify
Coolify automatycznie skonfiguruje domenę i SSL. Nginx jest wystawiony na porcie 80 (`expose: 80`).

## Instalacja WordPress

### Przez przeglądarkę
Otwórz http://localhost:8080 i przejdź przez instalator.

### Przez WP-CLI

```bash
# Instalacja WordPress
docker-compose exec -u www-data wordpress wp core install \
  --url="http://localhost:8080" \
  --title="Moja strona" \
  --admin_user="admin" \
  --admin_password="admin_password" \
  --admin_email="admin@example.com"

# Zmień permalinki
docker-compose exec -u www-data wordpress wp rewrite structure '/%postname%/'

# Zainstaluj motyw
docker-compose exec -u www-data wordpress wp theme install astra --activate

# Zainstaluj plugin
docker-compose exec -u www-data wordpress wp plugin install contact-form-7 --activate

# Lista użytkowników
docker-compose exec -u www-data wordpress wp user list

# Aktualizacja WordPress
docker-compose exec -u www-data wordpress wp core update
```

## Przydatne komendy WP-CLI

```bash
# Backup bazy danych
docker-compose exec -u www-data wordpress wp db export backup.sql

# Import bazy danych
docker-compose exec -u www-data wordpress wp db import backup.sql

# Wyszukaj i zamień w bazie
docker-compose exec -u www-data wordpress wp search-replace 'oldurl.com' 'newurl.com'

# Flush cache
docker-compose exec -u www-data wordpress wp cache flush

# Regeneruj miniaturki
docker-compose exec -u www-data wordpress wp media regenerate --yes

# Lista pluginów
docker-compose exec -u www-data wordpress wp plugin list

# Lista motywów
docker-compose exec -u www-data wordpress wp theme list
```

## Zarządzanie kontenerami

```bash
# Stop
docker-compose stop

# Start
docker-compose start

# Restart pojedynczego serwisu
docker-compose restart nginx

# Rebuild po zmianach w konfiguracji
docker-compose down
docker-compose up -d --force-recreate

# Usuń wszystko (UWAGA: skasuje dane!)
docker-compose down -v
```

## Dostęp do kontenerów

```bash
# WordPress/PHP (z WP-CLI w tym samym kontenerze)
docker-compose exec wordpress bash

# MariaDB
docker-compose exec db mysql -u wordpress -p

# Nginx
docker-compose exec nginx sh
```

## Backup i restore

### Backup

```bash
# Backup WordPress files
docker run --rm --volumes-from wiedzmin_wordpress \
  -v $(pwd):/backup alpine tar czf /backup/wordpress_files.tar.gz -C /var/www html

# Backup bazy danych
docker-compose exec -u www-data wordpress wp db export - > wordpress_db.sql
```

### Restore

```bash
# Restore plików
docker run --rm --volumes-from wiedzmin_wordpress \
  -v $(pwd):/backup alpine tar xzf /backup/wordpress_files.tar.gz -C /var/www

# Restore bazy
docker-compose exec -T -u www-data wordpress wp
```

## Debugging

```bash
# Logi nginx
docker-compose exec nginx tail -f /var/log/nginx/wordpress-error.log

# Logi PHP
docker-compose exec wordpress tail -f /var/log/php_errors.log

# Logi MariaDB
docker-compose exec db tail -f /var/log/mysql/slow.log

# Testy PHP-FPM
docker-compose exec wordpress php -v
docker-compose exec wordpress php -m  # załadowane moduły
```

## Optymalizacja

### Dla rozwoju
W `docker-compose.yml` ustaw:
```yaml
WORDPRESS_DEBUG: 1
```

### Dla produkcji (Coolify)
- Coolify automatycznie generuje silne hasła
- `WORDPRESS_DEBUG` jest ustawiony na `0` domyślnie
- Coolify automatycznie konfiguruje certyfikat SSL (Let's Encrypt)
- Nginx jest wystawiony przez `expose: 80` - Coolify proxy obsłuży routing
- Regularnie rób backupy (można zautomatyzować w Coolify)

## Troubleshooting

### WordPress nie może zapisać do katalogu
```bash
docker-compose exec wordpress chown -R www-data:www-data /var/www/html
```

### Błędy uprawnień WP-CLI
```bash
docker-compose exec wordpress chown -R www-data:www-data /var/www/html/wp-content
```

### Reset hasła admin
```bash
docker-compose exec -u www-data wordpress wp user update 1 --user_pass=nowe_haslo
```

### Sprawdź konfigurację nginx
```bash
docker-compose exec nginx nginx -t
```
