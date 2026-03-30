---
title: "Kamalを使ってRailsアプリを本番サーバーにデプロイする方法：複数アプリを1台のサーバーで運用"
date: 2026-03-30 18:00:00 +0900
categories: [技術ブログ, デプロイ]
tags: [Kamal, Rails, Docker, PostgreSQL, デプロイ, プロダクション, サーバー運用, kamal-proxy, Ruby]
description: "Kamal 2を使って複数のRailsアプリケーションを1台のサーバーに本番デプロイする手順を解説します。Docker Hubへのイメージ管理、PostgreSQLアクセサリー、kamal-proxyによるマルチアプリ運用まで実践的にまとめました。"
excerpt: "docker-composeやRenderからKamalに移行し、複数のRailsアプリを1台のサーバーで運用する方法を、実際の構築経験をもとに解説します。"
published: true
comments: true
---

# Kamalを使ってRailsアプリを本番サーバーにデプロイする

![Kamal Deploy]({{ site.url }}/assets/images/svg/kamal-deploy.svg)

Rails 8から標準のデプロイツールとなった**Kamal**を使って、複数のRailsアプリケーションを**1台のサーバー**に本番デプロイする方法をまとめます。

従来のdocker-composeやPaaS（Renderなど）からの移行も含め、実際の構築経験をもとに解説します。

## Kamalとは

[Kamal](https://kamal-deploy.org/)は、Dockerコンテナを使ったゼロダウンタイムデプロイツールです。37signals（Basecamp）が開発し、Rails 8からデフォルトのデプロイツールとして採用されました。

### 従来の方法との比較

| 項目 | docker-compose | PaaS (Render等) | **Kamal** |
|------|---------------|-----------------|-----------|
| デプロイ元 | サーバー上で直接操作 | Git push | **ローカルからコマンド1つ** |
| SSL | 手動設定 (nginx等) | 自動 | **自動 (kamal-proxy)** |
| ゼロダウンタイム | 手動対応 | 対応 | **標準対応** |
| 複数アプリ | 手動設定 | アプリごとに課金 | **1台で複数運用可能** |
| コスト | サーバー代のみ | 従量課金 | **サーバー代のみ** |

## 前提条件

- ローカル: macOS、Docker Desktop、Ruby 3.4+
- サーバー: Ubuntu 24.04、Docker インストール済み
- Docker Hubアカウント
- SSHでサーバーにアクセス可能

## 1. Kamalのインストール

RailsプロジェクトのGemfileにKamalが含まれていない場合：

```bash
gem install kamal
```

バージョン確認：

```bash
kamal version
# => 2.11.0
```

また、ローカルにDocker Buildxプラグインが必要です：

```bash
brew install docker-buildx
```

`~/.docker/config.json`にプラグインパスを追加：

```json
{
  "cliPluginsExtraDirs": [
    "/opt/homebrew/lib/docker/cli-plugins"
  ]
}
```

## 2. サーバーの準備

### システムアカウントの作成

アプリごとに専用のシステムアカウントを作成し、Dockerグループに追加します：

```bash
# サーバー上で実行
sudo useradd -r -m -s /bin/bash myapp
sudo usermod -aG docker myapp

# SSH鍵を設定
sudo mkdir -p /home/myapp/.ssh
sudo cp ~/.ssh/authorized_keys /home/myapp/.ssh/
sudo chown -R myapp:myapp /home/myapp/.ssh
sudo chmod 700 /home/myapp/.ssh
```

**ポイント**: アカウントのUIDは**2000以上**を推奨します。PostgreSQLコンテナ内部のpostgresユーザー（UID 999）とのUID衝突を避けるためです。

```bash
sudo usermod -u 2001 myapp
sudo groupmod -g 2001 myapp
sudo chown -R myapp:myapp /home/myapp
```

## 3. Kamalの設定

### config/deploy.yml

```yaml
service: myapp

image: your-dockerhub-user/myapp

servers:
  web:
    - 192.168.x.x

proxy:
  ssl: false  # 内部ネットワークの場合
  host: myapp.example.com

registry:
  username: your-dockerhub-user
  password:
    - KAMAL_REGISTRY_PASSWORD

env:
  secret:
    - RAILS_MASTER_KEY
    - DATABASE_PASSWORD
  clear:
    SOLID_QUEUE_IN_PUMA: true
    DB_HOST: myapp-db

volumes:
  - "myapp_storage:/rails/storage"

asset_path: /rails/public/assets

builder:
  arch: amd64

ssh:
  user: myapp  # 先ほど作成したアカウント

accessories:
  db:
    image: postgres:17
    host: 192.168.x.x
    port: "127.0.0.1:5432:5432"
    env:
      clear:
        POSTGRES_USER: myapp
        POSTGRES_DB: myapp_production
      secret:
        - POSTGRES_PASSWORD
    volumes:
      - myapp_postgres_data:/var/lib/postgresql/data
```

### .kamal/secrets

```bash
KAMAL_REGISTRY_PASSWORD=$KAMAL_REGISTRY_PASSWORD
RAILS_MASTER_KEY=$(cat config/master.key)
DATABASE_PASSWORD=your_secure_password
POSTGRES_PASSWORD=your_secure_password
```

> **重要**: `.kamal/secrets`は必ず`.gitignore`に追加してください。平文のシークレットが含まれるため、Gitにコミットしてはいけません。

```gitignore
# .gitignore に追加
/.kamal/secrets
```

### database.yml（production）

```yaml
production:
  primary: &primary_production
    <<: *default
    database: myapp_production
    username: myapp
    password: <%= ENV["DATABASE_PASSWORD"] %>
    host: <%= ENV.fetch("DB_HOST", "localhost") %>
```

`DB_HOST`のデフォルト値を`localhost`にすることで、ローカル開発とDocker環境の両方に対応できます。

## 4. Dockerビルド時の注意点

### ENV.fetchのエラー回避

Dockerビルド中の`assets:precompile`ステップでは環境変数が存在しません。`ENV.fetch`を使う初期化ファイル（SMTP設定など）は、環境変数の存在チェックを入れてください：

```ruby
# config/initializers/smtp.rb
if ENV["SMTP_HOST"].present?
  Rails.application.config.action_mailer.smtp_settings = {
    address: ENV.fetch("SMTP_HOST"),
    port: ENV.fetch("SMTP_PORT").to_i,
    # ...
  }
end
```

### force_sslの設定

SSL終端をKamal proxyやCloudflareに任せる場合、Railsの`force_ssl`は無効にします：

```ruby
# config/environments/production.rb
config.assume_ssl = false
config.force_ssl = false
```

## 5. デプロイの実行

### 初回デプロイ（setup）

```bash
KAMAL_REGISTRY_PASSWORD=your_token kamal setup
```

`kamal setup`は以下を自動実行します：

1. サーバーにDockerがインストールされているか確認
2. kamal-proxyコンテナを起動
3. Dockerイメージをビルド＆Docker Hubにプッシュ
4. PostgreSQLアクセサリーを起動
5. アプリコンテナを起動＆ヘルスチェック

### 2回目以降のデプロイ

```bash
KAMAL_REGISTRY_PASSWORD=your_token kamal deploy
```

### データベースのシード

```bash
KAMAL_REGISTRY_PASSWORD=your_token kamal app exec "bin/rails db:seed"
```

## 6. 複数アプリの運用

Kamalの**kamal-proxy**は、Hostヘッダーベースでリクエストをルーティングします。nginxは不要です。

```
クライアント → kamal-proxy (ポート80)
                ├─ Host: app1.example.com → app1コンテナ
                ├─ Host: app2.example.com → app2コンテナ
                └─ Host: app3.example.com → app3コンテナ
```

各アプリの`deploy.yml`で異なる`proxy.host`を設定するだけで、1台のサーバーで複数アプリを運用できます。

### ポート番号の管理

複数のPostgreSQLアクセサリーを使う場合、ホストポートを分けます：

| アプリ | 内部ポート | ホストポート |
|--------|-----------|-------------|
| app1-db | 5432 | 5432 |
| app2-db | 5432 | 5433 |
| app3-db | 5432 | 5434 |

アプリコンテナはDockerネットワーク内でコンテナ名（例：`app1-db`）で接続するため、内部ポート5432を使います。ホストポートは外部からの直接接続用です。

## 7. Solid Queue/Cache/Cable の対応

Rails 8のSolid Queue、Solid Cache、Solid Cableを使う場合、別データベースが必要です。

### init-db.sqlで追加データベースを作成

```sql
CREATE DATABASE myapp_production_cache OWNER myapp;
CREATE DATABASE myapp_production_queue OWNER myapp;
CREATE DATABASE myapp_production_cable OWNER myapp;
```

`deploy.yml`のアクセサリー設定で、このファイルをマウントします：

```yaml
accessories:
  db:
    files:
      - config/init-db.sql:/docker-entrypoint-initdb.d/init-db.sql
```

### docker-entrypointの修正

デフォルトの`db:prepare`は主データベースのみ処理します。全データベースを処理するには：

```bash
# bin/docker-entrypoint
if [ "${@: -2:1}" == "./bin/rails" ] && [ "${@: -1:1}" == "server" ]; then
  ./bin/rails runner 'ActiveRecord::Tasks::DatabaseTasks.prepare_all'
fi
```

## 8. サーバー再起動時の自動復旧

Kamalは全コンテナに`restart: unless-stopped`ポリシーを設定します。Dockerサービスが`enabled`であれば、サーバー再起動後に自動的に全コンテナが起動します。

```bash
# 確認
sudo systemctl is-enabled docker
# => enabled
```

## 9. トラブルシューティング

### コンテナのログ確認

```bash
kamal app logs -f
```

### コンテナ内でコマンド実行

```bash
kamal app exec "bin/rails console"
kamal app exec "bin/rails db:migrate"
```

### ヘルスチェック失敗時

デプロイ時に「target failed to become healthy」エラーが出る場合：

1. `kamal app logs`でエラー内容を確認
2. データベース接続エラー → `DB_HOST`環境変数を確認
3. Solid Queueテーブル不足 → `docker-entrypoint`で`prepare_all`を使用

## まとめ

Kamalを使えば、docker-composeの複雑さやPaaSのコストを避けつつ、**ローカルからコマンド1つ**で本番デプロイが可能です。

- **kamal-proxy**がnginxの代わりにリバースプロキシ・SSL・ルーティングを担当
- **アプリごとのシステムアカウント**でセキュリティを確保
- **Docker Hubを経由**してイメージを管理（無料プランではPublicリポジトリを活用）
- **サーバー再起動後も自動復旧**

特に複数のRailsアプリを1台のサーバーで運用したい場合、Kamalは非常に効率的な選択肢です。

---

*この記事は、実際に3つのRailsアプリケーション（CRM、見積システム、ホスティング管理）を1台のUbuntu 24.04サーバーにKamalでデプロイした経験をもとに執筆しました。*
