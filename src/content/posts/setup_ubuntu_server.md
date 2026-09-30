---
title: ubuntu server setup 
published: 2026-10-01
description: '眠っていたlaptopにubuntu serverを導入' 
image: ''
tags: [tech]
category: 'blog'
draft: false 
lang: 'ja'
---

# はじめに

動画に触発されてやってみた系の記事になります。

<iframe width="100%" height="315" src="https://www.youtube.com/embed/46T4cDQBkDs?si=aC6D3qlCY1H1BMxM" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

用途は明確に決めてないが、現時点ではdiscordのbotを常駐させるのが良いかなと思っている。

::github{repo="sifi-border/black-miruten-bot"}

別のデスクトップPCからsshで接続できるよう設定するまでを一旦ゴールとする。

---

# 準備

必要なのは以下の二つ

- ノートPC
- usbメモリ(4GBあれば十分らしいが、16GBのものを使用)

また、ノートPCの充電上限の設定をしておくとよいとclaudeから助言を賜った。

>メーカーのユーティリティ（Armoury Crate、Lenovo Vantageなど）で設定した値は、多くの場合ノートPC本体に保存されて、OSを入れ替えても残ります。Windowsが動いているうちに設定しておくのがおすすめです。

これは実際ubuntuを導入した後も有効だった。

---

# 手順

## 0. バックアップ

windows OSを吹き飛ばすため、大事なものはどこかに移しておく。

usbメモリもフォーマットされるため、こちらも同様。

## 1. インストール用USBの作成

### isoファイルのDownload

https://ubuntu.com/download/server からisoファイルをDLする。

今回は以下を使用。

`Ubuntu 26.04.1 LTS, Intel or AMD 64-bit architecture`

### Rufusでusbに書き込み

Rufusって何？

> ISOファイルをUSBメモリに書き込んで、そのUSBから起動できるようにするためのWindows用ツールです。無料のオープンソースソフトです。

らしいです。

1. https://rufus.ie/ja/ から、ポータブル版をDL。インストールは不要で、そのまま起動可能。
1. USBメモリを挿し、「デバイス」でそのUSBを選択。
1. 「選択」ボタンからUbuntuのISOファイルを指定。
1. パーティション構成が「GPT」、ターゲットシステムが「UEFI」になっていることを確認して、「スタート」を押下。
1. 書き込みモードは ISOイメージモード を選択。


「無効なUEFIブートローダが検出されました」みたいな警告ウィンドウが出たが、claudeに聞いたら問題なさそうなので続行。
問題なく完了した。

## 2. BIOS設定からubuntuを起動

### BIOSの画面に到達

メジャーなのは電源投入直後にF2などのキーを連打する方法だが、自分の連打力では不可能だった。

claudeに聞いたところ以下の方法で行けました。

> windows起動状態で 設定 → システム → 回復 → PCの起動をカスタマイズする（今すぐ再起動）

### ubuntuを起動

水色背景の画面で`EFI USB Device`を指定して起動。

ただ今回は `System doesn't have any USB boot option` という警告が出た。
OKを選択するとBoot Managerのメニューに遷移したので、ここからUSBメモリの表示名である`Linpus lite (BUFFALO USB Flash Disk)`を選択 > `Try or Install Ubuntu Server` を選択すると黒背景にログが流れ始めた。[^1]

[^1]: 何回か試行したらスッと始まった。マシンの機嫌次第なのか何か設定が書き換わったのかは不明。

## 3. インストールウィザードに従い設定

ここからは冒頭の動画を参考に、言語などの設定を適切に選択。
イレギュラーっぽかったのは以下の二点。 

- Network configuration だけ自動で拾われなかったので、`enp7s0`にIPv4のものを有効化してIPアドレスを取得した。
- Storage configuration にてubuntuのインストール先にHDDが選択されていたので、SSDに変更した。
    - `ubuntu-lv`(論理ボリューム)が100GBと少なめに設定されていたので、SSDの上限の値に変更した。

`Install complete!`の表示を確認し、USBメモリを抜いて`Reboot Now`を選択。

再起動を待つとubuntuのシェルプロンプトが表示され、感動。

## 4. ubuntu サーバーの設定

### laptopを閉じても問題ないよう設定

`sudo vi /etc/systemd/logind.conf` で以下の設定をコメント外し+書き換え。

```
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
...
LidSwitchIgnoreInhibited=no
```

スクリプトでもできるっぽいが、一応手動で設定した。

### その他設定 

```bash
# 諸々のパッケージを更新
sudo apt update && sudo apt full-upgrade -y
# timezone設定
sudo timedatectl set-timezone Asia/Tokyo
# スリープ系の状態（サスペンド、ハイバネートなど）への移行を、システム全体で禁止
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
# 再起動して設定を反映
sudo reboot
```

## 5. ssh接続

`ip a`コマンドなどでIPアドレスを確認。
```bash
$ ip a
...
2: enp7s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
... 
    inet 192.168.11.12/24 metric 100 brd 192.168.11.255 scope global dynamic enp7s0
       valid_lft 165358sec preferred_lft 165358sec
...
```

今回だと`192.168.11.12`.[^2]

[^2]: 後ほどルータの設定画面からDHCP予約を行いIPアドレスを固定した。

デスクトップからssh接続を試みる。

```
$ ssh <username>@192.168.11.12

(パスワードが要求される)

Welcome to Ubuntu 26.04.1 LTS (GNU/Linux 7.0.0-34-generic x86_64)

 * Documentation:  https://docs.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Wed Sep 30 09:05:04 PM JST 2026

  System load:  0.01               Temperature:             50.0 C
  Usage of /:   1.6% of 465.38GB   Processes:               219
  Memory usage: 2%                 Users logged in:         0
  Swap usage:   0%                 IPv4 address for enp7s0: 192.168.11.12


Expanded Security Maintenance for Applications is not enabled.

0 updates can be applied immediately.

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


Last login: Wed Sep 30 21:05:05 2026 from 192.168.11.4
<username>@<servername>:~$
```

感動～；；

## 6. (オマケ) security 設定

### ファイアウォール設定

```bash
sudo ufw allow OpenSSH
sudo ufw enable
```

> この二つで、外から入れるのはSSHだけになります。送信は許可のままなので、DiscordのGatewayへの接続のような、ノートPC側から外へ出る通信は影響を受けません。

とのことです。

### 公開鍵のみでssh接続

#### 公開鍵をserverに登録

```bash
# デスクトップ側
cat ~/.ssh/id_ed25519.pub
```

```bash
# server側
mkdir -p ~/.ssh && chmod 700 ~/.ssh
vi ~/.ssh/authorized_keys # 公開鍵の中身を貼る
chmod 600 ~/.ssh/authorized_keys
```

これでパスワードは要求されなくなる。

#### パスワード認証を無効化

スクリプトで設定を書き換えてしまっています！

```bash
sudo sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config.d/50-cloud-init.conf
sudo sshd -t && sudo systemctl restart ssh
```

sshの設定ファイルとして`50-cloud-init.conf`が効いてくるようなので、そのファイルに対して修正しています。

> 最初は /etc/ssh/sshd_config を書き換えたが、PasswordAuthentication yes のままだった。
> sshd_config.d/ 配下の設定が先に読まれ、sshdは最初に見つけた値を採用するため。
> `sudo sshd -T | grep -i passwordauthentication` で実際に効いている値を確認し、50-cloud-init.conf が原因と分かった。

以下で動作確認。

```bash
ssh -o PubkeyAuthentication=no <user>@<ip>
# → Permission denied (publickey). なら成功
```

# おわりに

claudeが有能すぎる。

zennやqiitaだと推敲に時間を持っていかれるが、個人のblogだと気楽でよい。