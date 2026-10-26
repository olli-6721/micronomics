ARG PHP_VERSION=8.5.12
ARG NGINX_TAG=stable-alpine

FROM php:${PHP_VERSION}-fpm-alpine AS php_base

WORKDIR /srv/app

COPY --from=mlocati/php-extension-installer:2 /usr/bin/install-php-extensions /usr/local/bin/
RUN install-php-extensions intl pdo_pgsql zip apcu amqp

COPY --from=composer:2 /usr/bin/composer /usr/bin/composer
ENV COMPOSER_HOME=/tmp/composer

COPY docker/symfony/php.ini $PHP_INI_DIR/conf.d/zz-app.ini

FROM php_base AS php_dev

ENV APP_ENV=dev \
    XDEBUG_MODE=off

RUN cp "$PHP_INI_DIR/php.ini-development" "$PHP_INI_DIR/php.ini" \
 && install-php-extensions xdebug

FROM php_base AS php_prod

ARG SERVICE_DIR
ENV APP_ENV=prod \
    APP_DEBUG=0

RUN cp "$PHP_INI_DIR/php.ini-production" "$PHP_INI_DIR/php.ini" \
 && printf "opcache.validate_timestamps=0\nopcache.preload=/srv/app/config/preload.php\nopcache.preload_user=www-data\n" \
    > "$PHP_INI_DIR/conf.d/zz-prod.ini"

COPY ${SERVICE_DIR}/composer.* ${SERVICE_DIR}/symfony.* ./
RUN composer install --no-dev --no-scripts --no-autoloader --prefer-dist --no-progress --no-interaction

COPY ${SERVICE_DIR}/ ./
RUN mkdir -p var \
 && composer dump-autoload --classmap-authoritative --no-dev \
 && composer dump-env prod \
 && composer run-script --no-dev post-install-cmd \
 && chown -R www-data:www-data var

USER www-data

FROM nginx:${NGINX_TAG} AS web

ARG SERVICE_DIR
ENV PHP_UPSTREAM=127.0.0.1:9000

COPY docker/symfony/nginx.conf.template /etc/nginx/templates/default.conf.template
COPY ${SERVICE_DIR}/public /srv/app/public
