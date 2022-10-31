---
title: "Windows ７のインストール用のブータブルUSBの作り方"
date: 2019-08-10 09:25:00 +0900
categories: 
published: true
---
1. Windows 7もしくはWindows Vistaにてcmdを開きます。
1. diskpartを実行します。
1. diskpartに入ります。
```
   list disk           （ここでUSBメモリーのパーティション番号を確認します。）   
   select disk 1    （上記のステップにて確認されたパーティション番号を入れます。）
   clean    
   create partition primary    
   select partition 1    
   active    
   format fs=ntfs quick    
   assign    
   exit               (ここまでしたら、 diskpartから出ます。)
```
1. xcopy x:\ y:\ /cherky (ここで x:\は Windows7 DVDが入っているパス y:\は USBメモリーのパスを入力すればオッケーです。)

終わり！