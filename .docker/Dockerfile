# Local dev image for php-core (nginx + php-fpm under supervisord, served on :80).
# Mirrors heynabo-dk/php-image (the base of the production images). Every external
# input is pinned by digest, so `docker compose build` only ever runs code we chose.
#
# PHP 8.5, not production's 8.4: the dev lock (composer.lock) needs 8.5
# (softcreatr/jsonpath 1.0.0 via codeception/module-rest). The library itself
# requires >=8.3.
#
# Bump: docker buildx imagetools inspect php:8.5-fpm-alpine  (take the top-level Digest)
FROM php:8.5-fpm-alpine@sha256:951fb5b1780219b81d5daffa4cebef8e0077c537f3893d9790a5dfcd66ca334d

COPY --from=mlocati/php-extension-installer:2.11.12@sha256:b6d3fa381b9ba5cf051117c1c601d6a523b590e534bf3d56eb4fbe352949c138 \
     /usr/bin/install-php-extensions /usr/local/bin/

# Same extensions as php-image (linux-headers is needed to build xdebug).
RUN apk add --no-cache linux-headers \
 && install-php-extensions opcache apcu mysqlnd pdo_mysql redis bcmath xdebug

RUN apk add --no-cache nginx supervisor curl bash \
 && chmod a+rw /var/lib/nginx/logs

COPY .docker/nginx.conf       /etc/nginx/nginx.conf
COPY .docker/fpm-pool.conf    /etc/php7/php-fpm.d/www.conf
COPY .docker/supervisord.conf /etc/supervisor/conf.d/supervisord.conf

# Composer from its official image instead of "latest-stable" over HTTP.
COPY --from=composer:2.11.1@sha256:8de2184c6de224f4689e8f80ed5f5edaf0030b6b8b7839a22a2bcb6722ac87d0 /usr/bin/composer /usr/local/bin/composer

WORKDIR /var/www
EXPOSE 80
CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/conf.d/supervisord.conf"]
