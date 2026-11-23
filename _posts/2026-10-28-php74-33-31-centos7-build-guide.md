---
title: "CentOS 7にPHP 7.4.33-31の配布パッケージは存在しない ― Remi SRPMからel7向けRPMを自前ビルドした記録"
date: 2026-10-28 10:00:00 +0900
categories: [セキュリティ, サーバー運用, PHP]
tags: [PHP, 脆弱性, PHP2Shell, CVE-2026-109618, CVE-2026-109619, CentOS 7, Remi, RPMビルド, サーバー運用]
description: "PHP2Shell（CVE-2026-109618 / CVE-2026-109619）の緩和修正を含むPHP 7.4.33-31は、CentOS 7（el7）向けのバイナリがRemiから提供されていません。本記事では、RemiのSRPMからel7向けRPMを自前ビルドした手順と、つまずいたポイント、検証方法、適用前の注意点を実務記録としてまとめます。"
excerpt: "7.4.33-31への更新が急務でも、CentOS 7では `yum update` できません。el7向けバイナリ自体が存在しないためです。SRPMの展開、specの3行修正、クリーン環境での検証まで、当社で実際に行ったビルド手順を公開します。"
published: true
comments: true
---

2026年10月に公表されたPHPの重大な脆弱性 **PHP2Shell（CVE-2026-109618 / CVE-2026-109619）** への対応として、RemiリポジトリはPHP 7.4向けに **7.4.33-31**（緩和修正を含むリビジョン）を公開しました。

ところが、**CentOS 7（el7）向けのバイナリはこのリビジョンが提供されていません。** el7向けのRemi PHP 7.4パッケージは古いリビジョンで止まっており、`yum update` では7.4.33-31に到達できません。OSのサポートも終了しているCentOS 7では、そもそも「配布元が直してくれる」前提が崩れています。

本記事は、CentOS 7上でPHP 7.4系を運用している環境向けに、当社が実際に行った **Remi SRPMからのel7向けRPM自前ビルド** の手順と注意点をまとめた実務記録です。

## まず結論

| 質問 | 回答 |
|------|------|
| CentOS 7で7.4.33-31にyum更新できるか | **できません。** el7向けバイナリが提供されていません |
| SRPMからビルドできるか | **できます。** RemiのSRPM（php-7.4.33-31.remi.src.rpm）が原典になります |
| specの修正は多いか | **3か所だけです。** 依存関係の置き換えと要件の緩和に限られます |
| ビルドしたRPMをそのまま本番に入れてよいか | **いけません。** クリーン環境でのインストール検証と、ロールバック手段の準備が前提です |
| これでPHP2Shell対応は完了か | **緩和修正です。** 完全な修正版ではありません。根本対応はサポート中OS・PHPへの移行です |

## なぜel7向けバイナリが存在しないのか

RemiリポジトリのPHP 7.4は、現行OS向けには継続してリビジョンが更新されていますが、**el7向けは保守の対象から外れており**、PHP2Shellの緩和修正を含む-31は公開されませんでした。el7で入手できる最終リビジョンには、CVE-2026-109618 / CVE-2026-109619の修正は含まれていません。

つまりCentOS 7環境では、次の3択になります。

1. 脆弱性を残したまま運用を続ける（推奨しません）
2. OSごとサポート中の環境へ移行する（根本対応ですが、時間と検証が必要です）
3. **SRPMから自前でビルドし、当面の緩和を入れる**（本記事の方法）

移行までの時間を確保するための現実的な選択肢として、3を実施しました。

## ビルド手順

### 1. ビルド環境はDockerのcentos:7コンテナで用意する

ホストOSを汚さず、再現性を確保するため、ビルドはDockerコンテナ（`centos:7`）で行いました。ビルド所要時間は約9分です。

CentOS 7はEOLのため、標準リポジトリは **vault.centos.org** を参照する必要があります。ただしvaultは応答が不安定で403が返ることがあり、当社では国内ミラー（RIKENミラー）に切り替えました。また、`CentOS-Sources.repo` が有効だと `yum-builddep` がソースリポジトリ側で403になり失敗するため、**Sources系のリポジトリ定義は外しておきます。**

具体的なセットアップは次のとおりです。EPELも7はアーカイブ行きになっているため、アーカイブ側から入れます。

```bash
# CentOS 7標準リポジトリをRIKENミラーのvaultへ向ける
rm -f /etc/yum.repos.d/CentOS-Sources.repo
sed -i -e 's|^mirrorlist=|#mirrorlist=|g' \
       -e 's|^#baseurl=http://mirror.centos.org/centos/$releasever|baseurl=https://ftp.riken.jp/Linux/centos-vault/7.9.2009|g' \
       /etc/yum.repos.d/CentOS-Base.repo

# EPEL 7（アーカイブ）とRemiのリリースRPM
yum -y install https://archives.fedoraproject.org/pub/archive/epel/7/x86_64/Packages/e/epel-release-7-14.noarch.rpm
yum -y install https://rpms.remirepo.net/enterprise/remi-release-7.rpm

# epel.repoのbaseurlをアーカイブへ向ける（metalinkは無効化）
sed -i -e 's|^metalink=|#metalink=|g' \
       -e 's|^#baseurl=http://download.example/pub/epel/7/$basearch|baseurl=https://archives.fedoraproject.org/pub/archive/epel/7/$basearch|g' \
       /etc/yum.repos.d/epel.repo

# ビルド道具
yum -y install rpm-build yum-utils cpio gcc gcc-c++ make
```

### 2. SRPMは `rpm -i` せず、`rpm2cpio` で展開する

ここが最初のつまずきポイントです。

RemiのSRPMは **Rich Dependencies（リッチ依存関係）** を使っており、el7のrpmはこれを解釈できないため、通常の `rpm -i php-7.4.33-31.remi.src.rpm` が失敗します。

回避策は単純で、SRPMをインストールせずに中身だけ取り出します。SRPM自体はRemiのサイトから取得できます。

```bash
curl -LO https://rpms.remirepo.net/SRPMS/php-7.4.33-31.remi.src.rpm

mkdir -p ~/rpmbuild/{BUILD,RPMS,SOURCES,SPECS,SRPMS}
rpm2cpio php-7.4.33-31.remi.src.rpm | cpio -idmv
cp php74.spec ~/rpmbuild/SPECS/
# spec以外の展開物（tarボール、パッチ、設定ファイル一式）をSOURCESへ
for f in *; do
  case "$f" in *.src.rpm|php74.spec) ;; *) mv "$f" ~/rpmbuild/SOURCES/ ;; esac
done
```

取り出した `.spec` ファイルを直接 `rpmbuild` に渡す形でビルドします。

### 3. specの修正は3か所だけ

el7のライブラリ事情に合わせて、specを最小限だけ変更しました。**変更点をdiffで保存しておくこと**が、後の再ビルドや説明責任のために重要です。

| 変更箇所 | 変更内容 | 理由 |
|----------|----------|------|
| GDの扱い | `with_libgd` を無効化（バンドル版GDを使用） | el7のlibgdではRemi specの要求を満たせないため |
| oniguruma | `oniguruma5php-devel` → `oniguruma-devel`（EPELの6.8.2） | el7向けにRemi版の開発パッケージが存在しないため。実行時は `libonig.so.5` が必要です |
| ICUのバージョン要件 | 「ICU 74以上」の要求を削除 | el7のシステムICUは50.2であり、要件を満たせないため |

実際のspecでは次の3行（群）です。`sed` でそのまま置換できます。

```bash
cd ~/rpmbuild/SPECS
# 102行目付近: %global with_libgd   1 → 0
sed -i 's|^%global with_libgd   1|%global with_libgd   0|' php74.spec
# BuildRequires: oniguruma5php-devel → oniguruma-devel
sed -i 's|^BuildRequires: oniguruma5php-devel|BuildRequires: oniguruma-devel|' php74.spec
# pkgconfig(icu-i18n / icu-io / icu-uc) の「>= 74」を削除（3行）
sed -i -E 's|(pkgconfig\(icu-[a-z0-9-]+\))[[:space:]]*>= 74|\1|' php74.spec
```

なお、ICUの要件を残したまま `yum-builddep` を実行すると `No Package found for pkgconfig(icu-i18n) >= 74` で依存解決が止まります。el7ではこの3行の緩和が実質的に必須です。

それ以外はRemiのspecそのままです。**修正をこの3点に抑えることで、「Remiの-31と同一のソース・同一のパッチセットである」ことを説明できる**のが、この方法の利点です。-31の緩和パッチ（`php-7.4-cve-2026-109618.patch`）自体はSRPMに含まれています。

### 4. 依存解決とビルド

```bash
yum-builddep -y ~/rpmbuild/SPECS/php74.spec
rpmbuild -ba ~/rpmbuild/SPECS/php74.spec
```

`rpmbuild` はCLI/FPM/CGI/Apache/ZTSの各バリアントを順にコンパイルし、最後にPHPのテストスイート（`%check`）まで実行します。マシンの性能に大きく左右されますが、余裕を持った時間枠で実行してください。

### 5. ビルド成果物

- PHP関連RPM **37個**（php、php-cli、php-fpm、php-mbstring、php-intl、php-gd、php-opcache など）
- SHA256SUMS（成果物の検証用）
- specの変更diff、ビルドログ一式

## 検証：クリーンなCentOS 7コンテナで入れ直す

ビルドが通ることと、使えることは別です。成果物は **まっさらなCentOS 7コンテナ** にインストールし直して検証しました。

- `php -v` が 7.4.33 を返すこと
- mbstring（oniguruma連携）、intl（ICU 50.2）、gd、OPcacheが読み込めること
- `php-fpm -t` が通ること
- RPMのchangelogに **CVE-2026-109618 / CVE-2026-109619** の修正が含まれること

検証時の出力例です。

```text
$ php -v
PHP 7.4.33 (cli) (built: Nov 23 2026 17:32:21) ( NTS )
    with Zend OPcache v7.4.33

$ php-fpm -t
NOTICE: configuration file /etc/php-fpm.conf test is successful

$ rpm -q --changelog php-common | grep CVE-2026-10961
  CVE-2026-109618, CVE-2026-109619
```

特にchangelogの確認は重要です。EOL系統へのバックポートではバージョン番号が変わらないため、**「-31相当である」ことの根拠はパッケージ側の記録で示す**必要があります。

### 再現検証について（2026年11月追記）

本記事の公開にあたり、上記の手順をまっさらなCentOS 7.9環境で最初から再現し直し、全手順がそのまま通ることを確認しました。再現時の結果は次のとおりです。

- 成果物：php関連RPM 35個（x86_64、debuginfo含む）
- `%check` のテストスイート：9,069件パス、失敗10件（0.1%）。失敗はいずれも `session rfc1867` 系、`disk_free_space`、`memory_limit` といった実行環境依存のテストで、ビルド自体の不具合ではありませんでした
- クリーン環境へのインストール、`php -v`、主要モジュール、`php-fpm -t`、changelogのCVE記載まで再確認済みです

## 適用前に必ず決めること

### ロールバック手段

自前ビルド品には配布元の戻し先がありません。適用前に、

- 既存のPHP関連ファイルとRPMデータベースのバックアップ
- 旧リビジョンのパッケージへの復帰手順（入手可能な最終リビジョンへのダウングレード可否を含む）

を用意し、「元の状態に完全に戻せるか」を先に確認してください。スナップショットが使える環境なら、それが最も確実です。

### PECL拡張の7.4用意

PHP 7.3等から7.4へ上げる場合、PECL拡張（apcu、mcrypt、zipなど）は7.4向けのものが別途必要です。本体のビルドと合わせて、利用中の拡張の入手可否を事前に確認してください。

### `session.upload_progress` の設定

7.4.33-31の緩和パッチはコード側でupload progressのコールバックを無効化しますが、当社では安全側に倒し、設定側でも `session.upload_progress.enabled = Off` を明示したiniを併せて配置しています（二重の安全策）。

## この対応の位置づけ

改めて強調しますが、7.4.33-31は **PHP2Shellに対する緩和修正** であり、完全な修正版ではありません。また、その後に公表される新しい脆弱性が-31に自動で含まれることもありません。2026年11月19日に公表された脆弱性（CVE-2026-129289ほか）と7.4.33-31の関係については、[関連記事](https://blog.tyis.co.jp/%E3%82%BB%E3%82%AD%E3%83%A5%E3%83%AA%E3%83%86%E3%82%A3/%E3%82%B5%E3%83%BC%E3%83%90%E3%83%BC%E9%81%8B%E7%94%A8/php/php74-november-vulnerabilities-guide/)で整理しています。

自前ビルドは「移行までの時間を安全に稼ぐ」ための手段です。CentOS 7 + PHP 7.4という構成自体がサポート終了の組み合わせである以上、**最終的にはサポート中のOS・PHPへの移行計画とセットで実施すること**をお勧めします。

---

PHP・サーバーの脆弱性対応や、サポート終了環境からの移行でお困りの場合は、[東洋インターネットサービス株式会社](https://www.tyis.co.jp/)までご相談ください。現行環境の調査から、緩和策の適用、移行計画の策定まで対応いたします。
