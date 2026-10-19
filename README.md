# guide-home-rke2-ha

自宅ラボ向けに、Ansible で **RKE2 の HA クラスター** を組み立てるガイドです。クローンして `inventory.ini` と `group_vars/all.yml` のダミー値を自分の環境に書き換えてから playbook を実行してください。

リポジトリ内の IP・トークン・パスワードはすべてサンプルです。そのままではクラスターは作れません。

## Project Structure

```
guide-home-rke2-ha/
├── ansible.cfg
├── inventory.ini
├── group_vars/
│   └── all.yml
├── playbooks/
│   ├── rke2.yml
│   ├── metallb.yml
│   ├── test-nginx.yml
│   └── rancher.yml
└── roles/
    └── ping/
```

## Requirements

- Ansible を実行する Linux マシン（コントローラ）
- SSH で到達できる Ubuntu ノード（例: Control Plane 3 台、Worker 3 台）
- ノード同士が同一 L2 セグメント（MetalLB の L2 Advertisement 用）
- コントローラから Kubernetes API に到達できること

ダミー構成の対応表です。必ず自分の LAN に合わせて書き換えてください。

| 役割 | ホスト | ダミー IP |
| --- | --- | --- |
| Control Plane | cp1 / cp2 / cp3 | 192.168.1.11–13 |
| Worker | worker1 / worker2 / worker3 | 192.168.1.21–23 |
| MetalLB VIP 範囲 | — | 192.168.1.100–110 |
| Ingress VIP（例） | chart-example.local / rancher.example.local | 192.168.1.100 |

## Setup

### 1. Clone

```
git clone https://github.com/pkthom/guide-home-rke2-ha.git
cd guide-home-rke2-ha
```

### 2. Install Ansible (Ubuntu/Debian)

```
sudo apt update
sudo apt install -y ansible
```

### 3. Set up SSH keys

コントローラで鍵を作成します。

```
ssh-keygen -t ed25519
```

公開鍵を各ノードの `~/.ssh/authorized_keys` に置いてください。

### 4. Edit inventory.ini and group_vars/all.yml

`inventory.ini` に対象ノードを書きます。

```
[cp]
cp1 ansible_host=192.168.1.11 ansible_user=ubuntu
cp2 ansible_host=192.168.1.12 ansible_user=ubuntu
cp3 ansible_host=192.168.1.13 ansible_user=ubuntu

[worker]
worker1 ansible_host=192.168.1.21 ansible_user=ubuntu
worker2 ansible_host=192.168.1.22 ansible_user=ubuntu
worker3 ansible_host=192.168.1.23 ansible_user=ubuntu

[rke2_cluster:children]
cp
worker
```

あわせて `group_vars/all.yml` も書き換えてください。

- `rke2_token` … クラスター参加用トークン（ダミーのまま使わない）
- `tls_sans` … API 証明書の SAN（先頭 Control Plane の IP と Rancher のホスト名）
- `metallb_ip_range` / `metallb_vip` … 自宅 LAN で空いている VIP
- `rancher_hostname` … Rancher に割り当てるホスト名
- `rancher_bootstrap_password` … Rancher の初期パスワード（ダミーのまま使わない）

接続確認（任意）:

```
ansible-playbook playbooks/ping.yml
```

## Deploy

### 5. RKE2 クラスターを作る

```
ansible-playbook playbooks/rke2.yml
```

最初の Control Plane（`cp[0]`）で server を初期化し、残りの Control Plane と Worker がそこに join します。

### 6. kubectl を入れる

コントローラに kubectl を入れてください。手順は公式ドキュメントに従います。

[Install and Set Up kubectl on Linux](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)

例（Linux amd64）:

```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

kubeconfig は先頭の Control Plane から取得します。`192.168.1.11` は自分の `cp1` の IP に置き換えてください。

```
mkdir -p ~/.kube
ssh ubuntu@192.168.1.11 "sudo cat /etc/rancher/rke2/rke2.yaml" > ~/.kube/config
sed -i 's/127.0.0.1/192.168.1.11/' ~/.kube/config
chmod 600 ~/.kube/config
```

### 7. ノードを確認する

```
kubectl get nodes
```

ノードが `Ready` になればクラスターは起動しています。Control Plane には `NoSchedule` の taint が付く想定です。

### 8. MetalLB と Ingress を入れる

```
ansible-playbook playbooks/metallb.yml
```

MetalLB が VIP を払い出し、RKE2 同梱の Ingress NGINX を `LoadBalancer` にします。

割り当てられた外部 IP は次で確認できます。

```
kubectl -n kube-system get svc rke2-ingress-nginx-controller
```

`group_vars/all.yml` の `metallb_vip` は、ここで実際に付いた IP と揃えてください。

### 9. テスト用 Nginx を入れる

```
ansible-playbook playbooks/test-nginx.yml
```

`chart-example.local` 向けの Ingress が `test` ネームスペースに作られます。

### 10. Rancher を入れる

```
ansible-playbook playbooks/rancher.yml
```

cert-manager と Rancher を入れ、CoreDNS に `rancher_hostname` を足します。初期パスワードは `group_vars/all.yml` の `rancher_bootstrap_password` です。初回ログイン後に必ず変更してください。

### 11. /etc/hosts を編集する

ブラウザを使うマシン（コントローラや自分の PC）の `/etc/hosts` に、MetalLB の VIP を書きます。IP とホスト名は自分の値に置き換えてください。

```
192.168.1.100 chart-example.local
192.168.1.100 rancher.example.local
```

## Access

- http://chart-example.local/
- https://rancher.example.local/

## References

- [RKE2](https://docs.rke2.io/)
- [MetalLB](https://metallb.universe.tf/)
- [Rancher](https://ranchermanager.docs.rancher.com/)
- [kubectl (Linux)](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)
- [Ansible](https://docs.ansible.com/)
