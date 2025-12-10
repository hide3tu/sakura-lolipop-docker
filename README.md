# レンタルサーバー再現 Docker環境

さくらインターネット・ロリポップのレンタルサーバー環境を再現したDocker構成です。
Claude Codeでの自立型開発を想定し、パス構造を統一しています。

## 環境構成

| サービス | コンテナ名 | HTTP | HTTPS | ベースOS |
|---------|-----------|------|-------|---------|
| さくら環境 | sakura | http://localhost:8081 | https://localhost:8443 | AlmaLinux 9 |
| ロリポ環境 | lolipop | http://localhost:8082 | https://localhost:8444 | Ubuntu 24.04 |
| MySQL 8.0 | rental-mysql | localhost:3306 | - | - |
| phpMyAdmin | rental-phpmyadmin | http://localhost:8080 | - | - |
| Mailpit | rental-mailpit | http://localhost:8025 | - | - |

## 共通パス構造（両環境）

```
/home/user/www/    ← ドキュメントルート
/home/user/log/    ← ログ出力先
/home/user/tmp/    ← 一時ファイル
```

## ディレクトリ構成

```
docker/
├── docker-compose.yml
├── sakura/
│   └── Dockerfile
├── lolipop/
│   └── Dockerfile
├── mysql/
│   └── my.cnf
└── projects/
    ├── sakura/
    │   ├── www/      ← ここにPHPファイルを置く
    │   └── log/
    └── lolipop/
        ├── www/      ← ここにPHPファイルを置く
        └── log/
```

## 使い方

### 初回セットアップ

```bash
# プロジェクトディレクトリ作成
mkdir -p projects/sakura/{www,log}
mkdir -p projects/lolipop/{www,log}

# ビルド＆起動
docker-compose up -d --build
```

### 動作確認

```bash
# さくら環境にテストファイル作成
echo '<?php phpinfo();' > projects/sakura/www/index.php

# ロリポ環境にテストファイル作成
echo '<?php phpinfo();' > projects/lolipop/www/index.php
```

ブラウザで確認：
- さくら: http://localhost:8081
- ロリポ: http://localhost:8082

### よく使うコマンド

```bash
# 起動
docker-compose up -d

# 停止
docker-compose down

# ログ確認
docker-compose logs -f sakura
docker-compose logs -f lolipop

# コンテナに入る
docker exec -it sakura bash
docker exec -it lolipop bash

# 再ビルド（Dockerfile変更時）
docker-compose up -d --build
```

## データベース接続情報

PHPから接続する場合：

```php
<?php
$host = 'mysql';  // コンテナ内からはホスト名で接続
$port = 3306;
$dbname = 'development';
$user = 'devuser';
$pass = 'devpass';

$pdo = new PDO(
    "mysql:host={$host};port={$port};dbname={$dbname};charset=utf8mb4",
    $user,
    $pass
);
```

外部ツール（Sequel Pro等）から接続する場合：
- Host: localhost (または 127.0.0.1)
- Port: 3306
- User: devuser
- Password: devpass

## Claude Code連携

両環境でパス構造が統一されているため、Claude Codeへの指示はシンプルに：

```
ドキュメントルート: /home/user/www/
ログ出力先: /home/user/log/
DB接続先: mysql:3306
```

環境の違いを意識せず、同じパス指定で開発できます。

## インストール済みPHP拡張

両環境共通：
- mbstring
- gd
- curl
- intl
- mysqlnd / pdo_mysql
- xml
- zip
- opcache
- bcmath
- soap
- iconv

## 制限事項（本番環境に合わせて）

以下は意図的に**インストールしていません**：
- Composer
- Node.js / npm
- その他ビルドツール

レンタルサーバーで使えない機能に依存したコードを書かないための制約です。

## 注意点

- さくらの本番はFreeBSDですが、DockerはLinuxベースのためAlmaLinuxで代替しています
- Apache/PHPの動作は互換性がありますが、OS固有の挙動は異なる場合があります
- 本番デプロイ前に実環境でのテストを推奨します

## SSL（HTTPS）

両環境でオレオレ証明書（自己署名）によるHTTPSが使えます。

- さくら: https://localhost:8443
- ロリポ: https://localhost:8444

**初回アクセス時の警告について**

ブラウザで「この接続ではプライバシーが保護されません」等の警告が出ますが、
開発環境なので「詳細設定」→「localhostにアクセスする」で進めてください。

**WordPressでSSLを使う場合**

`wp-config.php` に以下を追加：
```php
define('FORCE_SSL_ADMIN', true);
if ($_SERVER['HTTP_X_FORWARDED_PROTO'] == 'https') {
    $_SERVER['HTTPS'] = 'on';
}
```

または `WP_HOME` / `WP_SITEURL` をhttpsで指定：
```php
define('WP_HOME', 'https://localhost:8443');
define('WP_SITEURL', 'https://localhost:8443');
```

## メール送信テスト

両環境で `mail()` / `mb_send_mail()` / `wp_mail()` が使えます。
送信されたメールは Mailpit で確認できます: http://localhost:8025

```php
<?php
// テスト送信
mail('test@example.com', 'テスト件名', 'テスト本文');

// WordPressの場合
wp_mail('test@example.com', 'テスト件名', 'テスト本文');
```

実際には送信されず、Mailpit のWeb UIに届きます（開発中の誤送信防止）。

## ディレクトリごとのphp.ini

さくらインターネットと同様、各ディレクトリに `php.ini` を置くことで設定を上書きできます。

```
projects/sakura/www/
├── index.php
├── php.ini              ← ルート全体に適用
└── admin/
    ├── index.php
    └── php.ini          ← admin/ 以下に適用
```

例：`php.ini`
```ini
memory_limit = 512M
max_execution_time = 60
```

**注意**: 設定変更は最大5分キャッシュされます（`user_ini.cache_ttl = 300`）。
即座に反映させたい場合は php-fpm を再起動してください：

```bash
docker exec sakura supervisorctl restart php-fpm
docker exec lolipop supervisorctl restart php-fpm
```
