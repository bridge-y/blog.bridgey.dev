---
title: "Btrbkを使ったバックアップ"
slug: backup-using-btrbk
author: ""
type: ""
date: 2020-03-14T13:58:58+09:00
draft: false
subtitle: ""
image: ""
toc: true
tocMove: true
tags: ["btrfs", "backup", "btrbk"]
---

前回、BtrfsなサーバにHDDを増設し、RAIDを組んだという記事を書きました。  
(参考: [btrfsでHDDを増設してRAID1に変換する]({{< ref "/post/btrfs-raid1/index.md">}}))  

今回はRAIDを組んだときに設定した、バックアップについてまとめます。

## ツールの選定

せっかくBtrfsを使用しているので、その機能を利用した方法でバックアップをとりたいです。  
高速にバックアップできるようなので。  

{{< blockquote author="Incremental Backup - btrfs Wiki" link="https://btrfs.wiki.kernel.org/index.php/Incremental_Backup" >}}
Incremental Snapshot Transfer<br>
 [...]. This is far quicker than e.g. rsync could, especially on large file systems.
{{< /blockquote >}}

ノートPCでは`snapper`を利用してスナップショットをとっていますが、外部HDDへの転送は別途ツールをインストールする必要があるため[^1]、今回は利用を見送りました。  
[^1]: [Snapper - ArchWiki](https://wiki.archlinux.jp/index.php/Snapper#.E5.A4.96.E9.83.A8.E3.83.89.E3.83.A9.E3.82.A4.E3.83.96.E3.81.AB.E5.B7.AE.E5.88.86.E3.83.90.E3.83.83.E3.82.AF.E3.82.A2.E3.83.83.E3.83.97)

今回は、上記引用ページにも記載のある、[btrbk](https://digint.ch/btrbk/)を利用します。  

## インストール

ソースコードをビルドします[^2]。  
バージョンは`0.29.1`です。  

[^2]: [btrbk - Installation](https://digint.ch/btrbk/doc/install.html)

```
# cd /usr/local/src
# wget https://digint.ch/download/btrbk/releases/btrbk-0.29.1.tar.xz
# tar xf btrbk-0.29.1.tar.xz
# cd btrbk-0.29.1/
# make install
```

## 設定

### トップレベルサブボリュームのマウント

`btrbk`はトップレベルサブボリューム（btrfsのルート）をマウントしていると設定が楽です。  
そうでない場合の設定については、まだドキュメント[^3]を正確に理解していないため試していません。  
[Snapperの推奨ファイルシステムレイアウト](https://wiki.archlinux.jp/index.php/Snapper#.E6.8E.A8.E5.A5.A8.E3.83.95.E3.82.A1.E3.82.A4.E3.83.AB.E3.82.B7.E3.82.B9.E3.83.86.E3.83.A0.E3.83.AC.E3.82.A4.E3.82.A2.E3.82.A6.E3.83.88)のようにサブボリュームをマウントして使用している場合は、ルートをマウントします。  

```
$ sudo mount -t btrfs /dev/sda /mnt/raid
```

起動時にマウントするように、/etc/fstabに記述しておくとよいでしょう。  
私の場合は以下のような感じです。  

```
LABEL=hdd /mnt/raid/ btrfs defaults,noatime,autodefrag,compress-force=zstd,space_cache 0 2
```

[^3]: [btrbk - btrbk.conf(5)](https://digint.ch/btrbk/doc/btrbk.conf.5.html)  

### btrbk

解凍したディレクトリ内の設定ファイルのサンプルをコピーし、設定を記述します。  

```
# cp /usr/local/src/btrbk-0.29.1/btrbk.conf.example /etc/btrbk/btrbk.conf
```

私は下記のような設定を加えました。

```
snapshot_preserve_min   2d
snapshot_preserve       14d

target_preserve_min     no
target_preserve         20d 10w *m

archive_preserve_min    latest
archive_preserve        12m 10y

# ↓以下を追記

volume /mnt/raid/
  target send-receive /mnt/backup/internal/
  subvolume @videos
    snapshot_dir @snapshots
  subvolume @nextcloud
    snapshot_dir @snapshots
```

**/mnt/raid/** の2つのサブボリュームを、それぞれバックアップ用HDD（ **/mnt/backup/internal/**）にバックアップしています。  

下記スクリプトを **/etc/cron.daily/btrbk**に記述し、毎日バックアップするようにしました。  

```bash
#!/bin/sh
exec /usr/sbin/btrbk -q run
```

`btrbk run`では、以下の処理が行われます[^4]。  
- スナップショットの作成
- バックアップの作成
- 保持期間を過ぎたバックアップの削除
- 保持期間を過ぎたスナップショットの削除

デフォルトの設定では増分バックアップが行われます。  

[^4]: [btrbk - btrbk(1)](https://digint.ch/btrbk/doc/btrbk.1.html)  

#### 設定ファイルの読み方

設定ファイルの一部の記述について、その意味を記します。  

- **volume /mnt/raid/**  
  バックアップ対象のサブボリュームを含んでいるトップレベルサブボリュームのマウントポイント
- **target send-receive /mnt/backup/internal/**  
  バックアップ先のファイルパス  
  バックアップにはbtrfs send/receiveを用いる  
- **subvolume @nextcloud**  
  @nextcloudという名前のサブボリュームをバックアップする  
- **snapshot_dir @snapshots**  
  スナップショットの保存先（volumeで指定した場所からの相対パス）  
  指定したディレクトリが存在しないとbtrbkは実行に失敗するため、予め作成しておく必要がある  

snapshot_preserve_minやsnapshot_preserveなどの設定は、サンプルファイルのまま使用しています。 
このあたりの設定で、スナップショットやバックアップをどの程度残すかを決めるのですが、イマイチ挙動を理解できていません。  

- **snapshot_preserve 14d**  
  スナップショットを保持する期間の設定  
  上記設定では、14日間保持する  
  それ以前のスナップショットは削除される  
- **snapshot_preserve_min 2d**  
  ドキュメント[^3]によると、スナップショットを保持する最小期間のようだけれど、snapshot_preserveの設定が優先されます  
- **target_preserve 20d 10w \*m**  
  バックアップを保持する期間の設定  
  日単位と週単位など、複数設定した場合にどのような挙動になるのかわかっていません  

## 動作例

ある日の動作ログです。

```
2020-03-14T06:25:01+0900 startup v0.29.1 - - - # btrbk command line client, version 0.29.1
2020-03-14T06:25:02+0900 snapshot starting /mnt/raid/@snapshots/@videos.20200314 /mnt/raid/@videos - -
2020-03-14T06:25:03+0900 snapshot success /mnt/raid/@snapshots/@videos.20200314 /mnt/raid/@videos - -
2020-03-14T06:25:03+0900 snapshot starting /mnt/raid/@snapshots/@nextcloud.20200314 /mnt/raid/@nextcloud - -
2020-03-14T06:25:03+0900 snapshot success /mnt/raid/@snapshots/@nextcloud.20200314 /mnt/raid/@nextcloud - -
2020-03-14T06:25:03+0900 send-receive starting /mnt/backup/internal/@videos.20200314 /mnt/raid/@snapshots/@videos.20200314 /mnt/raid/@snapshots/@videos.20200313 -
2020-03-14T06:33:18+0900 send-receive success /mnt/backup/internal/@videos.20200314 /mnt/raid/@snapshots/@videos.20200314 /mnt/raid/@snapshots/@videos.20200313 -
2020-03-14T06:33:18+0900 send-receive starting /mnt/backup/internal/@nextcloud.20200314 /mnt/raid/@snapshots/@nextcloud.20200314 /mnt/raid/@snapshots/@nextcloud.20200313 -
2020-03-14T06:33:38+0900 send-receive success /mnt/backup/internal/@nextcloud.20200314 /mnt/raid/@snapshots/@nextcloud.20200314 /mnt/raid/@snapshots/@nextcloud.20200313 -
2020-03-14T06:33:38+0900 delete_snapshot starting /mnt/raid/@snapshots/@videos.20200228 - - -
2020-03-14T06:33:38+0900 delete_snapshot success /mnt/raid/@snapshots/@videos.20200228 - - -
2020-03-14T06:33:38+0900 delete_snapshot starting /mnt/raid/@snapshots/@nextcloud.20200228 - - -
2020-03-14T06:33:38+0900 delete_snapshot success /mnt/raid/@snapshots/@nextcloud.20200228 - - -
2020-03-14T06:33:38+0900 finished success - - - -
```
やはりスナップショットの作成は早いですね。  
バックアップは別HDDに転送しているので、データ量に応じて時間がかかっています。  

また、設定通り保持期間を過ぎたスナップショット（20200228）が削除されています。  
バックアップの削除ログがありませんが、どうも10wや*mの方の設定が効いている？？  
20日前よりも古いバックアップがまだ残っているんですよね。  

`btrbk run -n -S`でスケジュールを表示できるので、btrbkがどのようにスナップショットやバックアップの保持を決めているのか、今後調査してみたいと思います。  


## おわりに

btrbkを用いたバックアップについてまとめました。  
自宅サーバは私がDockerで遊ぶだけでなく、家族のデータ保管の役割を担うようになっていたので、バックアップの自動化ができて満足しています。  

HDDを定期的に新しいものに交換することを忘れないようにしたいです。  

