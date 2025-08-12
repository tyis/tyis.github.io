---
title: "Rocky Linux 9でのAPM(Apache, PHP, MariaDB)インストール方法"
date: 2025-01-13 10:00:00 +0900
categories: [Linux, RockyLinux9, Apache, PHP, MySQL, APM]
published: true
comments: true
permalink: /rockylinux9-install-apm/
---

## 1. システムパッケージの更新
まず、システムパッケージを最新の状態に更新します。

```sh
sudo dnf update -y
```

## 2. Apache(Apache HTTP Server)のインストール
Apache Webサーバーをインストールして有効化します。

```sh
sudo dnf install -y httpd
sudo systemctl enable --now httpd
```

ファイアウォールでHTTPおよびHTTPSトラフィックを許可します。

```sh
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

Webサーバーのステータスを確認します。

```sh
sudo systemctl status httpd
```

## 3. MariaDB(MySQL)データベースサーバーのインストール
MariaDBをインストールして有効化します。

```sh
sudo dnf install -y mariadb-server
sudo systemctl enable --now mariadb
```

セキュリティ設定を行います。

```sh
sudo mysql_secure_installation
```

インストールプロセスで以下のように設定することを推奨します。
- ルートパスワードの設定
- 匿名ユーザーの削除
- リモートルートログインの無効化
- テストデータベースの削除
- 権限テーブルのリフレッシュ

MariaDBのステータスを確認します。

```sh
sudo systemctl status mariadb
```

## 4. PHPのインストール
PHPと必要な拡張モジュールをインストールします。

```sh
sudo dnf install -y php php-mysqlnd php-cli php-common php-gd php-xml php-mbstring
```

インストールされたPHPのバージョンを確認します。

```sh
php -v
```

Apacheを再起動してPHPを適用します。

```sh
sudo systemctl restart httpd
```

## 5. テストページの作成
ApacheのデフォルトWebディレクトリ(`/var/www/html/`)にPHPテストファイルを作成します。

```sh
echo "<?php phpinfo(); ?>" | sudo tee /var/www/html/info.php
```

Webブラウザで `http://サーバー_IP/info.php` にアクセスし、PHP情報ページが正しく表示されることを確認します。

## 6. 追加設定 (オプション)
### SELinux設定の変更
SELinuxが有効な場合、ApacheでPHPを実行できるように設定します。

```sh
sudo setsebool -P httpd_execmem 1
```

### データベースの作成
MariaDBに接続し、新しいデータベースとユーザーを作成できます。

```sh
sudo mysql -u root -p
```

```sql
CREATE DATABASE mydb;
CREATE USER 'myuser'@'localhost' IDENTIFIED BY 'mypassword';
GRANT ALL PRIVILEGES ON mydb.* TO 'myuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

## 7. 結論
これでRocky Linux 9に**APM(Apache, PHP, MariaDB)**環境が正常にインストールされました。PHPアプリケーションをデプロイし、追加のモジュールやセキュリティ設定を適用して最適化できます。

