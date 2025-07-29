---
title: Raspberry Pi を NAS にする
emoji: "🗃"
type: "tech"
topics: ["RaspberryPi", "NAS"]
published: false
---

## はじめに

おうち倉庫にずっと籠もっていた Raspberry Pi (以降 RPi)を NAS 化したその備忘録です.

![](/images/raspberry_pi_nas/pi_nas.jpg =400x)


## 使うもの

### ハードウェア

Jeff 氏[^1]のブログを参考にしました.

| 品名                  | 内容                                                  |
| --------------------- | ----------------------------------------------------- |
| Raspberry Pi 5        | NAS にするマシン本体                                  |
| Radxa Penta SATA HAT  | SATA 接続で内蔵ディスクをたくさん使えるようにする HAT |
| Radxa POWER DC 12 60W | Radxa Penta SATA HAT 用の電源供給ユニット             |
| Radxa Penta Top Board | おしゃれなファンと OLED が載ったボード                |
| 内蔵ディスク          | エンジニアの家なら多分そこら辺に落ちてるやつ          |

### ソフトウェア

| 品名                 | 内容                     |
| -------------------- | ------------------------ |
| Raspberry Pi OS Lite | Raspberry Pi 用の OS     |
| OpenMediaVault 7     | NAS 構築用のソフトウェア |

## 0. 発注

Radxa 系のパーツをアリエクでとりあえず注文します.
1 週間もすれば届きます. 箱が多少潰れるのはご愛嬌.

## 1. Raspberry Pi の事前準備

rpi-imager を利用して OS を microSD カードに焼きます.
今回は OpenMediaVault (以降 OMV) を使いたいので, 互換性のある Raspberry Pi OS Lite を焼きます. OMV が Raspberry Pi OS のデスクトップ版をサポートしていないのでちょっと注意です.

ユーザ管理とかは好きなようにやりましょう.
以降は `lru` というユーザ名で, sudoers に入ってる想定で記述します.

最近の Imager はどうやら焼く時点で hostname とかの設定ができるようになっているようですが, まんまと忘れたので愚直に設定を入れていきます.

### 1.1 hostname の設定

インストールしたら適当な名称に hostname を変更しておきます.

```bash
$ sudo hostnamectl set-hostname pi-nas
```

`hostnamectl` では `/etc/hostname` まで変わってくれますが, `/etc/hosts` の内容は変わらないので合わせておきます. CLI editor は `vim` 一択です. `vim` を入れましょう.

```bash
$ sudo apt install -y vim
$ sudo vim /etc/hosts
# 自身を指す記述の行を修正
127.0.1.1  pi-nas
```

### 1.2 sshd の起動

Raspberry Pi をリモートから操作するために sshd を起動しておきます.

```bash
$ sudo raspi-config
# Interfacing Options > SSH > Enable

$ systemctl status sshd
● ssh.service - OpenBSD Secure Shell server
     Loaded: loaded (/lib/systemd/system/ssh.service; enabled; preset: enabled)
     Active: active (running) ...
```

ここまで来たら一旦 reboot しておきます.

```bash
$ sudo reboot
```

### 1.3 作業する PC からの ssh key 投げ

ssh で接続するための鍵を作成して登録しておきます.
ローカルネットワークでつなげて, `.local`ドメインで mDNS を利用して接続します.

```bash
# 鍵は好きなやつ作る
$ ssh-keygen -t ed25519
# ユーザ名部分は RPi のユーザ名に合わせる
$ ssh-copy-id -i ~/.ssh/id_ed25519.pub lru@pi-nas.local
```

## 2. ハードウェアの組み立て

届いたハードウェア群を組み立ててきます.
基本公式の手順[^2]に従えば問題ありません.

![](/images/raspberry_pi_nas/parts.jpg)

### 2.1 HAT の接続

Radxa Penta SATA HAT を接続します.
RPi のアクティブクーラーは付けたままにしたいですが, ヒートシンクとこの HAT はガッツリ干渉します. 仕方なしにヒートシンクの一部をニッパーでぶち飛ばして合わせます.

![](/images/raspberry_pi_nas/parts_collision.jpg =400x)
![](/images/raspberry_pi_nas/collision_resolved.jpg =400x)

その後ケーブルを RPi の PCIe スロットに接続します.
あとは GPIO ピンに HAT のピンヘッダを接続するだけです.

![](/images/raspberry_pi_nas/connect_fpc.jpg =400x)
![](/images/raspberry_pi_nas/assembled_hat.jpg =400x)

このとき, TopBoard をつける想定なら, TopBoard の方のクソ長ネジで止めましょう. (写真は忘れてそのまま止めた図)

### 2.2 ディスクの取り付け

Radxa Penta SATA HAT には SATA ポートが 4 つあります. (側面に eSATA ポートもあります)
基本的に上部の 4 つで済ませたいところですが, サイズ感からわかる通り, 2.5 インチのディスクしか上載せ形態での接続には対応していません. 3.5 インチの HDD もサポートされていますが, Serial ATA ケーブルで直接接続する形になります.

2.5 インチの SSD は 2 台しかなかったのでとりあえずこの 2 台を載っけます.

![](/images/raspberry_pi_nas/assembled_ssd.jpg =400x)

### 2.3 Top Board の取り付け

Radxa Penta Top Board を取り付けます. 小さいネジで止めてしまった人は頑張って小さいネジを外しましょう. クソ長ネジへと差し替えて最上部に Top Board を取り付けます.

![](/images/raspberry_pi_nas/assembled_top_board.jpg =400x)

## 3. 各種ハードウェア制御設定の追加

こちらも公式の手順[^3]に従っていきます.

### 3.1 PCIe スロットの設定

デフォルトでは RPi 5 の PCIe スロットが無効化されているのか, ディスクが認識されません.

```bash
$ sudo lsblk
# SD カードしかない
mmcblk0     179:0    0 119.1G  0 disk
├─mmcblk0p1 179:1    0   512M  0 part /boot/firmware
└─mmcblk0p2 179:2    0 118.6G  0 part /
```

`/boot/firmware/config.txt` の末尾に以下の設定を追加することで接続が有効化されます. `pcie_gen`指定は PCI Express 3.0×1 接続となるようにしているものですが任意です. 書かなくても良い.

```toml
[all]
dtparam=pciex1
dtparam=pciex1_gen=3
```

設定後に再起動します.
再起動後, ディスクが認識されるようになります.

```bash
$ sudo reboot
```

```bash
$ sudo lsblk

NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda           8:0    0 931.5G  0 disk
sdb           8:16   0 931.5G  0 disk
mmcblk0     179:0    0 119.1G  0 disk
├─mmcblk0p1 179:1    0   512M  0 part /boot/firmware
└─mmcblk0p2 179:2    0 118.6G  0 part /
```

PCIe スロットに接続されているデバイスの確認は以下のコマンドで行えます.
8GT/s なので PCIe 3.0 接続ができていますね. 先の任意記述を飛ばした場合は 5GT/s で PCIe 2.0 接続となります.

```bash
# PCIe デバイスの確認
$ sudo lspci
0001:00:00.0 PCI bridge: Broadcom Inc. and subsidiaries BCM2712 PCIe Bridge (rev 21)
0001:01:00.0 SATA controller: JMicron Technology Corp. JMB58x AHCI SATA controller
0002:00:00.0 PCI bridge: Broadcom Inc. and subsidiaries BCM2712 PCIe Bridge (rev 21)
0002:01:00.0 Ethernet controller: Raspberry Pi Ltd RP1 PCIe 2.0 South Bridge

# Link 状態の確認
$ sudo lspci -vvv -s 0001:01:00.0 | grep LnkSta
                LnkSta: Speed 8GT/s, Width x1 (downgraded)
                LnkSta2: Current De-emphasis Level: -6dB, EqualizationComplete+ EqualizationPhase1+
```

### 3.2 ファン / OLED の制御

せっかくつけた Top Board ですから制御パッケージを入れておきましょう. [^4]
rockpi-penta の GitHub から取得でき, ファンと OLED の制御が可能になります.

```bash
$ wget https://github.com/radxa/rockpi-penta/releases/download/v0.2.2/rockpi-penta-0.2.2.deb
$ sudo apt install -y ./rockpi-penta-0.2.2.deb
```

インストール後, `/etc/rockpi-penta.conf` にファンと OLED の設定が書かれています.
OLED に表示される内容はそう多くありませんが, uptime, CPU 温度, IP アドレスなどが表示されます.

![](/images/raspberry_pi_nas/top_board_control.jpg =400x)

フルパワーだとファンがそこそこの騒音を発しますが, 風量が結構あります. 温度測定器を持っていないのが悔やまれる.

:::message
Top Board のファンですが, ここで入れた制御パッケージを停止する, あるいは RPi 自体をシャットダウンすると, 何故か 100% フルパワーで回りだします.
なんかバグらせてそうなのでそのうち解決方法が分かったら追記します.
:::

## 4. OpenMediaVault のインストール

公式の手順[^5]に従います.

### 4.1 事前準備

必要なライブラリをインストールします. (`jq`が入るだけ)

```bash
wget -O - https://github.com/OpenMediaVault-Plugin-Developers/installScript/raw/master/preinstall | sudo bash
```

### 4.2 OMV のインストール

インストールスクリプトを実行します.
実行が完了すれば, もう OMV がインストールされ立ち上がっている状態になります.

```bash
wget -O - https://github.com/OpenMediaVault-Plugin-Developers/installScript/raw/master/install | sudo bash
```

### 4.3 Web UI へのアクセス

Web UI へのアクセスは, hostname が`pi-nas` であれば, ブラウザから `http://pi-nas.local` でアクセスできます.

デフォルトのユーザ名: `admin`, パスワード: `openmediavault` です. ログイン後, パスワードは変えておきましょう.

## 5. RAID

OMV は RAID の取り扱いに対応してはいますが, 構築には対応していません. RAID は`mdadm` を使って構築していきます.

### 5.1 RAID の構築

今回は RAID1 (ミラーリング) を構築します.
ディスクは完全に NAS 用に使う前提なので, パーティションは切らずにディスク全体を RAID に利用します. (ディスクの一部だけを RAID に利用したい場合は, 事前にパーティションを切っておく必要があります)

```bash
$ sudo mdadm --create --verbose /dev/md0 --level=raid1 --raid-devices 2 /dev/sda /dev/sdb
```

構築の進捗は以下のコマンドで確認できます.

```bash
$ cat /proc/mdstat

md0 : active raid1 sdb[1] sda[0]
      976630464 blocks super 1.2 [2/2] [UU]
      [>....................]  resync =  0.5% (4948928/976630464) finish=78.5min speed=206205K/sec
      bitmap: 2/2 pages [32KB], 65536KB chunk
```

ビルド完了まで結構待ちます. 気長に待ちましょう.

### 5.2 RAID のファイルシステム初期化

RAID の構築が完了したら, OMV からデバイスを認識できるようになります.
Web UI の `Storage > File Systems` にある `Create and Mount` ボタンから, ext4 あたりのファイルシステムを作成しマウントします.
最新を追うなら btrfs も選択肢. この辺は好みです.

![](/images/raspberry_pi_nas/disk_mounted.png)

## 6. SMB/CIFS の設定

NAS へアクセス可能としたい機体に Windows がいるので, SMB/CIFS を利用します.

### 6.1 共有フォルダの作成

`Storage > Shared Folders` から, `Create` ボタンを押して共有フォルダを作成します.
適当な名前を付けて, RAID デバイス上に作成します.

![](/images/raspberry_pi_nas/shared_disk.png)

### 6.2 SMB/CIFS の有効化

`Services > SMB/CIFS > Settings` を選択し, `Enable` にチェックを入れます.
特に Workgroup 等の設定が不要であれば, そのまま `Save` ボタンを押して設定を保存します.

### 6.3 共有フォルダの展開

`Services > SMB/CIFS > Shares` から, `Create` ボタンを押して先ほど作成した共有フォルダをアクセス可能な状態にします.

![](/images/raspberry_pi_nas/shared_smb.png)

### 6.4 ユーザの作成

`User Management > User` から, `Create` ボタンを押してユーザを作成します.
共有フォルダにアクセスしてくるだけなので, そんなにたくさんの権限は要らないでしょう.
Group を定義しておくと後々の権限設定が楽になるんじゃないかなと.

![](/images/raspberry_pi_nas/nas_user.png)

### 6.4 共有フォルダへのアクセス権限の設定

`Storage > Shared Folders` から先ほど作成した共有フォルダを選択し, `Permission` ボタンから権限を設定します.
先ほど作成したユーザまたはそのユーザに付与したのグループを選択し, `Read/Write` にチェックを入れます.

## 7. アクセス検証

### From Linux

![](/images/raspberry_pi_nas/accessible_from_linux.png)

無事アクセスできました. 適当にテキストファイルでも置いておきます.

### From Windows

![](/images/raspberry_pi_nas/accessible_windows.png)

こちらからも無事アクセス出来ました.

## おしまい

今まで常時使うわけでもないけどちょっとなくなると困るんだよな〜みたいなファイルの置き場に困ってましたがこれで解決です.
さっそく数百 GB ほど整理したファイルを突っ込んでみましたが現状特に問題は起きていません.

今後は HDD を追加して, 更に大容量に拡張していく見込みです.

[^1]: [Radxa's SATA HAT makes compact Pi 5 NAS - Jeff Geerling](https://www.jeffgeerling.com/blog/2024/radxas-sata-hat-makes-compact-pi-5-nas)
[^2]: [Radxa Penta SATA HAT - Assemble](https://docs.radxa.com/en/accessories/penta-sata-hat/penta-for-rpi5#assemble)
[^3]: [Radxa Penta SATA HAT - Start Use](https://docs.radxa.com/en/accessories/penta-sata-hat/penta-for-rpi5#start-use)
[^4]: [Radxa Penta SATA HAT Top Board - Software Support](https://docs.radxa.com/en/accessories/penta-sata-hat/sata-hat-top-board#software-support)
[^5]: [OpenMediaVault - Raspberry Pi Install](https://wiki.omv-extras.org/doku.php?id=omv7:raspberry_pi_install)
