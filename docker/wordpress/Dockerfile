FROM wordpress:php8.4-fpm

# Install dependencies for WP-CLI
RUN apt-get update && apt-get install -y \
    less \
    mariadb-client \
    && rm -rf /var/lib/apt/lists/*

# Install WP-CLI
RUN curl -O https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar \
    && chmod +x wp-cli.phar \
    && mv wp-cli.phar /usr/local/bin/wp

# Verify WP-CLI installation
RUN wp --info --allow-root

# Set proper permissions for www-data
RUN chown -R www-data:www-data /var/www/html

# Keep the default entrypoint and command from the base image
