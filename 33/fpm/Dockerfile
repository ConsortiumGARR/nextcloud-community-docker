# DO NOT EDIT: created by update.sh from Dockerfile-debian.template
FROM php:8.4-fpm-trixie

# entrypoint.sh and cron.sh dependencies
RUN set -ex; \
    \
    apt-get update; \
    apt-get install -y --no-install-recommends \
        busybox-static \
        bzip2 \
        libldap-common \
        libmagickcore-7.q16-10-extra \
    ; \
    apt-get dist-clean; \
    \
    mkdir -p /var/spool/cron/crontabs; \
    echo '*/5 * * * * php -f /var/www/html/cron.php' > /var/spool/cron/crontabs/www-data

# install the PHP extensions we need
# see https://docs.nextcloud.com/server/stable/admin_manual/installation/source_installation.html
ENV PHP_MEMORY_LIMIT 512M
ENV PHP_UPLOAD_LIMIT 512M
ENV PHP_OPCACHE_MEMORY_CONSUMPTION 128
RUN set -ex; \
    \
    savedAptMark="$(apt-mark showmanual)"; \
    \
    apt-get update; \
    apt-get install -y --no-install-recommends \
        libcurl4-openssl-dev \
        libevent-dev \
        libfreetype6-dev \
        libgmp-dev \
        libicu-dev \
        libjpeg-dev \
        libldap2-dev \
        liblz4-dev \
        libmagickwand-dev \
        libmemcached-dev \
        libpng-dev \
        libpq-dev \
        libwebp-dev \
        libxml2-dev \
        libzip-dev \
    ; \
    \
    debMultiarch="$(dpkg-architecture --query DEB_BUILD_MULTIARCH)"; \
    docker-php-ext-configure ftp --with-ftp-ssl; \
    docker-php-ext-configure gd --with-freetype --with-jpeg --with-webp; \
    docker-php-ext-configure ldap --with-libdir="lib/$debMultiarch"; \
    docker-php-ext-install -j "$(nproc)" \
        bcmath \
        exif \
        ftp \
        gd \
        gmp \
        intl \
        ldap \
        pcntl \
        pdo_mysql \
        pdo_pgsql \
        sysvsem \
        zip \
    ; \
    \
# pecl will claim success even if one install fails, so we need to perform each install separately
    pecl install APCu-5.1.28; \
    pecl install igbinary-3.2.17RC1; \
    pecl install imagick-3.8.1; \
    pecl install --configureoptions 'enable-memcached-igbinary="yes"' \
        memcached-3.4.0; \
    pecl install --configureoptions 'enable-redis-igbinary="yes" enable-redis-zstd="yes" enable-redis-lz4="yes"' \
        redis-6.3.0; \
    \
    docker-php-ext-enable \
        apcu \
        igbinary \
        imagick \
        memcached \
        redis \
    ; \
    rm -r /tmp/pear; \
    \
# reset apt-mark's "manual" list so that "purge --auto-remove" will remove all build dependencies
    apt-mark auto '.*' > /dev/null; \
    apt-mark manual $savedAptMark; \
    ldd "$(php -r 'echo ini_get("extension_dir");')"/*.so \
        | awk '/=>/ { so = $(NF-1); if (index(so, "/usr/local/") == 1) { next }; gsub("^/(usr/)?", "", so); print so }' \
        | sort -u \
        | xargs -rt dpkg-query --search \
# https://manpages.debian.org/trixie/dpkg/dpkg-query.1.en.html#S (we ignore diversions and it'll be really unusual for more than one package to provide any given .so file)
        | awk 'sub(":$", "", $1) { print $1 }' \
        | sort -u \
        | xargs -rt apt-mark manual; \
    \
    apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false; \
    apt-get dist-clean

# set recommended PHP.ini settings
# see https://docs.nextcloud.com/server/latest/admin_manual/installation/server_tuning.html#enable-php-opcache
RUN { \
        echo 'opcache.enable=1'; \
        echo 'opcache.interned_strings_buffer=32'; \
        echo 'opcache.max_accelerated_files=10000'; \
        echo 'opcache.memory_consumption=${PHP_OPCACHE_MEMORY_CONSUMPTION}'; \
        echo 'opcache.save_comments=1'; \
        echo 'opcache.revalidate_freq=60'; \
        echo 'opcache.jit=1255'; \
        echo 'opcache.jit_buffer_size=8M'; \
    } > "${PHP_INI_DIR}/conf.d/opcache-recommended.ini"; \
    \
    echo 'apc.enable_cli=1' >> "${PHP_INI_DIR}/conf.d/docker-php-ext-apcu.ini"; \
    \
    { \
        echo 'apc.serializer=igbinary'; \
        echo 'session.serialize_handler=igbinary'; \
    } >> "${PHP_INI_DIR}/conf.d/docker-php-ext-igbinary.ini"; \
    \
    { \
        echo 'memory_limit=${PHP_MEMORY_LIMIT}'; \
        echo 'upload_max_filesize=${PHP_UPLOAD_LIMIT}'; \
        echo 'post_max_size=${PHP_UPLOAD_LIMIT}'; \
    } > "${PHP_INI_DIR}/conf.d/nextcloud.ini"; \
    \
    mkdir /var/www/data; \
    mkdir -p /docker-entrypoint-hooks.d/pre-installation \
             /docker-entrypoint-hooks.d/post-installation \
             /docker-entrypoint-hooks.d/pre-upgrade \
             /docker-entrypoint-hooks.d/post-upgrade \
             /docker-entrypoint-hooks.d/before-starting; \
    chown -R www-data:root /var/www; \
    chmod -R g=u /var/www



ENV NEXTCLOUD_VERSION 33.0.10

# When set, the enterprise archive is downloaded from this URL instead of the
# community tarball.
ARG ENTERPRISE_ARCHIVE_URL=""

RUN set -ex; \
    fetchDeps=" \
        gnupg \
        dirmngr \
        unzip \
    "; \
    apt-get update; \
    apt-get install -y --no-install-recommends $fetchDeps; \
    \
    if [ -n "$ENTERPRISE_ARCHIVE_URL" ]; then \
        archive=nextcloud.zip; \
        archiveUrl="$ENTERPRISE_ARCHIVE_URL"; \
        ascUrl="$ENTERPRISE_ARCHIVE_URL.asc"; \
    else \
        archive=nextcloud.tar.bz2; \
        archiveUrl="https://github.com/nextcloud-releases/server/releases/download/v33.0.10/nextcloud-33.0.10.tar.bz2"; \
        ascUrl="https://github.com/nextcloud-releases/server/releases/download/v33.0.10/nextcloud-33.0.10.tar.bz2.asc"; \
    fi; \
    curl -fsSL -o "$archive" "$archiveUrl"; \
    curl -fsSL -o "$archive.asc" "$ascUrl"; \
    export GNUPGHOME="$(mktemp -d)"; \
# gpg key from https://nextcloud.com/nextcloud.asc
    gpg --batch --keyserver keyserver.ubuntu.com --recv-keys 28806A878AE423A28372792ED75899B9A724937A; \
    gpg --batch --verify "$archive.asc" "$archive"; \
    case "$archive" in \
        *.zip) unzip -q "$archive" -d /usr/src/ ;; \
        *) tar -xjf "$archive" -C /usr/src/ ;; \
    esac; \
    gpgconf --kill all; \
    rm "$archive" "$archive.asc"; \
    rm -rf "$GNUPGHOME" /usr/src/nextcloud/updater; \
    mkdir -p /usr/src/nextcloud/data; \
    mkdir -p /usr/src/nextcloud/custom_apps; \
    mkdir -p /usr/src/nextcloud/themes; \
    chmod +x /usr/src/nextcloud/occ; \
# The code is baked into the image, so there is no rsync at startup. This happens in
# the same layer as the extraction, otherwise the image would carry the code twice.
# Only the default config files are kept in /usr/src/nextcloud, as the reference the
# entrypoint compares the persistent config volume against.
    cp -a /usr/src/nextcloud/. /var/www/html/; \
    find /usr/src/nextcloud -mindepth 1 -maxdepth 1 ! -name config -exec rm -rf '{}' +; \
    chown -R www-data:root /var/www/html; \
    chmod -R g=u /var/www/html; \
    \
    apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false $fetchDeps; \
    apt-get dist-clean

COPY *.sh /
COPY config/* /usr/src/nextcloud/config/
COPY --chown=www-data:root config/* /var/www/html/config/

VOLUME /var/www/html/config
VOLUME /var/www/html/data
VOLUME /var/www/html/custom_apps
VOLUME /var/www/html/themes

ENTRYPOINT ["/entrypoint.sh"]
CMD ["php-fpm"]
