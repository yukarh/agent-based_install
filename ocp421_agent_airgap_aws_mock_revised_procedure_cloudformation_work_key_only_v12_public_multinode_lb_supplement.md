# OpenShift 4.21 Agent-based Installer エアギャップ AWS模擬検証手順書

対象: OpenShift Container Platform 4.21.21 / Agent-based Installer / `platform: none` / SNO / AWS閉域模擬 / mirror-registry for Red Hat OpenShift / Quay / oc-mirror v2 / Operator込みミラー  
版: v12-public-procedure-multinode-lb-supplement  
方針: GitHub 公開を想定し、作業に必要な設定・取得・生成コマンドを主導線に置く。冒頭で公式ドキュメントの読み方を整理し、目次と各作業章に `x.x` 節レベルの公式対応 URL と参考ログとの差分を記載する。確認コマンドには想定出力を併記し、エラーチェックや回避コマンドは補足に置く。

---

## 0. この手順書の読み方

この手順書は、AWS 上にオンプレ閉域を模擬する検証環境を作り、OpenShift Container Platform 4.21.21 の Agent ISO を生成するところまでを対象にする。

AWS は OpenShift のクラウド IPI/UPI 環境として使うのではなく、オンプレ閉域を模擬するための下回りとして使う。OpenShift の `install-config.yaml` は `platform: none` とする。

本手順の正規化値は以下。

| 用途 | 値 |
|---|---|
| work EC2 private IP | `10.0.0.10` |
| SNO dummy EC2 private IP | `10.0.1.10` |
| DNS / NTP / Quay | `10.0.0.10` |
| rendezvousIP | `10.0.1.10` |
| Quay | `quay.ocp.lab.test:8443` |
| Cluster name | `ocp` |
| Base domain | `lab.test` |
| SNO hostname | `master-0.ocp.lab.test` |
| SNO default gateway | `10.0.1.1` |

SNO の MAC アドレスは EC2 作成ごとに変わるため、手順書本文には固定値を書かない。CloudFormation Outputs の `SnoMac` を `config/env.sh` に反映する。

以降の work 側コマンドは、特記がない限り work EC2 に SSH 済みで、第3章で作成した `~/ocp-airgap` を作業ディレクトリとして使う。  
`cd ~/ocp-airgap` と `source config/env.sh` は第3章で作業開始時に実行する。同じシェルで連続作業する限り、各章で繰り返さない。新しい SSH セッション、tmux の新しいシェル、または `config/env.sh` を編集した直後だけ読み込み直す。

### 0.1 公式ドキュメントの読み方

OpenShift 4.21 の公式ドキュメントは、Agent-based Installer だけを読めばよさそうに見えるが、閉域/エアギャップで Operator までミラーする場合は、以下の 2 系統を合わせて読む必要がある。

| 系統 | 使う公式ドキュメント | この手順での使い方 |
|---|---|---|
| Agent-based Installer | `Installing an on-premise cluster with the Agent-based Installer` | Agent ISO、`install-config.yaml`、`agent-config.yaml`、`platform: none`、SNO、静的ネットワークを確認する |
| Disconnected environments | `Disconnected environments` | mirror-registry for Red Hat OpenShift、oc-mirror v2、Operator Catalog ミラー、IDMS/ITMS/CatalogSource/ClusterCatalog を確認する |

オンプレ閉域 SNO で、外部レジストリーなしのテクノロジープレビュー方式や PXE/iSCSI/MCE を使わない場合は、公式ドキュメントの読み方は次のように割り切る。

| 目的 | 読む章・節 | この手順での扱い |
|---|---|---|
| Agent-based Installer の全体像を掴む | Agent-based Installer `1.1`、`1.1.1`、`1.1.2`、`1.1.3` | SNO、rendezvous host、`platform: none` の前提理解 |
| ホストとネットワーク要件を確認する | Agent-based Installer `1.4`、`1.4.1`、`1.5`、`1.5.2`、`1.6`、`1.6.1`、`1.6.1.1`、`1.6.2` | MAC、role、静的 IP、DNS、API/Ingress、LB 相当の要件確認 |
| ISO 作成前の YAML 検証観点を見る | Agent-based Installer `1.10` | `install-config.yaml` / `agent-config.yaml` の必須項目の確認 |
| 非接続ミラーを Agent Installer に渡す方法を見る | Agent-based Installer `2.1`、`2.1.1` | `additionalTrustBundle`、pull secret、`imageDigestSources` の考え方 |
| 入力ファイルと ISO 生成の基本手順を見る | Agent-based Installer `3.2`、`3.3`、`3.4` | tools 取得、入力ファイル、Agent ISO 生成 |
| `install-config.yaml` / `agent-config.yaml` の推奨入力方式を見る | Agent-based Installer `4.4`、`4.6` | 本手順ではこの推奨方式を主導線にする |
| パラメーター仕様を見る | Agent-based Installer `9.1.1`、`9.1.2`、`9.1.3`、`9.2.1`、`9.2.2` | YAML の仕様確認用。作業手順そのものではない |
| SNO 以外の 3台 compact / HA 構成へ広げる | Agent-based Installer `1.1.1`、`1.1.2`、`1.4.1`、`1.5.2`、`1.6.1`、`1.6.2` | 本文の SNO 手順を直接置き換えるのではなく、DNS/LB/host 定義/replica 数を増やす補足として扱う |
| Quay 相当のミラーレジストリーを作る | Disconnected environments `4.1`、`4.2`、`4.4`、必要時 `4.8`、`4.9` | work EC2 上の mirror-registry / Quay 起動 |
| oc-mirror v2 で release と Operator をミラーする | Disconnected environments `5.1`、`5.1.1`、`5.3.1`、`5.3.2`、`5.4.1`、`5.4.2`、`5.4.3`、`5.4.4`、`5.5`、`5.5.2`、`5.12`、`5.13` | `ImageSetConfiguration`、`oc-mirror --v2`、生成リソース確認 |
| 今回は読まなくてよい | Agent-based Installer 5〜8章、Disconnected environments の更新/移行/OLM 詳細章 | 外部レジストリーなし方式、PXE、iSCSI、MCE、運用更新は本手順の範囲外 |

公式ドキュメントの章番号は HTML の更新で変わることがあるため、本手順では OpenShift Container Platform 4.21 の HTML で確認した `x.x` 節名と URL を併記する。章番号が変わった場合は、節名で追う。

### 0.2 目次、公式対応、参考ログとの関係

この表は、手順全体の流れ、公式ドキュメント上の対応箇所、参考ログとの関係を先に掴むためのもの。  
各作業章の冒頭にも、同じ観点を短く再掲する。

| 章 | 作業 | 主な公式対応 | 参考ログとの関係 |
|---:|---|---|---|
| 1 | 置換ルール | 本手順独自の運用ルール | 参考ログにはない。GitHub 公開と再実行性のために本手順で追加 |
| 2 | AWS リソース / Mac SSH / Cursor | Agent-based Installer `1.1.3` Supported platforms、`1.6` platform `none` 要件、`1.6.1` DNS、`1.6.2` LB 要件 | 参考ログも EC2 上で作業しているが、CloudFormation 化、work-key-only、SNO dummy Key pair なし、Mac SSH/Cursor 整理は本手順で追加 |
| 3 | work EC2 初期化 | Agent-based Installer `3.1` prerequisites、`3.2` tools download、`4.4` preferred configuration inputs | 参考ログも RHEL 作業ホストへ必要パッケージを導入している。本手順では作業ディレクトリと env 管理を整理 |
| 4 | DNS / NTP / firewall | Agent-based Installer `1.5.2` Static networking、`1.6.1` DNS requirements、`1.6.1.1` DNS example、`1.6.2` LB requirements | 参考ログは静的 DNS/default route 方針を YAML に反映。本手順では DNS/NTP/firewall の作成を前倒し |
| 5 | SNO dummy MAC 反映 | Agent-based Installer `1.4` Host configuration、`1.4.1` Host roles、`1.5.2` Static networking、`9.2.1`/`9.2.2` Agent config parameters | 参考ログも MAC を `agent-config.yaml` に明示。本手順では CloudFormation Outputs の `SnoMac` を採用 |
| 6 | OpenShift tools 取得 | Agent-based Installer `3.2` Downloading the Agent-based Installer、`4.2` Downloading the Agent-based Installer、`4.3` supported architecture | 参考ログも `oc`、`oc-mirror`、`openshift-install-fips` を配置。本手順では `latest` を避け固定 URL 化 |
| 7 | mirror-registry / Quay 起動 | Disconnected environments `4.1` prerequisites、`4.2` mirror registry introduction、`4.4` local host mirroring、必要時 `4.9` uninstall | 参考ログも work 上で mirror-registry/Quay を起動。本手順では `quay.ocp.lab.test:8443` に正規化し、認証情報保存ファイルは作らない |
| 8 | Quay CA / pull secret merge | Agent-based Installer `2.1`、`2.1.1` mirrored images 設定、Disconnected environments `5.3.2` credentials | 参考ログと同じく CA 信頼、Quay login、pull secret merge を実施。本手順では秘密情報の表示を抑制 |
| 9 | ImageSetConfiguration | Disconnected environments `5.1.1` high level workflow、`5.4.1` Creating the image set configuration、`5.12` ImageSet parameters | 参考ログも release + Operator + additionalImages を定義。本手順では `latest` タグを主導線から外し、必要時だけ明示タグで追加 |
| 10 | oc-mirror v2 実行 | Disconnected environments `5.4` Mirroring an image set、`5.4.2` workflow comparison、`5.5` generated resources、`5.13` command reference | 参考ログと同じく IDMS / ITMS / CatalogSource / ClusterCatalog を生成 |
| 11 | install-config.yaml 作成 | Agent-based Installer `2.1`、`2.1.1`、`4.4`、`9.1.1` required parameters、`9.1.2` network parameters、`9.1.3` optional parameters | 参考ログと同じく Quay CA、pull secret、ミラー設定、SNO 構成を反映。本手順では値を `ocp.lab.test` と `10.0.0.10/10.0.1.10` に正規化 |
| 12 | agent-config.yaml 作成 | Agent-based Installer `1.4`、`1.4.1`、`1.5.2`、`9.2.1` required parameters、`9.2.2` optional parameters | 参考ログと同じく MAC/IP/DNS/default route/rendezvousIP を明示。本手順では CloudFormation の SNO MAC を使用 |
| 13 | Agent ISO 生成 | Agent-based Installer `1.10` validation checks、`3.4` create image、`4.6` create image | 参考ログと同じく `install-config.yaml` と `agent-config.yaml` から Agent ISO を生成 |
| 14 | クラスター起動後の oc-mirror 生成リソース | Disconnected environments `5.5` generated resources、`5.5.1` restrictions、`5.5.2` configuring cluster to use generated resources | 参考ログでは生成リソースを確認。本手順ではクラスター起動後の適用候補として分離 |
| 15 | 最終成果物 | 本手順の成果物整理 | 参考ログと同じく ISO、kubeconfig、kubeadmin-password、rendezvousIP が生成される |
| 16 | 3台 compact / HA へ拡張する場合 | Agent-based Installer `1.1.1`、`1.1.2`、`1.4.1`、`1.5.2`、`1.6.1`、`1.6.2`、`9.1`、`9.2` | 参考ログは静的ネットワークの考え方を流用。本手順の SNO 固定値をそのまま増やすのではなく、DNS/LB/host 定義を拡張する |
| 17 | 注意点 | `platform: none`、非接続ミラー、Agent-based Installer 仕様の整理 | 参考ログから得た注意点を、公開用に一般化して記載 |
| 18 | 参照元 | 公式ドキュメントと参考ログ | 個人名を出さず、参考ログとして扱う |


---

## 1. 置換ルール

この手順では `<値>` のような表記は使わない。  
`REPLACE_ME_` で始まる値だけ、自分の環境に合わせて置き換える。

| 変数 | 入れる値 | 例 |
|---|---|---|
| `ADMIN_CIDR` | Mac など SSH 元のグローバル IP CIDR | `203.0.113.10/32` |
| `WORK_KEY_NAME` | work EC2 用 EC2 Key pair 名 | `ocp-airgap-lab` |
| `WORK_KEY_PATH` | Mac 上の work 用秘密鍵 | `$HOME/.ssh/ocp-airgap/ocp-airgap-lab.pem` |
| `WORK_PUBLIC_IP` | CloudFormation Outputs の `WorkPublicIp` | AWS 作成後に取得 |
| `SNO_MAC` | CloudFormation Outputs の `SnoMac` | AWS 作成後に取得 |
| `OC_MIRROR_VERSION` | 取得する `oc-mirror` の固定バージョン | `4.21.21` |
| `OC_MIRROR_URL` | 取得する `oc-mirror.rhel9.tar.gz` の固定URL | `https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.21.21/oc-mirror.rhel9.tar.gz` |

SNO dummy EC2 には Key pair を設定しない。  
SNO dummy は閉域ノードの IP / MAC / Security Group / routing を模擬するために立てるだけで、この手順では SNO dummy へ SSH ログインして作業しない。

**参考ログとの関係**

参考ログには、この公開用の置換ルール章はない。GitHub 公開時に秘密情報や個人名を混ぜないため、本手順で追加している。

---

## 2. AWS リソースを CloudFormation で払い出す

**公式対応**

- Agent-based Installer `1.1.3` Supported platforms  
  https://docs.redhat.com/ja/documentation/openshift_container_platform/4.21/html/installing_an_on-premise_cluster_with_the_agent-based_installer/preparing-to-install-with-agent-based-installer
- Agent-based Installer `1.6` Requirements for a cluster using the platform `none` option
- Agent-based Installer `1.6.1` Platform `none` DNS requirements
- Agent-based Installer `1.6.2` Platform `none` Load balancing requirements

AWS 自体は公式 Agent-based Installer の対象プラットフォームとして使わない。本手順では、AWS をオンプレ閉域のネットワーク、DNS、NTP、レジストリ到達性を模擬するための下回りとして使う。

**参考ログとの関係**

参考ログでも AWS 上の RHEL EC2 を作業端末兼ミラーレジストリとして使っている。  
本手順では、個別手作業ではなく CloudFormation で VPC / subnet / work EC2 / SNO dummy EC2 を払い出す。SNO dummy には Key pair を設定せず、Mac からは work EC2 のみへ SSH する。Mac の SSH 設定と Cursor 接続は、公開手順として本手順に統合している。

CloudFormation で以下を作成する。

| リソース | 値 |
|---|---|
| VPC | `10.0.0.0/16` |
| Public subnet | `10.0.0.0/24` |
| Private subnet | `10.0.1.0/24` |
| work EC2 | `ocp-airgap-work` / `10.0.0.10` / Public IP あり / SSH 用 Key pair あり |
| SNO dummy EC2 | `ocp-airgap-sno-master0-dummy` / `10.0.1.10` / Public IP なし / NAT なし / Key pair なし |
| Quay | `quay.ocp.lab.test:8443` on work |
| DNS / NTP | work `10.0.0.10` |
| API / API-int / Ingress | SNO `10.0.1.10` へ向ける |
| SNO default gateway | `10.0.1.1` |

Private subnet は NAT Gateway を持たない。SNO dummy は閉域ノードを模擬するため、外部インターネットへ出さない。

### 2.1 EC2 Key pair を作成する

AWS Console で実施する。

1. AWS Console で EC2 を開く
2. 左メニューから **Key Pairs** を開く
3. **Create key pair** を押す
4. 以下で作成する

| 項目 | 値 |
|---|---|
| Name | `ocp-airgap-lab` |
| Key pair type | `RSA` |
| Private key file format | `.pem` |

作成すると、Mac の Downloads に以下が保存される。

```text
~/Downloads/ocp-airgap-lab.pem
```

### 2.2 Mac 側で pem を SSH 用ディレクトリへ移動する

Mac 側で実行する。

```bash
mkdir -p "$HOME/.ssh/ocp-airgap"

mv "$HOME/Downloads/ocp-airgap-lab.pem" \
  "$HOME/.ssh/ocp-airgap/ocp-airgap-lab.pem"

chmod 700 "$HOME/.ssh"
chmod 700 "$HOME/.ssh/ocp-airgap"
chmod 600 "$HOME/.ssh/ocp-airgap/ocp-airgap-lab.pem"
```

確認する場合。

```bash
ls -l "$HOME/.ssh/ocp-airgap/ocp-airgap-lab.pem"
```

想定出力。

```text
-rw------- ... /Users/USER/.ssh/ocp-airgap/ocp-airgap-lab.pem
```

### 2.3 CloudFormation を作成する

CloudFormation テンプレートは以下を使う。

```text
ocp-airgap-aws-mock-cloudformation-v3-no-eip-work-key-only.yaml
```

AWS Console から作成する場合は、パラメーターを以下のようにする。

| パラメーター | 値 |
|---|---|
| `AdminCidr` | Mac の接続元グローバル IP `/32` |
| `WorkKeyName` | `ocp-airgap-lab` |

AWS CLI で作成する場合は、Mac または AWS CLI が使える端末で実行する。

```bash
aws cloudformation deploy \
  --stack-name ocp-airgap-aws-mock \
  --template-file ocp-airgap-aws-mock-cloudformation-v3-no-eip-work-key-only.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    AdminCidr=REPLACE_ME_ADMIN_GLOBAL_IP_CIDR \
    WorkKeyName=ocp-airgap-lab
```

この操作では AWS リソースが作成され、費用が発生する。  
`AdminCidr=0.0.0.0/0` はラボ利便性のためのデフォルトであり、通常は使わない。

### 2.4 CloudFormation Outputs を取得する

AWS CLI で確認する場合。

```bash
aws cloudformation describe-stacks \
  --stack-name ocp-airgap-aws-mock \
  --query "Stacks[0].Outputs[].[OutputKey,OutputValue]" \
  --output table
```

想定出力。

```text
--------------------------------------------------------------------------------
|                                DescribeStacks                                |
+----------------------+-------------------------------------------------------+
| WorkPublicIp         | 203.0.113.xxx                                         |
| WorkPrivateIp        | 10.0.0.10                                             |
| SnoPrivateIp         | 10.0.1.10                                             |
| SnoMac               | 06:xx:xx:xx:xx:xx                                     |
| WorkSshCommand       | ssh -i ./ocp-airgap-lab.pem ... ec2-user@203.0.113.xxx |
+----------------------+-------------------------------------------------------+
```

`WorkPublicIp` は Elastic IP ではない。work EC2 を stop/start すると変わる可能性がある。

### 2.5 Mac から work EC2 に SSH する

Mac 側で実行する。  
`REPLACE_ME_WORK_PUBLIC_IP` は CloudFormation Outputs の `WorkPublicIp` に置き換える。

```bash
export WORK_KEY_PATH="$HOME/.ssh/ocp-airgap/ocp-airgap-lab.pem"
export WORK_PUBLIC_IP="REPLACE_ME_WORK_PUBLIC_IP"

ssh -i "${WORK_KEY_PATH}" \
  -o IdentitiesOnly=yes \
  ec2-user@"${WORK_PUBLIC_IP}"
```

ログインできたら work EC2 側で確認する。

```bash
hostname
whoami
pwd
```

想定出力。

```text
work.ocp.lab.test
ec2-user
/home/ec2-user
```


### 2.6 Mac の SSH config と Cursor で work を開く

Mac 側で実行する。`REPLACE_ME_WORK_PUBLIC_IP` は CloudFormation Outputs の `WorkPublicIp` に置き換える。

```bash
touch "$HOME/.ssh/config"
chmod 600 "$HOME/.ssh/config"

cat >> "$HOME/.ssh/config" <<'EOF'

Host ocp-airgap-work
  HostName REPLACE_ME_WORK_PUBLIC_IP
  User ec2-user
  IdentityFile ~/.ssh/ocp-airgap/ocp-airgap-lab.pem
  IdentitiesOnly yes
  ServerAliveInterval 60
EOF
```

確認する場合。

```bash
ssh ocp-airgap-work
```

想定出力。

```text
[ec2-user@work ~]$
```

Cursor の Remote-SSH で開く場合は、接続先に `ocp-airgap-work` を選び、以下のフォルダを開く。

```text
/home/ec2-user/ocp-airgap
```

Mac で `cursor` コマンドが使える場合。

```bash
cursor --remote ssh-remote+ocp-airgap-work /home/ec2-user/ocp-airgap
```

想定状態。

```text
Cursor の Remote-SSH ウィンドウで /home/ec2-user/ocp-airgap が開く
```


---

## 3. 作業用 EC2 を初期化する

**公式対応**

- Agent-based Installer `3.1` Prerequisites for installing a cluster with the Agent-based Installer  
  https://docs.redhat.com/ja/documentation/openshift_container_platform/4.21/html/installing_an_on-premise_cluster_with_the_agent-based_installer/installing-with-agent-basic
- Agent-based Installer `3.2` Downloading the Agent-based Installer
- Agent-based Installer `4.4` Creating the preferred configuration inputs  
  https://docs.redhat.com/ja/documentation/openshift_container_platform/4.21/html/installing_an_on-premise_cluster_with_the_agent-based_installer/installing-with-agent-based-installer

公式手順はインストーラーと CLI の取得が中心で、作業用 RHEL ホストの DNS/NTP/Quay 用パッケージ導入は利用者側の準備として扱う。

**参考ログとの関係**

参考ログでも RHEL 作業ホストへ `podman`、`skopeo`、`firewalld`、`nmstate` などを導入している。  
本手順では、後続の YAML 生成に使う固定値を `config/env.sh` にまとめる。ただし各章で `source config/env.sh` を繰り返さず、この章で作業シェルに一度読み込む。

### 3.1 作業ディレクトリと環境変数ファイルを作る

work 側で実行する。  
`SNO_MAC` は後続章で CloudFormation Outputs の `SnoMac` に置き換える。

```bash
mkdir -p ~/ocp-airgap/{config,mirror-registry,oc-mirror-work,pkg}

cat > ~/ocp-airgap/config/env.sh <<'EOF'
export OCP_VERSION="4.21.21"

export VPC_CIDR="10.0.0.0/16"
export PUBLIC_SUBNET_CIDR="10.0.0.0/24"
export PRIVATE_SUBNET_CIDR="10.0.1.0/24"

export WORK_PRIVATE_IP="10.0.0.10"
export WORK_HOSTNAME="work.ocp.lab.test"

export QUAY_HOSTNAME="quay.ocp.lab.test"
export QUAY_PORT="8443"
export QUAY_FQDN="quay.ocp.lab.test:8443"

export CLUSTER_NAME="ocp"
export BASE_DOMAIN="lab.test"
export SNO_HOSTNAME="master-0.ocp.lab.test"
export SNO_PRIVATE_IP="10.0.1.10"
export SNO_NIC="eth0"
export SNO_MAC="REPLACE_ME_SNO_MAC_AFTER_AWS_CREATION"

export RENDEZVOUS_IP="10.0.1.10"
export DNS_IP="10.0.0.10"
export NTP_IP="10.0.0.10"
export DEFAULT_GATEWAY="10.0.1.1"

export UPSTREAM_DNS="10.0.0.2"
export UPSTREAM_NTP="169.254.169.123"

export OC_MIRROR_VERSION="4.21.21"
export OC_MIRROR_URL="https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.21.21/oc-mirror.rhel9.tar.gz"
EOF

cd ~/ocp-airgap
source config/env.sh
```

確認する場合。

```bash
grep -E '^(export OCP_VERSION|export CLUSTER_NAME|export BASE_DOMAIN|export QUAY_FQDN|export RENDEZVOUS_IP|export SNO_MAC)=' config/env.sh
```

想定出力。

```text
export OCP_VERSION="4.21.21"
export QUAY_FQDN="quay.ocp.lab.test:8443"
export CLUSTER_NAME="ocp"
export BASE_DOMAIN="lab.test"
export SNO_MAC="REPLACE_ME_SNO_MAC_AFTER_AWS_CREATION"
export RENDEZVOUS_IP="10.0.1.10"
```

### 3.2 OS パッケージを入れる

work 側で実行する。

```bash
sudo dnf clean all
sudo dnf install -y \
  openssl jq curl tar gzip \
  nmstate podman skopeo \
  firewalld tmux \
  bind bind-utils \
  chrony

sudo systemctl enable --now firewalld
sudo systemctl enable --now chronyd
```

確認する場合。

```bash
cat /etc/redhat-release
hostname
ip -br addr
rpm -q jq nmstate firewalld bind bind-utils chrony tmux
```

想定出力。

```text
Red Hat Enterprise Linux release 9.x
work.ocp.lab.test
eth0 UP 10.0.0.10/24 ...
jq-...
nmstate-...
firewalld-...
bind-...
bind-utils-...
chrony-...
tmux-...
```

---

## 4. DNS / NTP / firewall を構成する

**公式対応**

- Agent-based Installer `1.5` About networking  
  https://docs.redhat.com/ja/documentation/openshift_container_platform/4.21/html/installing_an_on-premise_cluster_with_the_agent-based_installer/preparing-to-install-with-agent-based-installer
- Agent-based Installer `1.5.2` Static networking
- Agent-based Installer `1.6.1` Platform `none` DNS requirements
- Agent-based Installer `1.6.1.1` Example DNS configuration for platform `none` clusters
- Agent-based Installer `1.6.2` Platform `none` Load balancing requirements

SNO では公式の platform `none` LB 要件の一部は適用外と明記されているが、DNS 正引き/逆引き、Quay 到達性、NTP、firewall、routing は事前に整える。

**参考ログとの関係**

参考ログでは、`agent-config.yaml` に DNS、default route、静的 IP を明示する方針を確認している。  
本手順では、その YAML を作る前に work 上の DNS / NTP / firewall を構成し、`platform: none` の前提となる名前解決と到達性を整える。

SNO でも `api.ocp.lab.test`、`api-int.ocp.lab.test`、`*.apps.ocp.lab.test`、`master-0.ocp.lab.test`、`quay.ocp.lab.test` を解決できる状態を先に作る。

### 4.1 hostname と hosts を設定する

work 側で実行する。

```bash
sudo hostnamectl set-hostname "${WORK_HOSTNAME}"

sudo cp -a /etc/hosts /etc/hosts.bak.$(date +%Y%m%d-%H%M%S)
sudo sed -i '/ocp\.lab\.test/d' /etc/hosts

sudo tee -a /etc/hosts >/dev/null <<EOF
${WORK_PRIVATE_IP} ${WORK_HOSTNAME} work ${QUAY_HOSTNAME} quay
${SNO_PRIVATE_IP} ${SNO_HOSTNAME} master-0 api.${CLUSTER_NAME}.${BASE_DOMAIN} api api-int.${CLUSTER_NAME}.${BASE_DOMAIN} api-int
EOF
```

確認する場合。

```bash
hostname -f
getent hosts "${QUAY_HOSTNAME}"
getent hosts "api.${CLUSTER_NAME}.${BASE_DOMAIN}"
```

想定出力。

```text
work.ocp.lab.test
10.0.0.10 quay.ocp.lab.test quay
10.0.1.10 master-0.ocp.lab.test master-0 api.ocp.lab.test api api-int.ocp.lab.test api-int
```

### 4.2 BIND を設定する

work 側で実行する。

```bash
sudo cp -a /etc/named.conf /etc/named.conf.bak.$(date +%Y%m%d-%H%M%S)

FORWARDERS_BLOCK=""
if [ -n "${UPSTREAM_DNS}" ]; then
  FORWARDERS_BLOCK="    forwarders { ${UPSTREAM_DNS}; };"
fi

sudo tee /etc/named.conf >/dev/null <<EOF
options {
    listen-on port 53 { 127.0.0.1; ${WORK_PRIVATE_IP}; };
    listen-on-v6 port 53 { none; };
    directory       "/var/named";
    dump-file       "/var/named/data/cache_dump.db";
    statistics-file "/var/named/data/named_stats.txt";
    memstatistics-file "/var/named/data/named_mem_stats.txt";
    secroots-file   "/var/named/data/named.secroots";
    recursing-file  "/var/named/data/named.recursing";
    allow-query     { localhost; ${VPC_CIDR}; };
    recursion yes;
    allow-recursion { localhost; ${VPC_CIDR}; };
${FORWARDERS_BLOCK}
    dnssec-validation no;
    pid-file "/run/named/named.pid";
    session-keyfile "/run/named/session.key";
};

logging {
    channel default_debug {
        file "data/named.run";
        severity dynamic;
    };
};

zone "${CLUSTER_NAME}.${BASE_DOMAIN}" IN {
    type master;
    file "${CLUSTER_NAME}.${BASE_DOMAIN}.zone";
    allow-update { none; };
};

zone "0.0.10.in-addr.arpa" IN {
    type master;
    file "0.0.10.in-addr.arpa.zone";
    allow-update { none; };
};

zone "1.0.10.in-addr.arpa" IN {
    type master;
    file "1.0.10.in-addr.arpa.zone";
    allow-update { none; };
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
EOF
```

```bash
sudo tee /var/named/${CLUSTER_NAME}.${BASE_DOMAIN}.zone >/dev/null <<EOF
\$TTL 300
@   IN SOA ${WORK_HOSTNAME}. root.${CLUSTER_NAME}.${BASE_DOMAIN}. (
        2026070401
        1H
        15M
        1W
        300 )
    IN NS ${WORK_HOSTNAME}.

work        IN A ${WORK_PRIVATE_IP}
quay        IN A ${WORK_PRIVATE_IP}

api         IN A ${SNO_PRIVATE_IP}
api-int     IN A ${SNO_PRIVATE_IP}
master-0    IN A ${SNO_PRIVATE_IP}
*.apps      IN A ${SNO_PRIVATE_IP}
EOF

sudo tee /var/named/0.0.10.in-addr.arpa.zone >/dev/null <<EOF
\$TTL 300
@   IN SOA ${WORK_HOSTNAME}. root.${CLUSTER_NAME}.${BASE_DOMAIN}. (
        2026070401
        1H
        15M
        1W
        300 )
    IN NS ${WORK_HOSTNAME}.

10 IN PTR ${WORK_HOSTNAME}.
EOF

sudo tee /var/named/1.0.10.in-addr.arpa.zone >/dev/null <<EOF
\$TTL 300
@   IN SOA ${WORK_HOSTNAME}. root.${CLUSTER_NAME}.${BASE_DOMAIN}. (
        2026070401
        1H
        15M
        1W
        300 )
    IN NS ${WORK_HOSTNAME}.

10 IN PTR ${SNO_HOSTNAME}.
10 IN PTR api.${CLUSTER_NAME}.${BASE_DOMAIN}.
10 IN PTR api-int.${CLUSTER_NAME}.${BASE_DOMAIN}.
EOF

sudo chown root:named \
  /var/named/${CLUSTER_NAME}.${BASE_DOMAIN}.zone \
  /var/named/0.0.10.in-addr.arpa.zone \
  /var/named/1.0.10.in-addr.arpa.zone

sudo chmod 640 \
  /var/named/${CLUSTER_NAME}.${BASE_DOMAIN}.zone \
  /var/named/0.0.10.in-addr.arpa.zone \
  /var/named/1.0.10.in-addr.arpa.zone

sudo restorecon -v \
  /var/named/${CLUSTER_NAME}.${BASE_DOMAIN}.zone \
  /var/named/0.0.10.in-addr.arpa.zone \
  /var/named/1.0.10.in-addr.arpa.zone
```

### 4.3 named を有効化し、work 自身もこの DNS を使う

work 側で実行する。

```bash
sudo named-checkconf /etc/named.conf
sudo named-checkzone "${CLUSTER_NAME}.${BASE_DOMAIN}" "/var/named/${CLUSTER_NAME}.${BASE_DOMAIN}.zone"
sudo named-checkzone 0.0.10.in-addr.arpa /var/named/0.0.10.in-addr.arpa.zone
sudo named-checkzone 1.0.10.in-addr.arpa /var/named/1.0.10.in-addr.arpa.zone

sudo systemctl enable --now named

DEV="$(ip route show default | awk '{print $5; exit}')"
CON="$(nmcli -t -f NAME,DEVICE con show --active | awk -F: -v dev="${DEV}" '$2==dev{print $1; exit}')"

sudo nmcli con mod "${CON}" ipv4.dns "${DNS_IP}" ipv4.ignore-auto-dns yes
sudo cp -a /etc/resolv.conf /etc/resolv.conf.bak.$(date +%Y%m%d-%H%M%S)
printf 'nameserver %s\n' "${DNS_IP}" | sudo tee /etc/resolv.conf
```

確認する場合。

```bash
dig "${QUAY_HOSTNAME}" +short
dig "api.${CLUSTER_NAME}.${BASE_DOMAIN}" +short
dig "api-int.${CLUSTER_NAME}.${BASE_DOMAIN}" +short
dig "${SNO_HOSTNAME}" +short
dig "console-openshift-console.apps.${CLUSTER_NAME}.${BASE_DOMAIN}" +short
dig -x "${WORK_PRIVATE_IP}" +short
dig -x "${SNO_PRIVATE_IP}" +short
```

想定出力。

```text
10.0.0.10
10.0.1.10
10.0.1.10
10.0.1.10
10.0.1.10
work.ocp.lab.test.
master-0.ocp.lab.test.
api.ocp.lab.test.
api-int.ocp.lab.test.
```

### 4.4 NTP と firewall を設定する

work 側で実行する。

```bash
sudo cp -a /etc/chrony.conf /etc/chrony.conf.bak.$(date +%Y%m%d-%H%M%S)

sudo sed -i '/^# ocp-airgap-ntp-start$/,/^# ocp-airgap-ntp-end$/d' /etc/chrony.conf
sudo sed -i '/^server 169\.254\.169\.123/d' /etc/chrony.conf
sudo sed -i '/^allow 10\.0\.0\.0\/16/d' /etc/chrony.conf
sudo sed -i '/^local stratum 10/d' /etc/chrony.conf

{
  echo '# ocp-airgap-ntp-start'
  if [ -n "${UPSTREAM_NTP}" ]; then
    echo "server ${UPSTREAM_NTP} prefer iburst"
  fi
  echo "allow ${VPC_CIDR}"
  echo "local stratum 10"
  echo '# ocp-airgap-ntp-end'
} | sudo tee -a /etc/chrony.conf >/dev/null

sudo systemctl enable --now chronyd
sudo systemctl restart chronyd

sudo firewall-cmd --permanent --add-service=dns
sudo firewall-cmd --permanent --add-service=ntp
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --permanent --add-port=${QUAY_PORT}/tcp
sudo firewall-cmd --reload
```

確認する場合。

```bash
systemctl is-active named
systemctl is-active chronyd
chronyc tracking
sudo firewall-cmd --list-all
```

想定出力。

```text
active
active
Reference ID    : ...
Leap status     : Normal
...
services: ... dns ntp ssh
ports: 8443/tcp
```

---

## 5. SNO dummy の MAC を反映する

**公式対応**

- Agent-based Installer `1.4` Host configuration  
  https://docs.redhat.com/ja/documentation/openshift_container_platform/4.21/html/installing_an_on-premise_cluster_with_the_agent-based_installer/preparing-to-install-with-agent-based-installer
- Agent-based Installer `1.4.1` Host roles
- Agent-based Installer `1.5.2` Static networking
- Agent-based Installer `9.2.1` Required Agent configuration parameters  
  https://docs.redhat.com/ja/documentation/openshift_container_platform/4.21/html/installing_an_on-premise_cluster_with_the_agent-based_installer/installation-config-parameters-agent
- Agent-based Installer `9.2.2` Optional Agent configuration parameters

`agent-config.yaml` では、設定対象ホストを NIC の MAC address で識別する。SNO dummy EC2 の MAC は作成ごとに変わるため、CloudFormation Outputs から取得して反映する。

**参考ログとの関係**

参考ログでも、対象ホストの NIC MAC を `agent-config.yaml` に明示している。  
本手順では SNO MAC を固定値として書かず、CloudFormation Outputs の `SnoMac` を `config/env.sh` に反映してから `agent-config.yaml` を生成する。

CloudFormation Outputs の `SnoMac` を `config/env.sh` に入れる。  
`REPLACE_ME_SNO_MAC_FROM_CLOUDFORMATION_OUTPUT` は実際の `SnoMac` に置き換える。

work 側で実行する。

```bash
export REAL_SNO_MAC="REPLACE_ME_SNO_MAC_FROM_CLOUDFORMATION_OUTPUT"

sed -i 's/^export SNO_NIC=.*/export SNO_NIC="eth0"/' config/env.sh
sed -i "s/^export SNO_MAC=.*/export SNO_MAC=\"${REAL_SNO_MAC}\"/" config/env.sh

source config/env.sh
```

確認する場合。

```bash
grep '^export SNO_' config/env.sh
```

想定出力。

```text
export SNO_HOSTNAME="master-0.ocp.lab.test"
export SNO_PRIVATE_IP="10.0.1.10"
export SNO_NIC="eth0"
export SNO_MAC="06:xx:xx:xx:xx:xx"
```

work から SNO dummy への TCP 到達性を見る場合だけ実行する。

```bash
timeout 3 bash -c '</dev/tcp/10.0.1.10/22'
```

想定出力。

```text
```

戻り値が `0` なら TCP 接続は成立している。何も表示されないのが正常である。

補足としてメッセージを出したい場合だけ、以下を使う。

```bash
timeout 3 bash -c '</dev/tcp/10.0.1.10/22' \
  && echo "OK: work can reach SNO dummy tcp/22" \
  || echo "NG: work cannot reach SNO dummy tcp/22"
```

---

## 6. OpenShift tools を取得する

**公式対応**

- Agent-based Installer `3.2` Downloading the Agent-based Installer  
  https://docs.redhat.com/ja/documentation/openshift_container_platform/4.21/html/installing_an_on-premise_cluster_with_the_agent-based_installer/installing-with-agent-basic
- Agent-based Installer `4.2` Downloading the Agent-based Installer  
  https://docs.redhat.com/ja/documentation/openshift_container_platform/4.21/html/installing_an_on-premise_cluster_with_the_agent-based_installer/installing-with-agent-based-installer
- Agent-based Installer `4.3` Verifying the supported architecture for an Agent-based installation

本手順では、`oc` / `kubectl` / `openshift-install-fips` を OpenShift `4.21.21` 固定で取得し、`oc-mirror` も固定 URL から取得する。

**参考ログとの関係**

参考ログでも `oc`、`oc-mirror`、`openshift-install-fips` を作業ホストへ配置している。  
本手順では GitHub 公開と再実行性を優先し、`latest` の自動追跡を避けて固定バージョンと固定 URL を使う。

`oc` / `kubectl` / `openshift-install-fips` は `OCP_VERSION` 固定で取得する。  
`oc-mirror` は `OC_MIRROR_URL` に明示した固定 URL から取得する。`latest` は使わない。

work 側で実行する。

```bash
pushd pkg >/dev/null

curl -fL -O "https://mirror.openshift.com/pub/openshift-v4/clients/ocp/${OCP_VERSION}/openshift-client-linux-amd64-rhel9.tar.gz"
curl -fL -O "https://mirror.openshift.com/pub/openshift-v4/clients/ocp/${OCP_VERSION}/openshift-install-rhel9-amd64.tar.gz"
curl -fL -o oc-mirror.rhel9.tar.gz "${OC_MIRROR_URL}"

mkdir -p "extract/${OCP_VERSION}/client" \
         "extract/${OCP_VERSION}/installer" \
         "extract/${OC_MIRROR_VERSION}/oc-mirror"

tar xzf openshift-client-linux-amd64-rhel9.tar.gz -C "extract/${OCP_VERSION}/client"
tar xzf openshift-install-rhel9-amd64.tar.gz -C "extract/${OCP_VERSION}/installer"
tar xzf oc-mirror.rhel9.tar.gz -C "extract/${OC_MIRROR_VERSION}/oc-mirror"

sudo install -m 0755 "extract/${OCP_VERSION}/client/oc" /usr/local/bin/oc
sudo install -m 0755 "extract/${OCP_VERSION}/client/kubectl" /usr/local/bin/kubectl
sudo install -m 0755 "extract/${OCP_VERSION}/installer/openshift-install-fips" /usr/local/bin/openshift-install-fips
sudo install -m 0755 "extract/${OC_MIRROR_VERSION}/oc-mirror/oc-mirror" /usr/local/bin/oc-mirror

popd >/dev/null
hash -r
```

確認する場合。

```bash
oc version --client
openshift-install-fips version
oc-mirror version || oc-mirror --version
which oc kubectl oc-mirror openshift-install-fips
```

想定出力。

```text
Client Version: 4.21.21
openshift-install-fips 4.21.21
oc-mirror version ...
/usr/local/bin/oc
/usr/local/bin/kubectl
/usr/local/bin/oc-mirror
/usr/local/bin/openshift-install-fips
```

---

## 7. mirror-registry / Quay を起動する

**公式対応**

- Disconnected environments `4.1` Prerequisites  
  https://docs.redhat.com/ja/documentation/openshift_container_platform/4.21/html/disconnected_environments/installing-mirroring-creating-registry
- Disconnected environments `4.2` Mirror registry for Red Hat OpenShift introduction
- Disconnected environments `4.4` Mirroring on a local host with mirror registry for Red Hat OpenShift
- Disconnected environments `4.8` Replacing mirror registry for Red Hat OpenShift SSL/TLS certificates
- Disconnected environments `4.9` Uninstalling the mirror registry for Red Hat OpenShift

本手順では work EC2 上に mirror-registry for Red Hat OpenShift / Quay をローカルホスト方式で構築し、`quay.ocp.lab.test:8443` として利用する。

**参考ログとの関係**

参考ログでも作業ホスト上に mirror-registry for Red Hat OpenShift / Quay を起動している。  
本手順では Quay FQDN を `quay.ocp.lab.test:8443` に正規化し、Quay 初期認証情報を追加の env ファイルに保存しない形へ整理している。

この手順ではラボ用の Quay 初期認証として `init` / `passwd123` を使う。共有環境や長期利用環境では強い値へ変更する。

### 7.1 mirror-registry tar を work へ転送する

Mac 側で実行する。

```bash
export WORK_PUBLIC_IP="REPLACE_ME_WORK_PUBLIC_IP"
export WORK_KEY_PATH="$HOME/.ssh/ocp-airgap/ocp-airgap-lab.pem"

scp -i "${WORK_KEY_PATH}" \
  "$HOME/Downloads/mirror-registry-amd64.tar.gz" \
  ec2-user@"${WORK_PUBLIC_IP}":/home/ec2-user/ocp-airgap/pkg/
```

想定出力。

```text
mirror-registry-amd64.tar.gz  100%  ...  MB/s  ...
```

### 7.2 work 側で展開する

work 側で実行する。

```bash
tar xzf pkg/mirror-registry-amd64.tar.gz -C mirror-registry
```

確認する場合。

```bash
ls -lh mirror-registry
```

想定出力。

```text
execution-environment.tar
image-archive.tar
mirror-registry
sqlite3.tar
```

### 7.3 Quay を起動する

work 側で実行する。

```bash
sudo firewall-cmd --permanent --add-port=${QUAY_PORT}/tcp
sudo firewall-cmd --reload

sudo cp -a /etc/hosts /etc/hosts.bak.$(date +%Y%m%d-%H%M%S)
sudo sed -i "/${QUAY_HOSTNAME//./\\.}/d" /etc/hosts
printf '%s %s quay\n' "${WORK_PRIVATE_IP}" "${QUAY_HOSTNAME}" | sudo tee -a /etc/hosts

pushd mirror-registry >/dev/null

./mirror-registry install \
  --quayHostname "${QUAY_HOSTNAME}" \
  --quayRoot "$HOME/ocp-airgap/mirror-registry/mirror" \
  --initUser init \
  --initPassword passwd123

popd >/dev/null
```

想定出力。

```text
Quay installed successfully
Quay is available at https://quay.ocp.lab.test:8443 with credentials (init, passwd123)
```

確認する場合。

```bash
podman ps
curl -k "https://${QUAY_FQDN}/health/instance"
curl -kI "https://${QUAY_FQDN}/v2/"
```

想定出力。

```text
quay-redis
quay-app
"status_code":200
HTTP/2 401
```

`/v2/` の `401` は、レジストリ API が認証を要求している状態であり、Quay が起動していることを示す。

### 7.4 Quay を作り直す場合だけ初期化する

既存の Quay を破棄して作り直す場合だけ実行する。`oc-mirror` 後は実行しない。

```bash
pushd mirror-registry >/dev/null

./mirror-registry uninstall \
  --quayRoot "$HOME/ocp-airgap/mirror-registry/mirror" \
  --autoApprove \
  -v || true

systemctl --user stop $(systemctl --user list-units --all --no-legend 2>/dev/null | awk '/quay|redis|pod|container/ {print $1}') 2>/dev/null || true
systemctl --user disable $(systemctl --user list-unit-files --no-legend 2>/dev/null | awk '/quay|redis|pod|container/ {print $1}') 2>/dev/null || true

rm -f ~/.config/systemd/user/*quay* \
      ~/.config/systemd/user/*redis* \
      ~/.config/systemd/user/pod-*.service \
      ~/.config/systemd/user/container-*.service

systemctl --user daemon-reload 2>/dev/null || true
systemctl --user reset-failed 2>/dev/null || true

podman pod rm -af 2>/dev/null || true
podman rm -af 2>/dev/null || true
podman volume rm -f $(podman volume ls -q) 2>/dev/null || true
podman secret rm $(podman secret ls --format '{{.Name}}') 2>/dev/null || true

podman system reset -f

popd >/dev/null
```

Quay のファイル領域も初期化する場合だけ実行する。

```bash
mv mirror-registry/mirror "mirror-registry/mirror.bak.$(date +%Y%m%d-%H%M%S)"
mkdir -p mirror-registry/mirror
```

---

## 8. Quay CA を信頼し、pull secret を merge する

**公式対応**

- Agent-based Installer `2.1` About mirroring the OpenShift Container Platform image repository for a disconnected registry  
  https://docs.redhat.com/ja/documentation/openshift_container_platform/4.21/html/installing_an_on-premise_cluster_with_the_agent-based_installer/understanding-disconnected-installation-mirroring
- Agent-based Installer `2.1.1` Configuring the Agent-based Installer to use mirrored images
- Disconnected environments `5.3.2` Configuring credentials that allow images to be mirrored  
  https://docs.redhat.com/ja/documentation/openshift_container_platform/4.21/html/disconnected_environments/about-installing-oc-mirror-v2

Quay CA は `additionalTrustBundle` とローカル trust store に関係し、Quay 認証は pull secret と `${XDG_RUNTIME_DIR}/containers/auth.json` に関係する。

**参考ログとの関係**

参考ログでも Quay CA を信頼させ、Quay へのログイン情報を pull secret に merge している。  
本手順でも同じことを行うが、公開手順として pull secret の中身を表示する確認は主導線に置かない。

### 8.1 Quay CA を work に信頼させる

work 側で実行する。

```bash
sudo cp mirror-registry/mirror/quay-rootCA/rootCA.pem /etc/pki/ca-trust/source/anchors/quay-ocp-lab-test.crt
sudo update-ca-trust extract

sudo mkdir -p "/etc/containers/certs.d/${QUAY_FQDN}"
sudo cp mirror-registry/mirror/quay-rootCA/rootCA.pem "/etc/containers/certs.d/${QUAY_FQDN}/ca.crt"
```

確認する場合。

```bash
curl "https://${QUAY_FQDN}/health/instance"
```

想定出力。

```text
{"status_code":200,...}
```

### 8.2 Quay にログインする

work 側で実行する。  
パスワード入力では 7.3 の値を入力する。

```bash
podman login "https://${QUAY_FQDN}" -u init
```

想定出力。

```text
Password:
Login Succeeded!
```

### 8.3 Red Hat pull secret を work へ転送する

Mac 側で実行する。

```bash
export WORK_PUBLIC_IP="REPLACE_ME_WORK_PUBLIC_IP"
export WORK_KEY_PATH="$HOME/.ssh/ocp-airgap/ocp-airgap-lab.pem"

scp -i "${WORK_KEY_PATH}" \
  "$HOME/Downloads/pull-secret.txt" \
  ec2-user@"${WORK_PUBLIC_IP}":/home/ec2-user/ocp-airgap/config/pull-secret.txt
```

想定出力。

```text
pull-secret.txt  100%  ...
```

### 8.4 pull secret に Quay 認証を merge する

work 側で実行する。

```bash
jq . config/pull-secret.txt > config/pull-secret.json
cp -a config/pull-secret.json config/pull-secret.original.json

AUTHFILE="${XDG_RUNTIME_DIR}/containers/auth.json"
REGISTRY="${QUAY_FQDN}"

jq --arg reg "${REGISTRY}" \
   --argjson entry "$(jq --arg reg "${REGISTRY}" '.auths[$reg]' "${AUTHFILE}")" \
   '.auths[$reg] = $entry' \
   config/pull-secret.json > config/pull-secret.merged.json

mv config/pull-secret.merged.json config/pull-secret.json
cp config/pull-secret.json "${AUTHFILE}"
chmod 600 config/pull-secret.json config/pull-secret.original.json "${AUTHFILE}"
```

確認する場合。

```bash
podman login --get-login "${QUAY_FQDN}"
jq -r '.auths | keys[]' config/pull-secret.json
```

想定出力。

```text
init
cloud.openshift.com
quay.io
quay.ocp.lab.test:8443
registry.connect.redhat.com
registry.redhat.io
```

---

## 9. ImageSetConfiguration を作成する

**公式対応**

- Disconnected environments `5.1.1` High level workflow  
  https://docs.redhat.com/ja/documentation/openshift_container_platform/4.21/html/disconnected_environments/about-installing-oc-mirror-v2
- Disconnected environments `5.4.1` Creating the image set configuration
- Disconnected environments `5.12` ImageSet configuration parameters for oc-mirror plugin v2
- Agent-based Installer `2.1.1` Configuring the Agent-based Installer to use mirrored images  
  https://docs.redhat.com/ja/documentation/openshift_container_platform/4.21/html/installing_an_on-premise_cluster_with_the_agent-based_installer/understanding-disconnected-installation-mirroring

`ImageSetConfiguration` は oc-mirror v2 が何を Quay へミラーするかを定義する。Operator は ISO に焼き込むのではなく、Quay へミラーして後続で CatalogSource / ClusterCatalog として扱う。

**参考ログとの関係**

参考ログでも release、Operator、additionalImages を `ImageSetConfiguration` に定義している。  
本手順では `additionalImages` の `latest` タグを主導線から外し、追加する場合は検証者が明示タグを選ぶ形にしている。

Operator は Agent ISO に焼き込まれない。ここでは Operator Catalog と関連イメージを Quay へ事前ミラーする。ISO には Quay CA、pull secret、ミラー参照設定が入る。

work 側で実行する。

```bash
cat > config/imageset-config.yaml <<'EOF'
apiVersion: mirror.openshift.io/v2alpha1
kind: ImageSetConfiguration
mirror:
  platform:
    architectures:
      - amd64
    channels:
      - name: stable-4.21
        type: ocp
        minVersion: "4.21.21"
        maxVersion: "4.21.21"
        shortestPath: true
    graph: false
  operators:
    - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.21
      packages:
        - name: kubernetes-nmstate-operator
          defaultChannel: stable
          channels:
            - name: stable
        - name: odf-operator
          defaultChannel: stable-4.21
          channels:
            - name: stable-4.21
        - name: redhat-oadp-operator
          defaultChannel: stable
          channels:
            - name: stable
        - name: kubevirt-hyperconverged
          defaultChannel: stable
          channels:
            - name: stable
        - name: mtv-operator
          defaultChannel: release-v2.11
          channels:
            - name: release-v2.11
        - name: cluster-logging
          defaultChannel: stable-6.5
          channels:
            - name: stable-6.5
        - name: cincinnati-operator
          defaultChannel: v1
          channels:
            - name: v1
        - name: nfd
          defaultChannel: stable
          channels:
            - name: stable
        - name: gpu-operator-certified
          defaultChannel: stable
          channels:
            - name: stable
EOF
```

追加イメージもミラーする場合だけ、明示タグで追記する。

```bash
cat >> config/imageset-config.yaml <<'EOF'
  additionalImages:
    - name: registry.redhat.io/rhel9/rhel-guest-image:REPLACE_ME_RHEL9_GUEST_IMAGE_TAG
EOF
```

確認する場合。

```bash
grep -E 'stable-4.21|redhat-operator-index:v4.21|kubernetes-nmstate-operator|odf-operator|kubevirt-hyperconverged|mtv-operator|cluster-logging|gpu-operator-certified' config/imageset-config.yaml
```

想定出力。

```text
      - name: stable-4.21
    - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.21
        - name: kubernetes-nmstate-operator
        - name: odf-operator
        - name: kubevirt-hyperconverged
        - name: mtv-operator
        - name: cluster-logging
        - name: gpu-operator-certified
```

---

## 10. oc-mirror v2 でミラーする

**公式対応**

- Disconnected environments `5.4` Mirroring an image set to a mirror registry  
  https://docs.redhat.com/ja/documentation/openshift_container_platform/4.21/html/disconnected_environments/about-installing-oc-mirror-v2
- Disconnected environments `5.4.2` Comparison of oc-mirror workflows
- Disconnected environments `5.5` About custom resources generated by oc-mirror plugin v2
- Disconnected environments `5.5.2` Configuring your cluster to use the resources generated by oc-mirror plugin v2
- Disconnected environments `5.13` Command reference for oc-mirror plugin v2

本手順では mirror-to-mirror 方式で、work EC2 から Quay に直接ミラーする。

**参考ログとの関係**

参考ログでも `oc-mirror --v2` により release image、Operator image、additional image を Quay へミラーし、IDMS / ITMS / CatalogSource / ClusterCatalog を生成している。  
本手順でも同じ流れを採用する。

処理が長いため tmux を使う。

work 側で実行する。

```bash
tmux new -s oc-mirror
```

tmux 内は新しいシェルなので、最初に作業ディレクトリと環境変数を読み込む。

```bash
cd ~/ocp-airgap
source config/env.sh

oc-mirror \
  --config config/imageset-config.yaml \
  --workspace file://oc-mirror-work \
  "docker://${QUAY_FQDN}" \
  --v2 \
  --image-timeout 30m
```

想定出力。

```text
[INFO]   : 👋 Hello, welcome to oc-mirror
[INFO]   : 🔀 workflow mode: mirrorToMirror
[INFO]   : Verified we can authenticate against registry "quay.ocp.lab.test:8443"
...
[INFO]   : idms-oc-mirror.yaml file created
[INFO]   : itms-oc-mirror.yaml file created
[INFO]   : cs-redhat-operator-index-v4-21.yaml file created
[INFO]   : cc-redhat-operator-index-v4-21.yaml file created
[INFO]   : 👋 Goodbye, thank you for using oc-mirror
```

tmux から抜ける場合は `Ctrl-b` の後に `d`。

確認する場合。

```bash
ls -lh oc-mirror-work/working-dir/cluster-resources
```

想定出力。

```text
idms-oc-mirror.yaml
itms-oc-mirror.yaml
cs-redhat-operator-index-v4-21.yaml
cc-redhat-operator-index-v4-21.yaml
signature-configmap.yaml
```

---

## 11. install-config.yaml を作成する

**公式対応**

- Agent-based Installer `2.1` About mirroring the OpenShift Container Platform image repository for a disconnected registry  
  https://docs.redhat.com/ja/documentation/openshift_container_platform/4.21/html/installing_an_on-premise_cluster_with_the_agent-based_installer/understanding-disconnected-installation-mirroring
- Agent-based Installer `2.1.1` Configuring the Agent-based Installer to use mirrored images
- Agent-based Installer `4.4` Creating the preferred configuration inputs  
  https://docs.redhat.com/ja/documentation/openshift_container_platform/4.21/html/installing_an_on-premise_cluster_with_the_agent-based_installer/installing-with-agent-based-installer
- Agent-based Installer `9.1.1` Required configuration parameters  
  https://docs.redhat.com/ja/documentation/openshift_container_platform/4.21/html/installing_an_on-premise_cluster_with_the_agent-based_installer/installation-config-parameters-agent
- Agent-based Installer `9.1.2` Network configuration parameters
- Agent-based Installer `9.1.3` Optional configuration parameters

`install-config.yaml` には `baseDomain`、cluster name、`platform: none`、SNO の replica 数、pull secret、Quay CA、oc-mirror 出力由来の `imageDigestSources` を入れる。

**参考ログとの関係**

参考ログでも `additionalTrustBundle`、pull secret、ミラー設定、SNO の `controlPlane replicas: 1` / `compute replicas: 0` を含む `install-config.yaml` を作成している。  
本手順ではクラスタ名、base domain、IP、Quay FQDN を公開用の正規化値に置き換えている。

oc-mirror v2 が生成する `ImageDigestMirrorSet` の `imageDigestMirrors` をもとに、`install-config.yaml` の `imageDigestSources` として反映する。

### 11.1 SSH 公開鍵と imageDigestSources を用意する

work 側で実行する。

```bash
awk '$1 ~ /^ssh-/ {print; exit}' ~/.ssh/authorized_keys > config/ssh.pub
chmod 600 config/ssh.pub

{
  printf 'imageDigestSources:\n'
  awk '
    /^  imageDigestMirrors:/ {capture=1; next}
    /^---/ {capture=0}
    capture && /^  - / {print}
    capture && /^    / {print}
  ' oc-mirror-work/working-dir/cluster-resources/idms-oc-mirror.yaml
} > config/imageDigestSources.from-idms.yaml
```

確認する場合。

```bash
cat config/imageDigestSources.from-idms.yaml
```

想定出力。

```text
imageDigestSources:
  - mirrors:
    - quay.ocp.lab.test:8443/openshift/release
    source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
  - mirrors:
    - quay.ocp.lab.test:8443/openshift/release-images
    source: quay.io/openshift-release-dev/ocp-release
...
```

### 11.2 install-config.yaml を生成する

work 側で実行する。

```bash
CA_FILE="$HOME/ocp-airgap/mirror-registry/mirror/quay-rootCA/rootCA.pem"
PULL_SECRET_JSON="$(jq -c . config/pull-secret.json)"
SSH_KEY="$(cat config/ssh.pub)"

cat > config/install-config.yaml <<EOF
apiVersion: v1
baseDomain: ${BASE_DOMAIN}
metadata:
  name: ${CLUSTER_NAME}
additionalTrustBundlePolicy: Always
additionalTrustBundle: |
$(sed 's/^/  /' "${CA_FILE}")
networking:
  networkType: OVNKubernetes
  machineNetwork:
    - cidr: ${VPC_CIDR}
  clusterNetwork:
    - cidr: 10.128.0.0/14
      hostPrefix: 23
  serviceNetwork:
    - 172.30.0.0/16
platform:
  none: {}
pullSecret: |
  ${PULL_SECRET_JSON}
$(cat config/imageDigestSources.from-idms.yaml)
sshKey: '${SSH_KEY}'
controlPlane:
  name: master
  replicas: 1
  platform: {}
compute:
  - name: worker
    replicas: 0
    platform: {}
EOF
```

確認する場合。pull secret は表示しない。

```bash
awk '
  /^pullSecret: \|/ {print "pullSecret: |"; print "  MASKED"; inps=1; next}
  inps && /^[^ ]/ {inps=0}
  inps {next}
  {print}
' config/install-config.yaml | sed -n '1,160p'
```

想定出力。

```text
apiVersion: v1
baseDomain: lab.test
metadata:
  name: ocp
additionalTrustBundlePolicy: Always
additionalTrustBundle: |
  -----BEGIN CERTIFICATE-----
  ...
networking:
  networkType: OVNKubernetes
  machineNetwork:
    - cidr: 10.0.0.0/16
platform:
  none: {}
pullSecret: |
  MASKED
imageDigestSources:
  - mirrors:
    - quay.ocp.lab.test:8443/openshift/release
...
controlPlane:
  name: master
  replicas: 1
compute:
  - name: worker
    replicas: 0
```

補足として、`openshift-install-fips agent create image` で `metadata.name` や `baseDomain` が空と出た場合は、`config/env.sh` を読み込み直したうえで 11.2 を再実行する。

```bash
source config/env.sh
```

---

## 12. agent-config.yaml を作成する

**公式対応**

- Agent-based Installer `1.4` Host configuration  
  https://docs.redhat.com/ja/documentation/openshift_container_platform/4.21/html/installing_an_on-premise_cluster_with_the_agent-based_installer/preparing-to-install-with-agent-based-installer
- Agent-based Installer `1.4.1` Host roles
- Agent-based Installer `1.5.2` Static networking
- Agent-based Installer `9.2.1` Required Agent configuration parameters  
  https://docs.redhat.com/ja/documentation/openshift_container_platform/4.21/html/installing_an_on-premise_cluster_with_the_agent-based_installer/installation-config-parameters-agent
- Agent-based Installer `9.2.2` Optional Agent configuration parameters

`agent-config.yaml` には SNO host、role、MAC、静的 IP、DNS、default route、rendezvousIP を明示する。

**参考ログとの関係**

参考ログでも `rendezvousIP`、hostname、role、interface name、MAC、静的 IP、DNS、default route を明示している。  
本手順でも同じ方針を採用し、SNO MAC だけは CloudFormation Outputs から取得した値を使う。

静的 NMState として、SNO の MAC、IP、DNS、default route、rendezvousIP を明示する。

work 側で実行する。

```bash
cat > config/agent-config.yaml <<EOF
apiVersion: v1alpha1
kind: AgentConfig
metadata:
  name: ${CLUSTER_NAME}
rendezvousIP: ${RENDEZVOUS_IP}
hosts:
  - hostname: ${SNO_HOSTNAME}
    role: master
    interfaces:
      - name: ${SNO_NIC}
        macAddress: ${SNO_MAC}
    networkConfig:
      interfaces:
        - name: ${SNO_NIC}
          type: ethernet
          state: up
          identifier: mac-address
          mac-address: ${SNO_MAC}
          ipv4:
            enabled: true
            address:
              - ip: ${SNO_PRIVATE_IP}
                prefix-length: 24
            dhcp: false
      dns-resolver:
        config:
          server:
            - ${DNS_IP}
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: ${DEFAULT_GATEWAY}
            next-hop-interface: ${SNO_NIC}
            table-id: 254
EOF
```

確認する場合。

```bash
cat config/agent-config.yaml
```

想定出力。

```text
apiVersion: v1alpha1
kind: AgentConfig
metadata:
  name: ocp
rendezvousIP: 10.0.1.10
hosts:
  - hostname: master-0.ocp.lab.test
    role: master
    interfaces:
      - name: eth0
        macAddress: 06:xx:xx:xx:xx:xx
...
              - ip: 10.0.1.10
                prefix-length: 24
...
            - 10.0.0.10
...
            next-hop-address: 10.0.1.1
            next-hop-interface: eth0
```

補足として、`SNO_MAC` が `REPLACE_ME_SNO_MAC_AFTER_AWS_CREATION` のままなら、第5章の `SnoMac` 反映後にこの章を再実行する。

---

## 13. Agent ISO を生成する

**公式対応**

- Agent-based Installer `1.10` Validation checks before agent ISO creation  
  https://docs.redhat.com/ja/documentation/openshift_container_platform/4.21/html/installing_an_on-premise_cluster_with_the_agent-based_installer/preparing-to-install-with-agent-based-installer
- Agent-based Installer `3.4` Creating and booting the agent image  
  https://docs.redhat.com/ja/documentation/openshift_container_platform/4.21/html/installing_an_on-premise_cluster_with_the_agent-based_installer/installing-with-agent-basic
- Agent-based Installer `4.6` Creating and booting the agent image  
  https://docs.redhat.com/ja/documentation/openshift_container_platform/4.21/html/installing_an_on-premise_cluster_with_the_agent-based_installer/installing-with-agent-based-installer

`openshift-install-fips agent create image --dir iso-create` が `install-config.yaml` と `agent-config.yaml` を読み、Agent ISO を生成する。

**参考ログとの関係**

参考ログでも `install-config.yaml` と `agent-config.yaml` を ISO 生成ディレクトリへコピーし、`openshift-install-fips agent create image` で ISO を生成している。  
本手順でも同じ流れを採用する。

work 側で実行する。

```bash
mkdir -p iso-create

cp config/install-config.yaml iso-create/install-config.yaml
cp config/agent-config.yaml iso-create/agent-config.yaml

openshift-install-fips agent create image --dir iso-create
```

想定出力。

```text
INFO Configuration has 1 master replicas, 0 arbiter replicas, and 0 worker replicas
INFO The rendezvous host IP (node0 IP) is 10.0.1.10
INFO Extracting base ISO from release payload
INFO Consuming Agent Config from target directory
INFO Consuming Install Config from target directory
INFO Generated ISO at agent.x86_64.iso.
```

確認する場合。

```bash
ls -lh iso-create
find iso-create -maxdepth 2 -type f -printf '%p %s bytes\n' | sort
cat iso-create/rendezvousIP
```

想定出力。

```text
agent.x86_64.iso
auth/kubeadmin-password
auth/kubeconfig
rendezvousIP
...
10.0.1.10
```

この時点で Agent ISO 生成は完了。

補足として、同じ `iso-create` を使って ISO を作り直す場合は、既存ディレクトリを残さず別名退避してから再作成する。

```bash
mv iso-create "iso-create.bak.$(date +%Y%m%d-%H%M%S)"
mkdir -p iso-create
```

---

## 14. クラスター起動後に使う oc-mirror 生成リソース

**公式対応**

- Disconnected environments `5.5` About custom resources generated by oc-mirror plugin v2  
  https://docs.redhat.com/ja/documentation/openshift_container_platform/4.21/html/disconnected_environments/about-installing-oc-mirror-v2
- Disconnected environments `5.5.1` Restrictions on modifying resources that are generated by the oc-mirror plugin
- Disconnected environments `5.5.2` Configuring your cluster to use the resources generated by oc-mirror plugin v2

`idms-oc-mirror.yaml`、`itms-oc-mirror.yaml`、CatalogSource、ClusterCatalog、signature configmap は、クラスター起動後に適用対象になる。

**参考ログとの関係**

参考ログでは oc-mirror 生成リソースの存在確認まで実施している。  
本手順では、クラスターが起動してから適用する候補として章を分け、ISO 生成の主導線とは分離する。

Agent ISO 生成時点ではクラスターがまだ存在しないため、以下はクラスター起動後に使う。

```bash
export KUBECONFIG=iso-create/auth/kubeconfig

oc apply -f oc-mirror-work/working-dir/cluster-resources/idms-oc-mirror.yaml
oc apply -f oc-mirror-work/working-dir/cluster-resources/itms-oc-mirror.yaml
oc apply -f oc-mirror-work/working-dir/cluster-resources/cs-redhat-operator-index-v4-21.yaml
oc apply -f oc-mirror-work/working-dir/cluster-resources/cc-redhat-operator-index-v4-21.yaml
oc apply -f oc-mirror-work/working-dir/cluster-resources/signature-configmap.yaml
```

想定出力。

```text
imagedigestmirrorset.config.openshift.io/... created
imagetagmirrorset.config.openshift.io/... created
catalogsource.operators.coreos.com/... created
clustercatalog.olm.operatorframework.io/... created
configmap/... created
```

注意: `idms-oc-mirror.yaml` 相当は、ISO 生成前の `install-config.yaml` に `imageDigestSources` として反映済みである。インストール後に IDMS / ITMS を適用するかは、実際のクラスター状態を確認してから決める。

---

## 15. 最終成果物

| 成果物 | パス |
|---|---|
| Agent ISO | `~/ocp-airgap/iso-create/agent.x86_64.iso` |
| kubeconfig | `~/ocp-airgap/iso-create/auth/kubeconfig` |
| kubeadmin password | `~/ocp-airgap/iso-create/auth/kubeadmin-password` |
| rendezvousIP | `~/ocp-airgap/iso-create/rendezvousIP` |
| install-config 原本 | `~/ocp-airgap/config/install-config.yaml` |
| agent-config 原本 | `~/ocp-airgap/config/agent-config.yaml` |
| oc-mirror 生成リソース | `~/ocp-airgap/oc-mirror-work/working-dir/cluster-resources/` |

---


## 16. SNO から 3台 compact / HA 構成へ拡張する場合

**公式対応**

- Agent-based Installer `1.1.1` Agent-based Installer のワークフロー  
  https://docs.redhat.com/ja/documentation/openshift_container_platform/4.21/html/installing_an_on-premise_cluster_with_the_agent-based_installer/preparing-to-install-with-agent-based-installer
- Agent-based Installer `1.1.2` 各トポロジーに推奨されるリソース
- Agent-based Installer `1.4.1` Host roles
- Agent-based Installer `1.5.2` Static networking
- Agent-based Installer `1.6.1` Platform `none` DNS requirements
- Agent-based Installer `1.6.2` Platform `none` Load balancing requirements
- Agent-based Installer `9.1` install-config parameters / `9.2` agent-config parameters  
  https://docs.redhat.com/ja/documentation/openshift_container_platform/4.21/html/installing_an_on-premise_cluster_with_the_agent-based_installer/installation-config-parameters-agent

**参考ログとの関係**

参考ログでは、静的 IP、MAC、DNS、default route、rendezvousIP を `agent-config.yaml` に明示する方針を確認している。  
3台 compact / HA でもこの考え方は同じだが、SNO の 1台分を単純にコピーするのではなく、全ノード分の host 定義、DNS 正引き/逆引き、API / Ingress の LB 相当を増やす。

### 16.1 API LB IP / Ingress LB IP の考え方

3台 compact や HA 構成の `api.ocp.lab.test` / `api-int.ocp.lab.test` は、master ノードのどれか 1台を仮置きで指す値ではない。  
事前に用意した API Load Balancer のフロント IP、または VIP を指す。

同様に、`*.apps.ocp.lab.test` は Application Ingress Load Balancer のフロント IP、または VIP を指す。  
ラボでは API LB と Ingress LB を同じ HAProxy / 同じ IP にまとめてもよいが、本番相当では分離してもよい。

| DNS 名 | 向ける先 | 意味 |
|---|---|---|
| `api.ocp.lab.test` | API LB のフロント IP / VIP | 外部クライアントとクラスターノードが Kubernetes API に到達する入口 |
| `api-int.ocp.lab.test` | API LB のフロント IP / VIP | クラスター内部通信で使う API 入口 |
| `*.apps.ocp.lab.test` | Ingress LB のフロント IP / VIP | Router / Ingress Controller への入口 |
| `master-0.ocp.lab.test` など | 各 master ノード IP | 各ノードそのもの |
| `worker-0.ocp.lab.test` など | 各 worker ノード IP | 各ノードそのもの |

SNO の場合だけは例外的に、API / API-int / Ingress を SNO ノード自身の IP に向ける。  
公式の platform `none` load balancing requirements でも、SNO にはこの LB 要件は適用されないとされているため、本手順の SNO 検証では `api` / `api-int` / `*.apps` を `10.0.1.10` に向けている。

3台 compact / HA では、Agent ISO を起動する前に、次のような LB 相当の設定を済ませておく。

| LB | フロント | バックエンド |
|---|---|---|
| API LB | `API_LB_IP:6443` | master 全台の `:6443` |
| Machine Config Server | `API_LB_IP:22623` | master 全台の `:22623` |
| Ingress LB | `INGRESS_LB_IP:80` / `:443` | worker 全台。3台 compact で worker 0 の場合は master 全台 |

API LB は Layer 4 のロードバランシングにする。API LB にはセッション永続性を設定しない。  
Ingress LB は Layer 4 のロードバランシングにし、アプリケーション要件に応じて connection / session based persistence を検討する。

AWS 模擬で専用ロードバランサーを増やさない場合は、work EC2 上に HAProxy を置き、work EC2 の private IP、または追加 ENI / 追加 private IP を API LB / Ingress LB のフロント IP として使う構成にできる。  
この場合も、その IP は「仮置き」ではなく、DNS が向き、各ノードから到達でき、HAProxy が実際に master / worker へ振り分ける実体のあるフロント IP として扱う。

### 16.2 3台 compact cluster の変更点

3台 compact cluster は、master 3台、worker 0台の構成である。master 3台が control plane と compute の両方を兼ねる。

`install-config.yaml` の replica 数は以下の考え方にする。

```yaml
controlPlane:
  name: master
  replicas: 3
  platform: {}
compute:
  - name: worker
    replicas: 0
    platform: {}
```

`agent-config.yaml` の `hosts` は master 3台分を書く。  
`rendezvousIP` は master 3台のうち 1台、通常は `master-0` の IP にする。

```yaml
rendezvousIP: 10.0.1.10
hosts:
  - hostname: master-0.ocp.lab.test
    role: master
    interfaces:
      - name: eth0
        macAddress: REPLACE_ME_MASTER0_MAC
    networkConfig:
      interfaces:
        - name: eth0
          type: ethernet
          state: up
          identifier: mac-address
          mac-address: REPLACE_ME_MASTER0_MAC
          ipv4:
            enabled: true
            address:
              - ip: 10.0.1.10
                prefix-length: 24
            dhcp: false
      dns-resolver:
        config:
          server:
            - 10.0.0.10
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 10.0.1.1
            next-hop-interface: eth0
            table-id: 254

  - hostname: master-1.ocp.lab.test
    role: master
    interfaces:
      - name: eth0
        macAddress: REPLACE_ME_MASTER1_MAC
    networkConfig:
      interfaces:
        - name: eth0
          type: ethernet
          state: up
          identifier: mac-address
          mac-address: REPLACE_ME_MASTER1_MAC
          ipv4:
            enabled: true
            address:
              - ip: 10.0.1.11
                prefix-length: 24
            dhcp: false
      dns-resolver:
        config:
          server:
            - 10.0.0.10
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 10.0.1.1
            next-hop-interface: eth0
            table-id: 254

  - hostname: master-2.ocp.lab.test
    role: master
    interfaces:
      - name: eth0
        macAddress: REPLACE_ME_MASTER2_MAC
    networkConfig:
      interfaces:
        - name: eth0
          type: ethernet
          state: up
          identifier: mac-address
          mac-address: REPLACE_ME_MASTER2_MAC
          ipv4:
            enabled: true
            address:
              - ip: 10.0.1.12
                prefix-length: 24
            dhcp: false
      dns-resolver:
        config:
          server:
            - 10.0.0.10
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 10.0.1.1
            next-hop-interface: eth0
            table-id: 254
```

DNS は以下の考え方にする。

```text
api.ocp.lab.test       A  API_LB_IP
api-int.ocp.lab.test   A  API_LB_IP
*.apps.ocp.lab.test    A  INGRESS_LB_IP

master-0.ocp.lab.test  A  10.0.1.10
master-1.ocp.lab.test  A  10.0.1.11
master-2.ocp.lab.test  A  10.0.1.12
```

3台 compact は worker 0 のため、Ingress Controller pod は control plane nodes 上で動く。  
そのため Ingress LB の `80` / `443` は master 3台へ振り分ける。

### 16.3 3 master + 複数 worker の HA 構成の変更点

3 master + 複数 worker の HA 構成では、`controlPlane.replicas` を `3` にし、`compute.replicas` を worker 台数にする。

worker 2台の例。

```yaml
controlPlane:
  name: master
  replicas: 3
  platform: {}
compute:
  - name: worker
    replicas: 2
    platform: {}
```

`agent-config.yaml` には master 3台分と worker 台数分の `hosts` を書く。  
静的ネットワークで確実に進める場合、各 host に NIC MAC、IP、DNS、default route を明示する。

DNS は以下の考え方にする。

```text
api.ocp.lab.test       A  API_LB_IP
api-int.ocp.lab.test   A  API_LB_IP
*.apps.ocp.lab.test    A  INGRESS_LB_IP

master-0.ocp.lab.test  A  10.0.1.10
master-1.ocp.lab.test  A  10.0.1.11
master-2.ocp.lab.test  A  10.0.1.12

worker-0.ocp.lab.test  A  10.0.1.20
worker-1.ocp.lab.test  A  10.0.1.21
```

LB は以下の考え方にする。

| ポート | フロント | バックエンド |
|---:|---|---|
| `6443` | API LB | master 3台 |
| `22623` | API LB | master 3台 |
| `80` | Ingress LB | worker 全台 |
| `443` | Ingress LB | worker 全台 |

worker を持つ HA 構成では、Ingress Controller pod はデフォルトで compute / worker 側に配置されるため、Ingress LB は worker 側へ振り分ける。

### 16.4 SNO から増やすときに変えるもの

SNO から 3台 compact / HA へ広げるときは、YAML の replica 数だけではなく、以下をまとめて変更する。

| 変更対象 | SNO | 3台 compact | 3 master + worker |
|---|---|---|---|
| `controlPlane.replicas` | `1` | `3` | `3` |
| `compute.replicas` | `0` | `0` | worker 台数 |
| `agent-config.yaml hosts` | 1台分 | master 3台分 | master 3台 + worker 台数分 |
| `rendezvousIP` | SNO IP | master 1台の IP | master 1台の IP |
| 必要な MAC | SNO 1台分 | master 3台分 | master 3台 + worker 台数分 |
| DNS A/PTR | SNO 1台分 | master 3台分 + API/Ingress LB | master/worker 全台分 + API/Ingress LB |
| API LB | SNO では不要 | 必要 | 必要 |
| Ingress LB | SNO では不要 | 必要。backend は master | 必要。backend は worker |

SNO 手順の `api` / `api-int` / `*.apps` を SNO IP に向ける設定は、3台 compact / HA へはそのまま流用しない。  
3台 compact / HA では、`api` / `api-int` は API LB のフロント IP / VIP、`*.apps` は Ingress LB のフロント IP / VIP へ向ける。

### 16.5 AWS 模擬で HA 検証に広げる場合

現在の CloudFormation は SNO dummy 1台を前提にしている。3台 compact / HA を AWS 模擬で検証する場合は、CloudFormation 側も次のように拡張する。

- SNO dummy ではなく master dummy 3台を作る。
- HA 構成なら worker dummy も必要台数作る。
- Outputs に各 master / worker の private IP と MAC を出す。
- work EC2 で BIND の A/PTR レコードを全ノード分に増やす。
- work EC2 または別 EC2 に HAProxy を置き、API LB / Ingress LB のフロント IP を持たせる。
- Security Group で、LB フロントから master / worker の必要ポートへ到達できるようにする。
- `agent-config.yaml` の `hosts` を全ノード分に増やす。
- Agent ISO を作り直す。

AWS 模擬で work EC2 を HAProxy 兼 DNS / NTP / Quay として使うことはできるが、その場合は work EC2 が単一障害点になる。  
この手順は閉域検証用であり、本番相当の HA 可用性を証明する構成ではない。

---


## 17. 注意点

Operator は Agent ISO に焼き込まれない。Operator 関連イメージと Catalog は Quay に事前ミラーされ、ISO には Quay を参照するための CA、pull secret、ミラー参照設定が入る。

`platform: none` では、DNS、API / Ingress の到達性、必要に応じた LB 相当、ノード間通信、Quay 到達性、NTP、firewall、routing を利用者側で用意する。SNO 検証では `api` / `api-int` / `*.apps` を同じ SNO IP へ向けている。

SNO dummy EC2 を作り直した場合、MAC が変わる。`config/env.sh` の `SNO_MAC` を更新し、`agent-config.yaml` と Agent ISO を作り直す。

実オンプレのベアメタルで複数ディスクが見える場合は、`agent-config.yaml` に `rootDeviceHints` を追加するか検討する。単一ディスク前提の AWS 模擬検証では省略している。

`oc-mirror` は `latest` を直接ダウンロードして使わない。公開 URL から検証者が使うバージョンを選び、`OC_MIRROR_VERSION` と `OC_MIRROR_URL` を `config/env.sh` に記録して再実行性を確保する。

`ImageSetConfiguration` の `additionalImages` でも、手順書の主導線では `latest` タグを使わない。必要な追加イメージがある場合は、検証者が明示タグを選んで追記する。

---

## 18. 参照元

### 18.1 公式ドキュメント

- OpenShift Container Platform 4.21: Installing an on-premise cluster with the Agent-based Installer  
  https://docs.redhat.com/ja/documentation/openshift_container_platform/4.21/html/installing_an_on-premise_cluster_with_the_agent-based_installer/index

- Agent-based Installer `1.1` Understanding Agent-based Installer / `1.1.1` workflow / `1.1.2` recommended resources / `1.1.3` supported platforms  
  https://docs.redhat.com/ja/documentation/openshift_container_platform/4.21/html/installing_an_on-premise_cluster_with_the_agent-based_installer/preparing-to-install-with-agent-based-installer

- Agent-based Installer `1.4` Host configuration / `1.4.1` Host roles / `1.4.2` root device hints  
  https://docs.redhat.com/ja/documentation/openshift_container_platform/4.21/html/installing_an_on-premise_cluster_with_the_agent-based_installer/preparing-to-install-with-agent-based-installer

- Agent-based Installer `1.5` About networking / `1.5.2` Static networking  
  https://docs.redhat.com/ja/documentation/openshift_container_platform/4.21/html/installing_an_on-premise_cluster_with_the_agent-based_installer/preparing-to-install-with-agent-based-installer

- Agent-based Installer `1.6` platform `none` requirements / `1.6.1` DNS requirements / `1.6.1.1` DNS example / `1.6.2` load balancing requirements  
  https://docs.redhat.com/ja/documentation/openshift_container_platform/4.21/html/installing_an_on-premise_cluster_with_the_agent-based_installer/preparing-to-install-with-agent-based-installer

- Agent-based Installer `1.10` Validation checks before agent ISO creation  
  https://docs.redhat.com/ja/documentation/openshift_container_platform/4.21/html/installing_an_on-premise_cluster_with_the_agent-based_installer/preparing-to-install-with-agent-based-installer

- Agent-based Installer `2.1` About mirroring the OpenShift Container Platform image repository for a disconnected registry / `2.1.1` Configuring the Agent-based Installer to use mirrored images  
  https://docs.redhat.com/ja/documentation/openshift_container_platform/4.21/html/installing_an_on-premise_cluster_with_the_agent-based_installer/understanding-disconnected-installation-mirroring

- Agent-based Installer `3.2` Downloading the Agent-based Installer / `3.3` Creating the configuration inputs / `3.4` Creating and booting the agent image  
  https://docs.redhat.com/ja/documentation/openshift_container_platform/4.21/html/installing_an_on-premise_cluster_with_the_agent-based_installer/installing-with-agent-basic

- Agent-based Installer `4.4` Creating the preferred configuration inputs / `4.6` Creating and booting the agent image  
  https://docs.redhat.com/ja/documentation/openshift_container_platform/4.21/html/installing_an_on-premise_cluster_with_the_agent-based_installer/installing-with-agent-based-installer

- Agent-based Installer `9.1.1` Required install-config parameters / `9.1.2` Network parameters / `9.1.3` Optional parameters / `9.2.1` Required agent-config parameters / `9.2.2` Optional agent-config parameters  
  https://docs.redhat.com/ja/documentation/openshift_container_platform/4.21/html/installing_an_on-premise_cluster_with_the_agent-based_installer/installation-config-parameters-agent

- OpenShift Container Platform 4.21: Disconnected environments  
  https://docs.redhat.com/ja/documentation/openshift_container_platform/4.21/html/disconnected_environments/index

- Disconnected environments `4.1` Prerequisites / `4.2` mirror registry introduction / `4.4` local host mirroring / `4.8` TLS certificates / `4.9` uninstall  
  https://docs.redhat.com/ja/documentation/openshift_container_platform/4.21/html/disconnected_environments/installing-mirroring-creating-registry

- Disconnected environments `5.1` About oc-mirror plugin v2 / `5.1.1` workflow / `5.3.1` install plugin / `5.3.2` credentials / `5.4.1` ImageSetConfiguration / `5.4` mirroring / `5.5` generated resources / `5.12` parameters / `5.13` command reference  
  https://docs.redhat.com/ja/documentation/openshift_container_platform/4.21/html/disconnected_environments/about-installing-oc-mirror-v2

### 18.2 参考ログ

- 参考ログは、閉域想定で `install-config.yaml` と `agent-config.yaml` を作成し、静的ネットワーク、MAC、IP、DNS、default route、rendezvousIP を明示する流れを確認するために使った。
- 参考ログ内の IP、FQDN、MAC、認証情報、個人情報は本手順へ直接転記しない。
- GitHub 公開版では、個人名、メールアドレス、秘密情報、環境固有の値は記載しない。

### 18.3 本手順で採用した方針

- AWS はオンプレ閉域の模擬環境として使う。
- OpenShift の `platform` は `none` とする。
- work EC2 は DNS / NTP / Quay / oc-mirror / Agent ISO 生成を担う。
- SNO dummy EC2 は閉域ノードの IP / MAC / Security Group / routing を模擬するためだけに使う。
- SNO dummy EC2 には Public IP、NAT route、Key pair を設定しない。
- Operator は Agent ISO に焼き込まれない。Operator 関連イメージと Catalog を Quay へ事前ミラーし、ISO には Quay CA、pull secret、ミラー参照設定を入れる。
