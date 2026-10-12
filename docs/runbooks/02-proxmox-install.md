# 02: Proxmox VE のインストールと初期設定

実施する場所: Beelink（本体の画面とキーボード）。その後は Mac のブラウザ

出典: https://pve.proxmox.com/pve-docs/chapter-pve-installation.html（Proxmox VE 9.2.13）、https://pve.proxmox.com/pve-docs/chapter-sysadmin.html

## 前提（公式ドキュメントの要件）

| 項目 | 内容 |
|---|---|
| CPU | 64-bit x86（Intel 64 または AMD64）。KVM のために Intel VT または AMD-V が必要 |
| 最小要件（評価用） | RAM 1 GB とゲスト用の RAM、ハードディスク、NIC 1 つ |
| 推奨 | OS と Proxmox VE のサービス用に最低 2 GB とゲスト用の RAM。SSD が望ましい |
| ブラウザ | Firefox、Chrome、Edge、Safari（当年のリリース） |

インストールすると、選んだディスクの既存のデータがすべて消える（公式ドキュメントの警告）。Beelink に入っている Windows も消える。Windows のライセンスを控えるなら、先に控えておく。

## 1. インストーラーの起動

1. USB（[01](01-mac-prepare.md)）を Beelink に挿して電源を入れ、USB から起動する
   - BIOS に入るキー: 電源を入れた直後に、Delete キーを繰り返し押す。Beelink の公式サポートが、Mini S12 Pro の質問に答えた内容である（2024-12-16、https://bbs.bee-link.com/d/3726-boot-from-usb-pen-drive ）
   - USB から起動できないときは、次の 4 点を確認する（同じ回答）
     1. Secure Boot が有効だと、署名のないブートローダーは起動しない。無効にする
     2. 起動順序の先頭が USB になっているか
     3. USB のフォーマット
     4. 別の USB ポートを試す
   - 起動メニューに入るキー: Beelink の公式サポートの回答（2025-05-21、https://bbs.bee-link.com/d/182-factory-reset-to-return-device ）に、2 つの記述がある。「起動時に F7 または F9 を繰り返し押す（一部の Beelink モデルのショートカットキー）」と、「F7 または ESC を繰り返し押す（具体的なキーは、マニュアルを参照）」である。この回答では、Mini S12 Pro のキーを特定できない。同じ Mini S12 Pro のフォーラムの質問者（2024-12）は、F7 で起動メニューに USB が出たと書いている。実機で試して、効いたキーをここに書く
   - Proxmox 公式ドキュメントの条件は、ファームウェア設定で、インストールメディアからの起動が有効になっていることである。Secure Boot の無効化が必要なのは、Proxmox VE 8.1 より前のインストーラーで、9.2 は対象外である
2. メニューで「Install Proxmox VE (Graphical)」を選ぶ（通常のインストール）
   - 画面が出ない、または固まる場合は、「Install Proxmox VE (Terminal UI)」を使う。それでも固まる場合は、同じページの「Adding the nomodeset Kernel Parameter」の手順に従う
   - キーボードだけで操作する場合は、`ALT` と、ボタンの下線の付いた文字を同時に押す（例: `ALT + N` で Next）

## 2. インストーラーの設定項目

公式ドキュメントの流れは、次のとおり。

1. EULA を読んで同意する
2. インストール先のディスクを選ぶ。デフォルトは「サーバー全体を使い、既存のデータをすべて消す」である
   - Options でファイルシステムを選べる。デフォルトは ext4 で、ext4 または xfs を選ぶと LVM が使われる
   - ZFS も選べる。ただし、ハードウェア RAID の上の ZFS はサポート外で、データを失うおそれがある
3. 場所、タイムゾーン、キーボードレイアウトを選ぶ（自動検出されることが多い）
4. root のパスワードとメールアドレスを入力する
   - パスワードは 8 文字以上。公式は、12 文字以上で、大文字、小文字、数字、記号を含めることを推奨している
   - メールアドレスは、更新情報や cron のエラー通知の送信先になる
5. ネットワークを設定する（最後のステップ）
   - 管理用のネットワークインターフェースを選ぶ
   - IPv4 と IPv6 は、どちらか一方だけ指定できる（両方は指定できない）。デュアルスタックにするなら、インストール後に追加する
   - IP アドレス、ホスト名、ゲートウェイ、DNS は、環境に依存する値で、まだ決めていない
6. サマリーを確認して、Install をクリックする
7. インストールが終わったら、USB を抜いて再起動する（数秒後に自動で再起動される）

インストールに失敗したときは、`CTRL + ALT + F2` で 2 番目の TTY を開いて、エラーを確認する（公式ドキュメント）。

### ext4（LVM）を選んだときの領域サイズ

出典: https://pve.proxmox.com/pve-docs/chapter-pve-installation.html の「Advanced LVM Configuration Options」（2026-10-13）

ext4 または xfs を選ぶと、インストーラーは、ボリュームグループ `pve` と、論理ボリューム `root`、`data`、`swap` を作る。Options で、次の値を指定できる。

| 項目 | 内容（公式） |
|---|---|
| `hdsize` | 使うディスクの合計サイズ。残りを別の用途に空けておける |
| `swapsize` | swap のサイズ。既定は、搭載メモリと同じ大きさで、最小 4GB、最大 8GB。ただし `hdsize / 8` を超えない。0 にすると、swap を作らない |
| `maxroot` | root（OS）の最大サイズ。ストレージが 48GiB より大きい場合、既定は `hdsize / 4` で、上限は 96GiB。48GiB 未満の場合は、少なくとも `hdsize / 2` |
| `maxvz` | data の最大サイズ。data の実際のサイズは `hdsize − rootsize − swapsize − minfree` で、`maxvz` を超えない。LVM thin の場合、data のプールは、サイズが 4GB を超えるときだけ作られる |
| `minfree` | ボリュームグループに残す空き。128GB より大きい場合の既定は 16GB で、それ以下の場合は `hdsize / 8` |

この式に、Beelink の SSD（500GB、メモリ 16GB）を当てはめた計算値は、次のとおり。実機の `hdsize` で、もう一度計算する。

- swap: メモリが 16GB なので、上限の 8GB
- root: `hdsize / 4` が 96GiB を超えるので、96GiB
- 空き（`minfree`）: 16GB
- data（VM のディスクを置く領域）: 約 345GiB。`hdsize` を 500GB（約 465.7GiB）として、465.7 − 96 − 8 − 16 ≒ 345.7 で求めた。GB と GiB の違いがあるので、おおよその値である

## 3. 管理画面へのアクセス

1. Mac のブラウザで `https://<インストール時に指定した IP>:8006` を開く
2. ユーザー名に `root`、Realm に PAM、パスワードにインストール時のものを指定する

公式ドキュメントが示す、インストール後のチェックリストは、次のとおり。

- 購読キーを持っていなければ、公開リポジトリを設定する（次の手順）
- IP 設定とホスト名を確認する
- タイムゾーンを確認する
- ファイアウォール設定を確認する

インストーラーは、最初の Ethernet カードに接続したブリッジ `vmbr0` を 1 つ作る（公式ドキュメント）。VM と LXC は、このブリッジに接続する。

## 4. パッケージリポジトリの設定（購読キーなし）

出典: https://pve.proxmox.com/pve-docs/chapter-sysadmin.html の「Package Repositories」

- 既定では、`pve-enterprise` リポジトリが有効になっていて、購読キーがないとエラーになる
- 購読キーがない場合は、`pve-enterprise` を無効にして、`pve-no-subscription` を設定する
- `pve-no-subscription` について、公式ドキュメントは「テストと非本番用。本番サーバーでは推奨しない」と書いている

### 4-1. enterprise リポジトリを無効にする

`/etc/apt/sources.list.d/pve-enterprise.sources` の該当する stanza に、`Enabled: no` の行を追加する（公式ドキュメント）。元の内容は、次のとおり。

```
Types: deb
URIs: https://enterprise.proxmox.com/debian/pve
Suites: trixie
Components: pve-enterprise
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
```

### 4-2. no-subscription リポジトリを追加する

ファイルは、公式ドキュメントが推奨する `/etc/apt/sources.list.d/proxmox.sources` である。

```
Types: deb
URIs: http://download.proxmox.com/debian/pve
Suites: trixie
Components: pve-no-subscription
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
```

公式ドキュメントの注記: Debian の基本リポジトリ（`/etc/apt/sources.list.d/debian.sources`）も、常に必要である。

### 4-3. 更新の取得とインストール

出典: https://pve.proxmox.com/pve-docs/chapter-sysadmin.html の「System Software Updates」

更新は、GUI でも CLI でもインストールできる。CLI では、次のコマンドを実行する。

```
# apt update
# apt full-upgrade
```

公式の注意: Proxmox VE は、Debian の上でローリングリリースを採用している。`apt upgrade` を使うと、一部のパッケージだけが更新された、壊れた状態になることがある。`apt full-upgrade` は、依存関係を満たすために既存のパッケージを削除できるが、`apt upgrade` は削除できないためである。更新には `apt full-upgrade` を使う。

## 5. ファイアウォールの既定（公式ドキュメント）

出典: https://pve.proxmox.com/pve-docs/chapter-pve-firewall.html（2026-10-13）

- ファイアウォールは、既定では完全に無効になっている。有効にするには、クラスタ全体の設定ファイル `/etc/pve/firewall/cluster.fw` に `enable: 1` を書く。GUI の Datacenter → Firewall でも設定できる
- 有効にすると、すべてのホストへの通信が、既定でブロックされる。ただし、ローカルネットワークからの次の通信は許可される: Web 管理画面（TCP 8006）、SSH（TCP 22）、VNC コンソール（TCP 5900〜5999）、SPICE プロキシ（TCP 3128）、corosync（クラスタ用）など。ループバックと確立済みの接続も許可される
- ローカルネットワークの外から管理画面に入る場合は、IPSet `management` にリモートの IP を追加する
- ホスト個別の設定は、`/etc/pve/nodes/<ノード名>/host.fw` に書く

この構成では、Proxmox の管理画面を、フラットな LAN の中だけで使う（Tailscale には参加させない）。ファイアウォールを有効にするかどうかは、構築後に決める。

## 6. 確認

- GUI のノードのサマリーで、リポジトリの状態を確認できる（公式: Repository パネル）
- `pveperf` で、CPU とディスクの簡易ベンチマークを取れる（公式: 「quick and general benchmark」）
