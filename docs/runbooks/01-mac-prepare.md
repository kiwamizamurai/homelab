# 01: Mac の準備と Proxmox のインストール USB

実施する場所: Mac

## 1. USB 書き込みツールの選択

Ventoy は Mac では使えない。Mac では、balenaEtcher か、Proxmox 公式の `dd` の手順を使う。

| ツール | Mac 対応 | 根拠 |
|---|---|---|
| Ventoy（GitHub 79,926 スター） | 非対応 | 公式ドキュメント（Get Started）のインストール方法は、Windows（GUI / CLI）と Linux（GUI / CLI）だけで、macOS の記載はない。Proxmox VE は、README にある Ventoy のテスト済み ISO の一覧に載っている |
| balenaEtcher（GitHub 34,504 スター） | 対応 | README に「macOS 10.13 (High Sierra) and later; both Intel and Apple Silicon」とある。説明は「Flash OS images to SD cards & USB drives, safely and easily」 |
| Rufus（GitHub 37,949 スター） | 非対応（Windows 向け） | Proxmox 公式ドキュメントの Windows の節に記載がある |
| UNetbootin | 使わない | Proxmox 公式ドキュメントに「Do not use UNetbootin. It does not work with the Proxmox VE installation image.」とある |
| `dd`（Mac 標準） | 対応 | Proxmox 公式ドキュメントに macOS の手順がある（下記） |

スター数は 2026-10-12 時点の値で、`gh repo view` で確認した。

## 2. 使う USB メモリ

Proxmox 公式ドキュメントが示す条件は、次の 3 点である。

- ISO イメージより大きいこと。x86 版 Proxmox VE 9.2-1 の ISO は 1.71 GB（ダウンロードページの表記）
- マウントされていないこと
- 中身が消えるので、大事なデータが入っていないこと

USB の規格（USB 2.0、3.0 など）や容量の下限は、公式ドキュメントに書かれていない（2026-10-13 に確認）。

## 3. ISO のダウンロードと検証

1. https://www.proxmox.com/en/downloads/proxmox-virtual-environment/iso を開く
2. 「Proxmox VE 9.2 ISO Installer」（Version 9.2-1、File Size 1.71 GB）をダウンロードする
   - 同じページの「For ARM64: Proxmox VE 9.2 ISO Installer」（9.2-1-arm64）は Arm 用なので、Beelink（Intel N100）には使わない
3. SHA256 を確認する。ページの表記は `4e88fe416df9b527624a175f24c9aa07c714d3332afb1ee3dbf3879573ef2c6c` である

```
shasum -a 256 <ダウンロードした ISO のパス>
```

出力が、ページの SHA256SUM と一致することを確認する。`shasum` は、kubectl の公式手順でも macOS の検証に使われている。

## 4. USB に書き込む

### 方法 A: balenaEtcher（GUI）

1. https://github.com/balena-io/etcher のダウンロードページから、macOS 版を入れる
2. Etcher の画面で、ISO と USB メモリを選んで書き込む。Proxmox 公式ドキュメントは、「It will guide you through the process of selecting the ISO and your USB flash drive」と案内している

Etcher の公式情報（https://etcher.balena.io/ 、https://github.com/balena-io/etcher/blob/master/docs/USER-DOCUMENTATION.md 、2026-10-13）は次のとおり。

- 書き込みが完了したことを確認する（「Validated Flashing」）
- システムのドライブを隠して警告を出す（「Hard Drive Friendly」）。書き込み先の取り違えを避けるための機能である
- イメージをバイト単位でそのままコピーする。Windows のイメージのように特別な処理が必要なものは、そのままでは起動しない（Etcher の FAQ）
- Proxmox の ISO は、Proxmox 公式が `dd` での書き込みを案内している。Etcher も、同じバイト単位のコピーなので、書き込める

### 方法 B: `dd`（Proxmox 公式の macOS 手順）

出典: https://pve.proxmox.com/pve-docs/chapter-pve-installation.html の「Instructions for macOS」

```
# hdiutil convert proxmox-ve_*.iso -format UDRW -o proxmox-ve_*.dmg
```

公式は、`.dmg` が自動で付く場合があると書いている。

```
# diskutil list
```

USB を挿して、もう一度実行する。増えたディスクが USB で、デバイス番号（`/dev/diskX`）が分かる。

```
# diskutil list
# diskutil unmountDisk /dev/diskX
```

`X` は、上で確認したディスク番号である。

```
# sudo dd if=proxmox-ve_*.dmg bs=1M of=/dev/rdiskX
```

公式の注記: 最後のコマンドでは、`diskX` ではなく `rdiskX` を使う（書き込みが速くなる）。デバイス名を間違えると、別のディスクの中身が消える。`diskutil list` で、サイズが USB と一致することを確認する。

## 5. 作業用ツールのインストール（Mac）

ここで入れるツールは、後の手順で使う。

### kubectl

出典: https://kubernetes.io/docs/tasks/tools/install-kubectl-macos/

Apple Silicon の場合は、次のコマンドを実行する。

```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/arm64/kubectl"
```

Intel の場合は、`arm64` を `amd64` に変える。次は、検証用のチェックサムである。

```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/arm64/kubectl.sha256"
echo "$(cat kubectl.sha256)  kubectl" | shasum -a 256 --check
```

`kubectl: OK` と出れば、検証は成功している。

```
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
sudo chown root: /usr/local/bin/kubectl
kubectl version --client
```

公式の注意: kubectl とクラスタのマイナーバージョンの差は、1 つ以内にする。

Homebrew を使う場合は、同じ公式ページの次のコマンドでも入れられる。

```
brew install kubectl
```

`brew install kubernetes-cli` でもよい。確認は `kubectl version --client` で行う。

### Terraform

出典: https://developer.hashicorp.com/terraform/install

```
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
```

### Argo CD の CLI

出典: https://argo-cd.readthedocs.io/en/stable/getting_started/

```
brew install argocd
```

### Cilium の CLI

Cilium の CLI は、Mac には入れない。k8s-home VM の上に入れる（[04](04-k8s-home.md)）。
