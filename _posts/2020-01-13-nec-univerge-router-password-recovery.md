---
title: "NEC UNIVERGEルータのパスワードを忘れた場合"
date: 2020-01-13 16:00:00 +0900
categories: 
published: true
---

#### NECではパスワードリカバリーをスーパリセットと言うらしく、下記の様にパスワードをリセットする事が可能です。
#### 注意事項ですが、すべての設定が消えてしまうので注意してください。

### スーパーリセット
アドミニストレータ権限ユーザのパスワードを忘れてしまったときや、すべての設定データを工場出荷時の設定に戻したいときには、スーパーリセットを行います 。
* 注意　スーパーリセットを行うと、工場出荷時の設定となるため、ローカルコンソールのみからのアクセスとなります。
* 注意　スーパーリセットを行うと、ランニングコンフィグ、スタートアップコンフィグの設定情報もすべて消去されます。ただし、ライセンスキーは消去されません。
* メモ　現在のランニングコンフィグの設定内容を保存しておきたい場合には、本章「コンフィグの管理」を参照して、スーパーリセット前に保存してください。
* メモ　スーパーリセットを行っても、日付・時刻の値は保持されます。 

#### **スーパーリセット手順**
1. **電源スイッチONによる起動**

本製品にローカルコンソールを接続した状態で、電源スイッチをONにします。
1. **ブートモニタモードへの移行**

プログラムファイルのロード中を示す文字「##」が出力されている途中でCtrl+cを入力し、ブートモニタモードに移行します。
```
NEC Bootstrap SoftwareCopyright
© NEC Corporation 2001-2014. All rights reserved.

%BOOT-INFO: Trying flash load, exec-image [ix2215-ms-9.0.9.ldc].
Loading: #########　　　＜－－－Ctrl+Cを入力
NEC Bootstrap Software, Version 6.0
Copyright © NEC Corporation 2001-2014. All rights reserved.
boot[0]>  
```
1. **スーパーリセット実行**

ccコマンドを実行し、スタートアップコンフィグを削除します。
```
boot[0]> cc　＜－－Enter
Enter “Y” to clear startup configuration: y
% Startup configuration is cleared.

NEC Bootstrap Software, Version 6.0
Copyright © NEC Corporation 2001-2014. All rights reserved.
boot[0]>
>
```
1. **プログラムファイルの起動**

bコマンドを実行し、プログラムファイルのロードを開始します 。
```
boot[0]>b 　　＜－－Enter
NEC Bootstrap Software
Copyright © NEC Corporation 2001-2014. All rights reserved.

%BOOT-INFO: Trying flash load, exec-image [ix2215-ms-9.0.9.ldc].
Loading: ##################################### [OK]

<省略>

Router# 
>
```
## 初期化完了！