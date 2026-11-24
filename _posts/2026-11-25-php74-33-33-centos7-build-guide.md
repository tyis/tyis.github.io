---
title: "PHP 7.4.33-33をCentOS 7でビルドする ― -31から変わった2つの罠（ICU 78要件とlibpq）"
date: 2026-11-25 02:00:00 +0900
categories: [セキュリティ, サーバー運用, PHP]
tags: [PHP, 脆弱性, CVE-2026-129289, CVE-2026-129496, CentOS 7, Remi, RPMビルド, サーバー運用]
description: "2026年11月24日にRemiが公開したPHP 7.4.33-33は、11月公開の脆弱性バックポートを含む最新リビジョンですが、CentOS 7（el7）向けバイナリは提供されません。前回の-31ビルド手順をそのまま使うと、ICU要件の引き上げ（>= 78）とlibpq-develへの変更で止まります。本記事では、その差分と対処、検証結果をまとめます。"
excerpt: "-31の手順書どおりに進めると、-33では2か所で止まります。ICUのバージョン要件が >= 78 に上がったことと、BuildRequiresがlibpq-develに変わったことです。バージョン番号に依存しないsedとPGDGリポジトリで解決した実記録を公開します。"
published: true
comments: true
---

2026年11月24日、RemiリポジトリがPHP 7.4.33の新しいリビジョン **-33** を公開しました。11月19日に公表された脆弱性群（PHP-FPMのリクエストスマグリング CVE-2026-129289、Timezoneキャッシュのuse-after-free CVE-2026-129496、PHP-CGIの境界外読み書き CVE-2026-129538、PHP-FPMステータス画面のXSS CVE-2026-129292など）へのバックポートは-32で入り、-33はその上にlibpqでのリビルドを重ねたリビジョンです。

しかし前回の-31と同様、**CentOS 7（el7）向けのバイナリは提供されません。** el7でこのリビジョンに到達するには、RemiのSRPMから自前でビルドするしかありません。

当社では-33をCentOS 7でビルドし、検証まで終えました。ビルドスクリプトと成果物（RPM一式）はGitHubで公開しています。

- リポジトリ: [tyis/php74-el7-build](https://github.com/tyis/php74-el7-build)
- 成果物: [Releases v7.4.33-33](https://github.com/tyis/php74-el7-build/releases/tag/v7.4.33-33)

本記事では、[-31のビルド記録](https://blog.tyis.co.jp/%E3%82%BB%E3%82%AD%E3%83%A5%E3%83%AA%E3%83%86%E3%82%A3/%E3%82%B5%E3%83%BC%E3%83%90%E3%83%BC%E9%81%8B%E7%94%A8/php/php74-33-31-centos7-build-guide/)から変わった点を中心に解説しますが、手順自体はこの記事だけで完結するよう、後半に全体の流れを通しで載せています。

## まず結論

| 質問 | 回答 |
|------|------|
| -31の手順はそのまま使えるか | **使えません。** 2か所で止まります（ICU要件、libpq） |
| specの修正は何か所か | **4か所**です（-31の3か所＋libpqの置き換え1か所） |
| ICUのsedはどう変えるか | バージョン番号を固定しない正規表現にします |
| libpq-develはどこにあるか | **el7のどのリポジトリにもありません。** PGDGの `libpq5-devel` が実体です |
| ビルドしたRPMは検証済みか | クリーンなCentOS 7.9へのインストール、テスト結果まで確認済みです（後述） |

## 罠1：ICUの要件が >= 74 から >= 78 に上がった

-31のspecでは、ICUのBuildRequiresは次の形でした。

```text
BuildRequires: pkgconfig(icu-i18n) >= 74
```

-33ではこれが **>= 78** に上がっています（3行とも）。前回記事のsedは `>= 74` を文字列として固定していたため、-33ではマッチせず、要件が残ったまま `yum-builddep` が `No Package found for pkgconfig(icu-i18n) >= 78` で止まります。

対処は、バージョン番号に依存しない形に変えることです。

```bash
# 悪い例（-31用。-33ではマッチしない）
sed -i -E 's|(pkgconfig\(icu-[a-z0-9-]+\))[[:space:]]*>= 74|\1|' php74.spec

# 汎用版（-31 / -32 / -33 のいずれにも効く）
sed -i -E 's|(pkgconfig\(icu-[a-z0-9-]+\))[[:space:]]*>=[[:space:]]*[0-9.]+|\1|' php74.spec
```

el7のシステムICUは50.2であり、要件を満たせない点は-31と同じです。今後のリビジョンでもバージョン番号だけが上がる可能性が高いため、汎用版にしておくことをお勧めします。

## 罠2：BuildRequiresが postgresql-devel から libpq-devel に変わった

-33のspecでは、PostgreSQL開発パッケージの指定が次のように変わりました。

```text
-31: BuildRequires: postgresql-devel
-33: BuildRequires: libpq-devel
```

ところが、**`libpq-devel` という名前のパッケージは、CentOS 7の標準リポジトリにも、EPELにも、Remiにも存在しません。** EL8以降では標準の名前ですが、el7では提供されていないのです。

実体はPostgreSQL公式の **PGDGリポジトリ** にある `libpq5-devel`（libpq 15/16系）です。PGDGリポジトリを追加し、spec側の名前を置き換えます。

```bash
# PGDG（EL7向け PostgreSQL 15）のリポジトリを追加
cat > /etc/yum.repos.d/pgdg.repo << 'EOF'
[pgdg15]
name=PostgreSQL 15 for RHEL 7
baseurl=https://download.postgresql.org/pub/repos/yum/15/redhat/rhel-7-x86_64/
enabled=1
gpgcheck=0
[pgdg-common]
name=PostgreSQL common for RHEL 7
baseurl=https://download.postgresql.org/pub/repos/yum/common/redhat/rhel-7-x86_64/
enabled=1
gpgcheck=0
EOF

# specの置き換え
sed -i 's|^BuildRequires: libpq-devel|BuildRequires: libpq5-devel|' php74.spec
```

なお、PGDGの `libpq5-devel` は `Provides: libpq-devel` を持たないため、リポジトリを追加するだけでは依存解決できません。specの置き換えが必須です。

### 実行時の注意：pgsql拡張にはPGDGのlibpqが必要

この変更は実行時にも影響します。-33のpgsql拡張はlibpq 15/16系でビルドされるため、CentOS 7標準の `postgresql-libs`（9.2）のlibpqでは、次のエラーでロードできません。

```text
PHP Warning: PHP Startup: Unable to load dynamic library 'pgsql'
(... /usr/lib64/php/modules/pgsql.so: undefined symbol: lo_truncate64)
```

当社のクリーン環境検証でも、標準構成のままではこの警告が出ることを確認しました。php-pgsqlを使う環境では、PGDGの `libpq5` へ切り替えてください。

```bash
yum swap postgresql-libs libpq5
```

切り替え後はpgsql / pdo_pgsqlが正常に読み込まれることを確認済みです（pgsqlを使わないモジュールには影響ありません）。

## specの修正は合計4か所

-33での修正点は次の4か所です（diff全文はGitHubリポジトリの `php74.spec.diff` に置いています）。

| 変更箇所 | 変更内容 | 備考 |
|----------|----------|------|
| GDの扱い | `%global with_libgd 1 → 0` | -31から継続 |
| oniguruma | `oniguruma5php-devel` → `oniguruma-devel` | -31から継続（EPEL） |
| ICUのバージョン要件 | `>= 78` を削除（3行） | **汎用sedに変更** |
| PostgreSQL | `libpq-devel` → `libpq5-devel` | **-33で新規追加** |

## ビルド手順（全体の流れ）

前回記事を読まなくても、この手順だけでビルドできます。同じ内容をスクリプト化したものがGitHubリポジトリの `build.sh` です。

### 1. 環境を用意する

ビルドはCentOS 7の中で行います。Dockerがあれば、ホストを汚さずに始められます。

```bash
# ホスト側
docker run -it --name php74build centos:7
```

コンテナ内（root）で、まずリポジトリを整えます。CentOS 7はEOLのため標準リポジトリはvaultへ、EPELはアーカイブへ向けます。Sources系の定義が残っていると依存解決で失敗するため外しておきます。あわせて、罠2で説明したPGDGリポジトリもここで追加します。

```bash
rm -f /etc/yum.repos.d/CentOS-Sources.repo /etc/yum.repos.d/CentOS-CR.repo \
      /etc/yum.repos.d/CentOS-Media.repo /etc/yum.repos.d/CentOS-fasttrack.repo \
      /etc/yum.repos.d/CentOS-Debuginfo.repo /etc/yum.repos.d/CentOS-x86_64-kernel.repo
sed -i -e 's|^mirrorlist=|#mirrorlist=|g' \
       -e 's|^#baseurl=http://mirror.centos.org/centos/$releasever|baseurl=https://vault.centos.org/7.9.2009|g' \
       /etc/yum.repos.d/CentOS-Base.repo
yum clean all

yum -y install https://archives.fedoraproject.org/pub/archive/epel/7/x86_64/Packages/e/epel-release-7-14.noarch.rpm \
               https://rpms.remirepo.net/enterprise/remi-release-7.rpm

# epel.repoのbaseurlをアーカイブへ（metalinkは無効化）、remi-safeはbaseurlを有効化
sed -i -e 's|^metalink=|#metalink=|g' \
       -e 's|^#baseurl=http://download.example/pub/epel/7/$basearch|baseurl=https://archives.fedoraproject.org/pub/archive/epel/7/$basearch|g' \
       /etc/yum.repos.d/epel.repo
sed -i -e '/^mirrorlist=/d' \
       -e 's|^#baseurl=http://rpms.remirepo.net/enterprise/7/safe/$basearch/|baseurl=https://rpms.remirepo.net/enterprise/7/safe/$basearch/|' \
       /etc/yum.repos.d/remi-safe.repo

# PGDG（libpq5-develの取得元）
cat > /etc/yum.repos.d/pgdg.repo << 'EOF'
[pgdg15]
name=PostgreSQL 15 for RHEL 7
baseurl=https://download.postgresql.org/pub/repos/yum/15/redhat/rhel-7-x86_64/
enabled=1
gpgcheck=0
[pgdg-common]
name=PostgreSQL common for RHEL 7
baseurl=https://download.postgresql.org/pub/repos/yum/common/redhat/rhel-7-x86_64/
enabled=1
gpgcheck=0
EOF

yum -y install rpm-build yum-utils cpio gcc gcc-c++ make curl
```

### 2. SRPMは `rpm -i` せず、`rpm2cpio` で展開する

RemiのSRPMはRich Dependenciesを使っており、el7のrpmではインストールできません。中身だけを取り出します。

```bash
mkdir -p ~/rpmbuild/{BUILD,RPMS,SOURCES,SPECS,SRPMS} ~/work
cd ~/work
curl -LO https://rpms.remirepo.net/SRPMS/php-7.4.33-33.remi.src.rpm
rpm2cpio php-7.4.33-33.remi.src.rpm | cpio -idm
cp php74.spec ~/rpmbuild/SPECS/
# spec以外の展開物（tarボール、パッチ、設定ファイル一式）をSOURCESへ
for f in *; do
  case "$f" in *.src.rpm|php74.spec) ;; *) mv "$f" ~/rpmbuild/SOURCES/ ;; esac
done
```

### 3. specを修正する（4か所）

理由は前半の「罠1」「罠2」と表のとおりです。コマンドは次の4つです。

```bash
cd ~/rpmbuild/SPECS
sed -i 's|^%global with_libgd   1|%global with_libgd   0|' php74.spec
sed -i 's|^BuildRequires: oniguruma5php-devel|BuildRequires: oniguruma-devel|' php74.spec
sed -i 's|^BuildRequires: libpq-devel|BuildRequires: libpq5-devel|' php74.spec
sed -i -E 's|(pkgconfig\(icu-[a-z0-9-]+\))[[:space:]]*>=[[:space:]]*[0-9.]+|\1|' php74.spec
```

### 4. 依存解決とビルド

```bash
yum-builddep -y ~/rpmbuild/SPECS/php74.spec
rpmbuild -ba ~/rpmbuild/SPECS/php74.spec
```

`rpmbuild` はCLI/FPM/CGI/Apache/ZTSの各バリアントを順にコンパイルし、最後にテストスイート（`%check`）まで実行します。マシン性能に大きく左右されるため、余裕を持った時間枠で実行してください。

### 5. 成果物

RPMは `~/rpmbuild/RPMS/x86_64/` に、SRPMは `~/rpmbuild/SRPMS/` に出力されます。Dockerの場合はホスト側から取り出せます。

```bash
docker cp php74build:/root/rpmbuild/RPMS ./RPMS
```

## ビルドと検証の結果

GitHubリポジトリの `build.sh` で、Dockerの `centos:7` コンテナ（またはCentOS 7環境）からそのまま実行できます。環境準備から含めた手順はリポジトリのREADMEにまとめました。

当社環境での実績は次のとおりです。

- 成果物：x86_64 RPM **35個**（約71MB）＋ SRPM
- ビルド所要時間：約83分（小規模VMでの実績。マシン性能に大きく左右されます）
- テストスイート（`%check`）：総数15,967件／実行9,158件／**パス9,116件**／失敗1件／スキップ6,809件
  - 失敗した1件は `disk_free_space`（実行環境依存のテスト）で、ビルド自体の不具合ではありません
- クリーンなCentOS 7.9環境へのインストール検証：
  - `php -v` → `PHP 7.4.33 (cli) (built: Nov 24 2026 16:18:51) ( NTS )`
  - 読み込みモジュール70個（pgsql、pdo_pgsql、intl、mbstring、gdを含む）
  - `php-fpm -t` → `configuration file /etc/php-fpm.conf test is successful`
  - RPMのchangelogに、11月公開分のCVE番号（CVE-2026-129538、CVE-2026-129496、CVE-2026-129289、CVE-2026-129292など）が含まれることを確認

## この対応の位置づけ

-33に更新しても、PHP 7.4がEOLである事実は変わりません。11月分の脆弱性には公式修正自体が8.2以降にしか提供されておらず、7.4向けはRemiのバックポートに依存する形です。自前ビルドは「移行までの時間を安全に稼ぐ」ための手段であり、最終的にはサポート中のOS・PHPへの移行計画とセットで実施することをお勧めします。

11月公開の脆弱性そのものの内容については、[関連記事](https://blog.tyis.co.jp/%E3%82%BB%E3%82%AD%E3%83%A5%E3%83%AA%E3%83%86%E3%82%A3/%E3%82%B5%E3%83%BC%E3%83%90%E3%83%BC%E9%81%8B%E7%94%A8/php/php74-november-vulnerabilities-guide/)で整理しています。

---

PHP・サーバーの脆弱性対応や、サポート終了環境からの移行でお困りの場合は、[東洋インターネットサービス株式会社](https://www.tyis.co.jp/)までご相談ください。現行環境の調査から、緩和策の適用、移行計画の策定まで対応いたします。
