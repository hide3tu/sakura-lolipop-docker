# さくらインターネット レンタルサーバー環境再現
# 本番: FreeBSD 13.0 / Apache 2.4.65 / PHP 8.3 (CGI/FastCGI)
# 再現: AlmaLinux 9 / Apache 2.4 + php-fpm / PHP 8.3

FROM almalinux:9

# タイムゾーン設定
ENV TZ=Asia/Tokyo
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

# 基本パッケージとEPEL/Remiリポジトリ
RUN dnf -y update && \
    dnf -y install epel-release && \
    dnf -y install https://rpms.remirepo.net/enterprise/remi-release-9.rpm && \
    dnf -y module reset php && \
    dnf -y module enable php:remi-8.3

# Apache + PHP-FPM 8.3 + 標準拡張 + SSL
RUN dnf -y install \
    httpd \
    mod_ssl \
    openssl \
    php-fpm \
    php-cli \
    php-common \
    php-mbstring \
    php-gd \
    php-curl \
    php-intl \
    php-mysqlnd \
    php-pdo \
    php-xml \
    php-zip \
    php-json \
    php-opcache \
    php-bcmath \
    php-soap \
    php-iconv \
    msmtp \
    msmtp-mta \
    supervisor \
    && dnf clean all

# msmtp設定（Mailpitに転送）
RUN echo 'account default' > /etc/msmtprc && \
    echo 'host mailpit' >> /etc/msmtprc && \
    echo 'port 1025' >> /etc/msmtprc && \
    echo 'from noreply@localhost' >> /etc/msmtprc && \
    echo 'auth off' >> /etc/msmtprc && \
    echo 'tls off' >> /etc/msmtprc && \
    chmod 644 /etc/msmtprc

# ユーザー作成（さくらの構成に合わせる）
RUN useradd -m -s /bin/bash user && \
    mkdir -p /home/user/www /home/user/log /home/user/tmp && \
    chown -R user:user /home/user

# Apache設定（FastCGI用）
RUN sed -i 's|DocumentRoot "/var/www/html"|DocumentRoot "/home/user/www"|g' /etc/httpd/conf/httpd.conf && \
    sed -i 's|<Directory "/var/www/html">|<Directory "/home/user/www">|g' /etc/httpd/conf/httpd.conf && \
    sed -i 's|AllowOverride None|AllowOverride All|g' /etc/httpd/conf/httpd.conf && \
    sed -i 's|DirectoryIndex index.html|DirectoryIndex index.php index.html|g' /etc/httpd/conf/httpd.conf

# .htaccess と mod_rewrite 有効化
RUN echo '<Directory "/home/user/www">' >> /etc/httpd/conf/httpd.conf && \
    echo '    Options FollowSymLinks' >> /etc/httpd/conf/httpd.conf && \
    echo '    AllowOverride All' >> /etc/httpd/conf/httpd.conf && \
    echo '    Require all granted' >> /etc/httpd/conf/httpd.conf && \
    echo '</Directory>' >> /etc/httpd/conf/httpd.conf

# Apache + PHP-FPM 連携設定
RUN echo '<FilesMatch \.php$>' > /etc/httpd/conf.d/php-fpm.conf && \
    echo '    SetHandler "proxy:fcgi://127.0.0.1:9000"' >> /etc/httpd/conf.d/php-fpm.conf && \
    echo '</FilesMatch>' >> /etc/httpd/conf.d/php-fpm.conf

# エラーログ設定
RUN sed -i 's|ErrorLog "logs/error_log"|ErrorLog "/home/user/log/error_log"|g' /etc/httpd/conf/httpd.conf

# オレオレSSL証明書生成
RUN mkdir -p /etc/httpd/ssl && \
    openssl req -x509 -nodes -days 3650 -newkey rsa:2048 \
    -keyout /etc/httpd/ssl/server.key \
    -out /etc/httpd/ssl/server.crt \
    -subj "/C=JP/ST=Tokyo/L=Tokyo/O=LocalDev/CN=localhost" \
    -addext "subjectAltName=DNS:localhost,IP:127.0.0.1"

# SSL VirtualHost設定
RUN echo '<VirtualHost *:443>' > /etc/httpd/conf.d/ssl-vhost.conf && \
    echo '    DocumentRoot "/home/user/www"' >> /etc/httpd/conf.d/ssl-vhost.conf && \
    echo '    SSLEngine on' >> /etc/httpd/conf.d/ssl-vhost.conf && \
    echo '    SSLCertificateFile /etc/httpd/ssl/server.crt' >> /etc/httpd/conf.d/ssl-vhost.conf && \
    echo '    SSLCertificateKeyFile /etc/httpd/ssl/server.key' >> /etc/httpd/conf.d/ssl-vhost.conf && \
    echo '    <Directory "/home/user/www">' >> /etc/httpd/conf.d/ssl-vhost.conf && \
    echo '        Options FollowSymLinks' >> /etc/httpd/conf.d/ssl-vhost.conf && \
    echo '        AllowOverride All' >> /etc/httpd/conf.d/ssl-vhost.conf && \
    echo '        Require all granted' >> /etc/httpd/conf.d/ssl-vhost.conf && \
    echo '    </Directory>' >> /etc/httpd/conf.d/ssl-vhost.conf && \
    echo '    DirectoryIndex index.php index.html' >> /etc/httpd/conf.d/ssl-vhost.conf && \
    echo '    <FilesMatch \.php$>' >> /etc/httpd/conf.d/ssl-vhost.conf && \
    echo '        SetHandler "proxy:fcgi://127.0.0.1:9000"' >> /etc/httpd/conf.d/ssl-vhost.conf && \
    echo '    </FilesMatch>' >> /etc/httpd/conf.d/ssl-vhost.conf && \
    echo '    ErrorLog "/home/user/log/ssl_error_log"' >> /etc/httpd/conf.d/ssl-vhost.conf && \
    echo '</VirtualHost>' >> /etc/httpd/conf.d/ssl-vhost.conf

# Apache実行ユーザー変更
RUN sed -i 's|User apache|User user|g' /etc/httpd/conf/httpd.conf && \
    sed -i 's|Group apache|Group user|g' /etc/httpd/conf/httpd.conf

# PHP-FPM設定
RUN sed -i 's|user = apache|user = user|g' /etc/php-fpm.d/www.conf && \
    sed -i 's|group = apache|group = user|g' /etc/php-fpm.d/www.conf && \
    sed -i 's|listen = /run/php-fpm/www.sock|listen = 127.0.0.1:9000|g' /etc/php-fpm.d/www.conf

# PHP設定（レンタルサーバー風）
RUN echo "date.timezone = Asia/Tokyo" >> /etc/php.ini && \
    echo "mbstring.language = Japanese" >> /etc/php.ini && \
    echo "mbstring.internal_encoding = UTF-8" >> /etc/php.ini && \
    echo "upload_max_filesize = 50M" >> /etc/php.ini && \
    echo "post_max_size = 50M" >> /etc/php.ini && \
    echo "memory_limit = 256M" >> /etc/php.ini && \
    echo "max_execution_time = 30" >> /etc/php.ini && \
    echo "sendmail_path = /usr/bin/msmtp -t" >> /etc/php.ini

# さくら風：ディレクトリごとのphp.iniを有効化
# user_ini.filename = "php.ini" でphp.iniファイル名が使える
RUN echo "user_ini.filename = php.ini" >> /etc/php.ini && \
    echo "user_ini.cache_ttl = 300" >> /etc/php.ini

# Supervisor設定（Apache + PHP-FPM 同時起動）
RUN echo '[supervisord]' > /etc/supervisord.conf && \
    echo 'nodaemon=true' >> /etc/supervisord.conf && \
    echo 'logfile=/dev/null' >> /etc/supervisord.conf && \
    echo 'logfile_maxbytes=0' >> /etc/supervisord.conf && \
    echo '' >> /etc/supervisord.conf && \
    echo '[program:php-fpm]' >> /etc/supervisord.conf && \
    echo 'command=/usr/sbin/php-fpm --nodaemonize' >> /etc/supervisord.conf && \
    echo 'autostart=true' >> /etc/supervisord.conf && \
    echo 'autorestart=true' >> /etc/supervisord.conf && \
    echo 'stdout_logfile=/dev/stdout' >> /etc/supervisord.conf && \
    echo 'stdout_logfile_maxbytes=0' >> /etc/supervisord.conf && \
    echo 'stderr_logfile=/dev/stderr' >> /etc/supervisord.conf && \
    echo 'stderr_logfile_maxbytes=0' >> /etc/supervisord.conf && \
    echo '' >> /etc/supervisord.conf && \
    echo '[program:httpd]' >> /etc/supervisord.conf && \
    echo 'command=/usr/sbin/httpd -D FOREGROUND' >> /etc/supervisord.conf && \
    echo 'autostart=true' >> /etc/supervisord.conf && \
    echo 'autorestart=true' >> /etc/supervisord.conf && \
    echo 'stdout_logfile=/dev/stdout' >> /etc/supervisord.conf && \
    echo 'stdout_logfile_maxbytes=0' >> /etc/supervisord.conf && \
    echo 'stderr_logfile=/dev/stderr' >> /etc/supervisord.conf && \
    echo 'stderr_logfile_maxbytes=0' >> /etc/supervisord.conf

WORKDIR /home/user/www

EXPOSE 80 443

CMD ["/usr/bin/supervisord", "-c", "/etc/supervisord.conf"]
