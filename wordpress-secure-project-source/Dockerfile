FROM wordpress:latest

# Instala paquetes requeridos y la extensión PHP Redis para conectar WordPress con Redis.
RUN apt-get update \
 && apt-get install -y --no-install-recommends libzip-dev unzip \
 && pecl install redis \
 && docker-php-ext-enable redis \
 && rm -rf /var/lib/apt/lists/*
