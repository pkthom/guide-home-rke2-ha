# guide-home-rke2-ha

自宅ラボ環境向けに、Ansibleで **RKE2 HA クラスター** を全自動構築するプレイブックです。
OS設定、RKE2構築、MetalLB、Rancherの導入を `site.yml` ひとつで一括実行します。

> [!WARNING]
> リポジトリ内の設定値（IP・トークン・パスワード）はダミーです。必ず自身の環境に書き換えてください。

## Requirements

- **Ansible コントローラ**: Ansible と kubectl が動作する Linux マシン
- **ターゲットノード**: SSH接続可能な Ubuntu マシン（例: Control Plane x3, Worker x3）
- ノード同士が同一 L2 ネットワークに属していること（MetalLB用）

## Quick Start

### 1. 準備
リポジトリをクローンし、コントローラから各ノードへ公開鍵（パスワードなしSSH）を設定します。

```bash
git clone [https://github.com/pkthom/guide-home-rke2-ha.git](https://github.com/pkthom/guide-home-rke2-ha.git)
cd guide-home-rke2-ha
```

### 2. 設定ファイルの編集
環境に合わせて以下の2ファイルを修正します。

- **`inventory.ini`**: 対象ノードのIPアドレス
- **`group_vars/all.yml`**:
  - `rke2_token`: クラスター参加用トークン
  - `tls_sans`: 先頭Control PlaneのIP と Rancherのホスト名
  - `metallb_ip_range` / `metallb_vip`: 割り当てるVIP範囲
  - `rancher_hostname` / `rancher_bootstrap_password`: Rancherの設定

*(任意) 接続確認: `ansible-playbook playbooks/ping.yml`*

### 3. デプロイ
マスタープレイブックを実行します。

```bash
ansible-playbook site.yml
```
※自動的にコントローラの `~/.kube/config` までセットアップされます。

### 4. 確認とアクセス
デプロイ完了後、すぐに `kubectl` が利用可能です。

```bash
kubectl get nodes
```

ブラウザからアクセスするため、お使いのPCの `/etc/hosts` に VIP を追記します。
```text
192.168.1.100 chart-example.local
192.168.1.100 rancher.example.local
```
- Rancher UI: `https://rancher.example.local/`
- Test Nginx: `http://chart-example.local/` (※ `playbooks/test-nginx.yml` 実行時)

## Project Structure

```text
.
├── README.md
├── ansible.cfg
├── inventory.ini        # ターゲットノードの定義
├── site.yml             # 全自動構築用マスタープレイブック ★
├── group_vars/
│   └── all.yml          # クラスター全体で使う変数群
├── playbooks/
│   ├── ping.yml         # 疎通確認用
│   ├── rke2.yml         # RKE2構築
│   ├── metallb.yml      # MetalLBデプロイ
│   ├── rancher.yml      # Rancherデプロイ
│   └── test-nginx.yml   # テスト用アプリ
└── roles/               # タスク定義
    ├── ping/
    ├── rke2/
    ├── metallb/
    ├── rancher/
    └── test_nginx/
```
