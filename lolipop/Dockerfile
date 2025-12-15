# ロリポップ レンタルサーバー環境再現
# 本番: Linux (Ubuntu) / Apache 2.4.x / PHP 8.3 (CGI/FastCGI)
# 再現: Ubuntu 24.04 / Apache 2.4 + php-fpm / PHP 8.3

FROM ubuntu:24.04

# 非対話モード
ENV DEBIAN_FRONTEND=noninteractive
ENV TZ=Asia/Tokyo

# タイムゾーン設定
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

# 基本パッケージとPPAリポジトリ
RUN apt-get update && \
    apt-get install -y software-properties-common && \
    add-apt-repository -y ppa:ondrej/php && \
    apt-get update

# Apache + PHP-FPM 8.3 + 標準拡張 + SSL
RUN apt-get install -y \
    apache2 \
    openssl \
    php8.3-fpm \
    php8.3-cli \
    php8.3-common \
    php8.3-mbstring \
    php8.3-gd \
    php8.3-curl \
    php8.3-intl \
    php8.3-mysql \
    php8.3-xml \
    php8.3-zip \
    php8.3-opcache \
    php8.3-bcmath \
    php8.3-soap \
    php8.3-imap \
    msmtp \
    msmtp-mta \
    supervisor \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# msmtp設定（Mailpitに転送）
RUN echo 'account default' > /etc/msmtprc && \
    echo 'host mailpit' >> /etc/msmtprc && \
    echo 'port 1025' >> /etc/msmtprc && \
    echo 'from noreply@localhost' >> /etc/msmtprc && \
    echo 'auth off' >> /etc/msmtprc && \
    echo 'tls off' >> /etc/msmtprc && \
    chmod 644 /etc/msmtprc

# ユーザー作成（パス構造をさくらと統一）
RUN useradd -m -s /bin/bash user && \
    mkdir -p /home/user/www /home/user/log /home/user/tmp && \
    chown -R user:user /home/user

# Apache モジュール有効化
RUN a2enmod rewrite && \
    a2enmod headers && \
    a2enmod proxy && \
    a2enmod proxy_fcgi && \
    a2enmod ssl && \
    a2dismod php8.3 2>/dev/null || true

# オレオレSSL証明書生成
RUN mkdir -p /etc/apache2/ssl && \
    openssl req -x509 -nodes -days 3650 -newkey rsa:2048 \
    -keyout /etc/apache2/ssl/server.key \
    -out /etc/apache2/ssl/server.crt \
    -subj "/C=JP/ST=Tokyo/L=Tokyo/O=LocalDev/CN=localhost" \
    -addext "subjectAltName=DNS:localhost,IP:127.0.0.1"

# Apache設定（HTTP）
RUN echo '<VirtualHost *:80>' > /etc/apache2/sites-available/000-default.conf && \
    echo '    DocumentRoot /home/user/www' >> /etc/apache2/sites-available/000-default.conf && \
    echo '    <Directory /home/user/www>' >> /etc/apache2/sites-available/000-default.conf && \
    echo '        Options FollowSymLinks' >> /etc/apache2/sites-available/000-default.conf && \
    echo '        AllowOverride All' >> /etc/apache2/sites-available/000-default.conf && \
    echo '        Require all granted' >> /etc/apache2/sites-available/000-default.conf && \
    echo '    </Directory>' >> /etc/apache2/sites-available/000-default.conf && \
    echo '    DirectoryIndex index.php index.html' >> /etc/apache2/sites-available/000-default.conf && \
    echo '    ErrorLog /home/user/log/error.log' >> /etc/apache2/sites-available/000-default.conf && \
    echo '    CustomLog /home/user/log/access.log combined' >> /etc/apache2/sites-available/000-default.conf && \
    echo '    <FilesMatch \.php$>' >> /etc/apache2/sites-available/000-default.conf && \
    echo '        SetHandler "proxy:fcgi://127.0.0.1:9000"' >> /etc/apache2/sites-available/000-default.conf && \
    echo '    </FilesMatch>' >> /etc/apache2/sites-available/000-default.conf && \
    echo '</VirtualHost>' >> /etc/apache2/sites-available/000-default.conf

# Apache設定（HTTPS）
RUN echo '<VirtualHost *:443>' > /etc/apache2/sites-available/default-ssl.conf && \
    echo '    DocumentRoot /home/user/www' >> /etc/apache2/sites-available/default-ssl.conf && \
    echo '    SSLEngine on' >> /etc/apache2/sites-available/default-ssl.conf && \
    echo '    SSLCertificateFile /etc/apache2/ssl/server.crt' >> /etc/apache2/sites-available/default-ssl.conf && \
    echo '    SSLCertificateKeyFile /etc/apache2/ssl/server.key' >> /etc/apache2/sites-available/default-ssl.conf && \
    echo '    <Directory /home/user/www>' >> /etc/apache2/sites-available/default-ssl.conf && \
    echo '        Options FollowSymLinks' >> /etc/apache2/sites-available/default-ssl.conf && \
    echo '        AllowOverride All' >> /etc/apache2/sites-available/default-ssl.conf && \
    echo '        Require all granted' >> /etc/apache2/sites-available/default-ssl.conf && \
    echo '    </Directory>' >> /etc/apache2/sites-available/default-ssl.conf && \
    echo '    DirectoryIndex index.php index.html' >> /etc/apache2/sites-available/default-ssl.conf && \
    echo '    ErrorLog /home/user/log/ssl_error.log' >> /etc/apache2/sites-available/default-ssl.conf && \
    echo '    <FilesMatch \.php$>' >> /etc/apache2/sites-available/default-ssl.conf && \
    echo '        SetHandler "proxy:fcgi://127.0.0.1:9000"' >> /etc/apache2/sites-available/default-ssl.conf && \
    echo '    </FilesMatch>' >> /etc/apache2/sites-available/default-ssl.conf && \
    echo '</VirtualHost>' >> /etc/apache2/sites-available/default-ssl.conf && \
    a2ensite default-ssl

# PHP-FPM設定（ポート9000でリッスン）
# clear_env = no: Docker環境変数をPHP-FPMワーカーに渡す（DB接続情報など）
RUN sed -i 's|listen = /run/php/php8.3-fpm.sock|listen = 127.0.0.1:9000|g' /etc/php/8.3/fpm/pool.d/www.conf && \
    sed -i 's|user = www-data|user = user|g' /etc/php/8.3/fpm/pool.d/www.conf && \
    sed -i 's|group = www-data|group = user|g' /etc/php/8.3/fpm/pool.d/www.conf && \
    sed -i 's|;clear_env = no|clear_env = no|g' /etc/php/8.3/fpm/pool.d/www.conf

# PHP設定（レンタルサーバー風）
RUN echo "date.timezone = Asia/Tokyo" >> /etc/php/8.3/fpm/php.ini && \
    echo "mbstring.language = Japanese" >> /etc/php/8.3/fpm/php.ini && \
    echo "mbstring.internal_encoding = UTF-8" >> /etc/php/8.3/fpm/php.ini && \
    echo "upload_max_filesize = 50M" >> /etc/php/8.3/fpm/php.ini && \
    echo "post_max_size = 50M" >> /etc/php/8.3/fpm/php.ini && \
    echo "memory_limit = 256M" >> /etc/php/8.3/fpm/php.ini && \
    echo "max_execution_time = 30" >> /etc/php/8.3/fpm/php.ini && \
    echo "sendmail_path = /usr/bin/msmtp -t" >> /etc/php/8.3/fpm/php.ini

# ディレクトリごとのphp.iniを有効化（さくらと統一）
RUN echo "user_ini.filename = php.ini" >> /etc/php/8.3/fpm/php.ini && \
    echo "user_ini.cache_ttl = 300" >> /etc/php/8.3/fpm/php.ini

# Apache実行ユーザー変更
RUN sed -i 's|APACHE_RUN_USER=www-data|APACHE_RUN_USER=user|g' /etc/apache2/envvars && \
    sed -i 's|APACHE_RUN_GROUP=www-data|APACHE_RUN_GROUP=user|g' /etc/apache2/envvars

# PHP-FPM用ディレクトリ作成
RUN mkdir -p /run/php && chown user:user /run/php

# Supervisor設定（Apache + PHP-FPM 同時起動）
RUN echo '[supervisord]' > /etc/supervisor/conf.d/supervisord.conf && \
    echo 'nodaemon=true' >> /etc/supervisor/conf.d/supervisord.conf && \
    echo 'logfile=/dev/null' >> /etc/supervisor/conf.d/supervisord.conf && \
    echo 'logfile_maxbytes=0' >> /etc/supervisor/conf.d/supervisord.conf && \
    echo '' >> /etc/supervisor/conf.d/supervisord.conf && \
    echo '[program:php-fpm]' >> /etc/supervisor/conf.d/supervisord.conf && \
    echo 'command=/usr/sbin/php-fpm8.3 --nodaemonize' >> /etc/supervisor/conf.d/supervisord.conf && \
    echo 'autostart=true' >> /etc/supervisor/conf.d/supervisord.conf && \
    echo 'autorestart=true' >> /etc/supervisor/conf.d/supervisord.conf && \
    echo 'stdout_logfile=/dev/stdout' >> /etc/supervisor/conf.d/supervisord.conf && \
    echo 'stdout_logfile_maxbytes=0' >> /etc/supervisor/conf.d/supervisord.conf && \
    echo 'stderr_logfile=/dev/stderr' >> /etc/supervisor/conf.d/supervisord.conf && \
    echo 'stderr_logfile_maxbytes=0' >> /etc/supervisor/conf.d/supervisord.conf && \
    echo '' >> /etc/supervisor/conf.d/supervisord.conf && \
    echo '[program:apache2]' >> /etc/supervisor/conf.d/supervisord.conf && \
    echo 'command=/usr/sbin/apachectl -D FOREGROUND' >> /etc/supervisor/conf.d/supervisord.conf && \
    echo 'autostart=true' >> /etc/supervisor/conf.d/supervisord.conf && \
    echo 'autorestart=true' >> /etc/supervisor/conf.d/supervisord.conf && \
    echo 'stdout_logfile=/dev/stdout' >> /etc/supervisor/conf.d/supervisord.conf && \
    echo 'stdout_logfile_maxbytes=0' >> /etc/supervisor/conf.d/supervisord.conf && \
    echo 'stderr_logfile=/dev/stderr' >> /etc/supervisor/conf.d/supervisord.conf && \
    echo 'stderr_logfile_maxbytes=0' >> /etc/supervisor/conf.d/supervisord.conf

WORKDIR /home/user/www

EXPOSE 80 443

CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/conf.d/supervisord.conf"]
