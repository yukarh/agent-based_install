# OpenShift 4.21 Agent-based Installer エアギャップ AWS模擬検証手順書

対象: OpenShift Container Platform 4.21.21 / Agent-based Installer / `platform: none` / SNO / AWS閉域模擬 / mirror-registry for Red Hat OpenShift / Quay / oc-mirror v2 / Operator込みミラー  
版: v3  
方針: 今回の実行ログを根拠に、再実行しやすい順番へ整理した手順書。ログ取得用コマンドは載せず、作業に必要な設定コマンドは省略しない。

---

## 0. この手順書の読み方

この手順書では、今回の実検証で後から実施した DNS / NTP / firewall を、再実行時に迷わないよう前半へ移動している。

今回ログで実際に払い出された IP は `10.0.0.17` / `10.0.1.169` だったが、本手順では再利用しやすいように以下へ正規化している。

| 用途 | 本手順の値 |
|---|---|
| 作業用 EC2 private IP | `10.0.0.10` |
| SNO 想定 EC2 private IP | `10.0.1.10` |
| DNS / NTP / Quay | `10.0.0.10` |
| rendezvousIP | `10.0.1.10` |

SNO の MAC アドレスは EC2 を作るたびに変わるため、本手順では固定値にしない。AWS で SNO 想定 EC2 を作成した後に取得し、`config/env.sh` の `SNO_MAC` に入れる。

---

## 1. 自分で置き換える箇所

### 1.1 置換ルール

この手順では `<値>` のような表記は使わない。

`REPLACE_ME_` で始まる値だけ、自分の環境に合わせて置き換える。  
シングルクォート `'...'` やダブルクォート `"..."` はシェル構文なので残す。中身だけ置き換える。

例:

```bash
export ADMIN_CIDR="REPLACE_ME_ADMIN_GLOBAL_IP_CIDR"
```

自分の接続元 IP が `203.0.113.10` の場合はこうする。

```bash
export ADMIN_CIDR="203.0.113.10/32"
```

### 1.2 置き換えが必要な値

| 変数 | 入れる値 | 例 |
|---|---|---|
| `AWS_REGION` | AWS リージョン | `ap-northeast-1` |
| `ADMIN_CIDR` | Mac など SSH 元のグローバル IP CIDR | `203.0.113.10/32` |
| `WORK_KEY_NAME` | work EC2 用 Key pair 名 | `ocp-airgap` |
| `SNO_KEY_NAME` | SNO dummy EC2 用 Key pair 名 | `ocp-airgap-2` |
| `RHEL_AMI_ID` | RHEL 9 x86_64 AMI ID | `ami-xxxxxxxxxxxxxxxxx` |
| `WORK_KEY_PATH` | Mac 上の work 用秘密鍵 | `$HOME/.ssh/ocp-airgap/ocp-airgap.pem` |
| `SNO_KEY_PATH` | Mac 上の SNO 用秘密鍵 | `$HOME/.ssh/ocp-airgap/ocp-airgap-2.pem` |
| `WORK_PUBLIC_IP` | work EC2 の Public IP | AWS 作成後に取得 |
| `SNO_MAC` | SNO EC2 の eth0 MAC | AWS 作成後に取得 |

---

## 2. 参照対応表

| 手順 | 作業 | 三井田さんログでの対応 | 成功手順 md での対応 | 公式ドキュメントでの対応 |
|---:|---|---|---|---|
| 3 | AWS リソース払い出し | AWS 上の RHEL EC2 を作業端末兼ミラーレジストリとして利用 | AWS 検証構成 | Agent-based Installer 第1章、特に 1.7 `platform: none` 要件 |
| 4 | 作業用 EC2 初期化 | RHEL へ podman / skopeo / firewalld / nmstate などを導入 | 作業ディレクトリ、必要パッケージ導入 | Agent-based Installer 第3章 3.1、3.2 |
| 5 | DNS / NTP / firewall | 静的 IP、DNS、default route を YAML に明示する方針 | 今回ログでは後追加。本手順では前倒し | Agent-based Installer 第1章 1.6、1.7.1、1.7.2 |
| 6 | SNO NIC / MAC 確認 | `agent-config.yaml` で MAC / IP / DNS / default route / rendezvousIP を明示 | agent-config 作成前確認 | Agent-based Installer 第1章 1.5、1.6.2、1.11 |
| 7 | tools 取得 | `oc`、`openshift-install-fips` を使用 | OpenShift tools 取得 | Agent-based Installer 第3章 3.2、第4章 4.2 |
| 8 | mirror-registry / Quay 起動 | mirror-registry を展開し Quay を `:8443` で起動 | mirror-registry 展開、Quay 起動 | Disconnected environments: mirror registry for Red Hat OpenShift |
| 9 | Quay CA / pull secret | Quay へログインし pull secret にローカル認証を追加 | Quay CA/login、pull secret merge | Agent-based Installer 第2章 2.2.1 |
| 10 | ImageSetConfiguration | release + Operator + additionalImages を指定 | ImageSetConfiguration | Disconnected environments: oc-mirror plugin v2 |
| 11 | oc-mirror v2 | IDMS / ITMS / CatalogSource / ClusterCatalog を生成 | oc-mirror 実行、生成リソース確認 | Agent-based Installer 第2章、Disconnected environments: oc-mirror plugin v2 |
| 12 | install-config.yaml | Quay CA、pull secret、ミラー参照、SNO 構成を反映 | install-config 作成 | Agent-based Installer 第4章 4.4、第9章 9.1 |
| 13 | agent-config.yaml | NMState 静的 IP / DNS / default route / rendezvousIP を明示 | agent-config 作成 | Agent-based Installer 第1章 1.5、1.6.2、第9章 9.2 |
| 14 | Agent ISO 生成 | `openshift-install-fips agent create image` | Agent ISO 生成 | Agent-based Installer 第3章 3.4、第4章 4.6 |

---

## 3. AWS リソースを払い出す

対応:

- 三井田さんログ: EC2 上の RHEL を作業端末兼ミラーレジストリとして利用。
- 公式 docs: Agent-based Installer 第1章 `platform: none` では、DNS、負荷分散相当、ノード間通信、ミラーレジストリ到達性を利用者側で用意する。
- 本手順: CloudFormation テンプレートは使わず、md 内に必要な払い出し内容と AWS CLI 例を記載する。

### 3.1 作成する構成

| リソース | 値 |
|---|---|
| VPC | `10.0.0.0/16` |
| Public subnet | `10.0.0.0/24` |
| Private subnet | `10.0.1.0/24` |
| work EC2 | `ocp-airgap-work` / `10.0.0.10` / Public IP あり |
| SNO dummy EC2 | `ocp-airgap-sno-master0-dummy` / `10.0.1.10` / Public IP なし / NAT なし |
| Quay | `quay.ocp.lab.test:8443` on work |
| DNS / NTP | work `10.0.0.10` |
| API / API-int / Ingress | SNO `10.0.1.10` へ向ける |
| SNO default gateway | `10.0.1.1` |

Private subnet は NAT Gateway を持たない。SNO dummy は閉域ノードを模擬するため、外部インターネットへ出さない。

### 3.2 AWS CLI 用変数を設定する

このブロックは Mac または AWS CLI が使える端末で実行する。  
`REPLACE_ME_` の値だけ置き換える。

```bash
export AWS_REGION="ap-northeast-1"
export ADMIN_CIDR="REPLACE_ME_ADMIN_GLOBAL_IP_CIDR"
export WORK_KEY_NAME="REPLACE_ME_WORK_KEY_PAIR_NAME"
export SNO_KEY_NAME="REPLACE_ME_SNO_KEY_PAIR_NAME"
export RHEL_AMI_ID="REPLACE_ME_RHEL9_X86_64_AMI_ID"

export WORK_INSTANCE_TYPE="m6i.2xlarge"
export SNO_INSTANCE_TYPE="m6i.2xlarge"

export VPC_CIDR="10.0.0.0/16"
export PUBLIC_SUBNET_CIDR="10.0.0.0/24"
export PRIVATE_SUBNET_CIDR="10.0.1.0/24"

export WORK_PRIVATE_IP="10.0.0.10"
export SNO_PRIVATE_IP="10.0.1.10"
```

### 3.3 VPC / subnet / route を作る

```bash
VPC_ID="$(aws ec2 create-vpc \
  --region "${AWS_REGION}" \
  --cidr-block "${VPC_CIDR}" \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=ocp-airgap-vpc}]' \
  --query 'Vpc.VpcId' \
  --output text)"

aws ec2 modify-vpc-attribute --region "${AWS_REGION}" --vpc-id "${VPC_ID}" --enable-dns-support '{"Value":true}'
aws ec2 modify-vpc-attribute --region "${AWS_REGION}" --vpc-id "${VPC_ID}" --enable-dns-hostnames '{"Value":true}'

IGW_ID="$(aws ec2 create-internet-gateway \
  --region "${AWS_REGION}" \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=ocp-airgap-igw}]' \
  --query 'InternetGateway.InternetGatewayId' \
  --output text)"

aws ec2 attach-internet-gateway --region "${AWS_REGION}" --vpc-id "${VPC_ID}" --internet-gateway-id "${IGW_ID}"

PUBLIC_SUBNET_ID="$(aws ec2 create-subnet \
  --region "${AWS_REGION}" \
  --vpc-id "${VPC_ID}" \
  --cidr-block "${PUBLIC_SUBNET_CIDR}" \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=ocp-airgap-public-subnet}]' \
  --query 'Subnet.SubnetId' \
  --output text)"

PRIVATE_SUBNET_ID="$(aws ec2 create-subnet \
  --region "${AWS_REGION}" \
  --vpc-id "${VPC_ID}" \
  --cidr-block "${PRIVATE_SUBNET_CIDR}" \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=ocp-airgap-private-subnet}]' \
  --query 'Subnet.SubnetId' \
  --output text)"

aws ec2 modify-subnet-attribute --region "${AWS_REGION}" --subnet-id "${PUBLIC_SUBNET_ID}" --map-public-ip-on-launch

PUBLIC_RT_ID="$(aws ec2 create-route-table \
  --region "${AWS_REGION}" \
  --vpc-id "${VPC_ID}" \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=ocp-airgap-public-rt}]' \
  --query 'RouteTable.RouteTableId' \
  --output text)"

aws ec2 create-route --region "${AWS_REGION}" --route-table-id "${PUBLIC_RT_ID}" --destination-cidr-block "0.0.0.0/0" --gateway-id "${IGW_ID}"
aws ec2 associate-route-table --region "${AWS_REGION}" --route-table-id "${PUBLIC_RT_ID}" --subnet-id "${PUBLIC_SUBNET_ID}"
```

### 3.4 Security Group を作る

```bash
WORK_SG_ID="$(aws ec2 create-security-group \
  --region "${AWS_REGION}" \
  --vpc-id "${VPC_ID}" \
  --group-name "ocp-airgap-work-sg" \
  --description "ocp airgap work security group" \
  --query 'GroupId' \
  --output text)"

SNO_SG_ID="$(aws ec2 create-security-group \
  --region "${AWS_REGION}" \
  --vpc-id "${VPC_ID}" \
  --group-name "ocp-airgap-sno-sg" \
  --description "ocp airgap sno dummy security group" \
  --query 'GroupId' \
  --output text)"

aws ec2 authorize-security-group-ingress --region "${AWS_REGION}" --group-id "${WORK_SG_ID}" --ip-permissions "IpProtocol=tcp,FromPort=22,ToPort=22,IpRanges=[{CidrIp=${ADMIN_CIDR}}]"
aws ec2 authorize-security-group-ingress --region "${AWS_REGION}" --group-id "${WORK_SG_ID}" --ip-permissions "IpProtocol=tcp,FromPort=8443,ToPort=8443,IpRanges=[{CidrIp=${VPC_CIDR}}]"
aws ec2 authorize-security-group-ingress --region "${AWS_REGION}" --group-id "${WORK_SG_ID}" --ip-permissions "IpProtocol=tcp,FromPort=53,ToPort=53,IpRanges=[{CidrIp=${VPC_CIDR}}]"
aws ec2 authorize-security-group-ingress --region "${AWS_REGION}" --group-id "${WORK_SG_ID}" --ip-permissions "IpProtocol=udp,FromPort=53,ToPort=53,IpRanges=[{CidrIp=${VPC_CIDR}}]"
aws ec2 authorize-security-group-ingress --region "${AWS_REGION}" --group-id "${WORK_SG_ID}" --ip-permissions "IpProtocol=udp,FromPort=123,ToPort=123,IpRanges=[{CidrIp=${VPC_CIDR}}]"

aws ec2 authorize-security-group-ingress --region "${AWS_REGION}" --group-id "${SNO_SG_ID}" --ip-permissions "IpProtocol=tcp,FromPort=22,ToPort=22,UserIdGroupPairs=[{GroupId=${WORK_SG_ID}}]"
aws ec2 authorize-security-group-ingress --region "${AWS_REGION}" --group-id "${SNO_SG_ID}" --ip-permissions "IpProtocol=tcp,FromPort=80,ToPort=80,UserIdGroupPairs=[{GroupId=${WORK_SG_ID}}]"
aws ec2 authorize-security-group-ingress --region "${AWS_REGION}" --group-id "${SNO_SG_ID}" --ip-permissions "IpProtocol=tcp,FromPort=443,ToPort=443,UserIdGroupPairs=[{GroupId=${WORK_SG_ID}}]"
aws ec2 authorize-security-group-ingress --region "${AWS_REGION}" --group-id "${SNO_SG_ID}" --ip-permissions "IpProtocol=tcp,FromPort=6443,ToPort=6443,UserIdGroupPairs=[{GroupId=${WORK_SG_ID}}]"
aws ec2 authorize-security-group-ingress --region "${AWS_REGION}" --group-id "${SNO_SG_ID}" --ip-permissions "IpProtocol=tcp,FromPort=22623,ToPort=22623,UserIdGroupPairs=[{GroupId=${WORK_SG_ID}}]"
```

### 3.5 EC2 を起動する

```bash
WORK_INSTANCE_ID="$(aws ec2 run-instances \
  --region "${AWS_REGION}" \
  --image-id "${RHEL_AMI_ID}" \
  --instance-type "${WORK_INSTANCE_TYPE}" \
  --key-name "${WORK_KEY_NAME}" \
  --subnet-id "${PUBLIC_SUBNET_ID}" \
  --private-ip-address "${WORK_PRIVATE_IP}" \
  --security-group-ids "${WORK_SG_ID}" \
  --block-device-mappings '[{"DeviceName":"/dev/sda1","Ebs":{"VolumeSize":500,"VolumeType":"gp3","DeleteOnTermination":true}}]' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=ocp-airgap-work}]' \
  --query 'Instances[0].InstanceId' \
  --output text)"

SNO_INSTANCE_ID="$(aws ec2 run-instances \
  --region "${AWS_REGION}" \
  --image-id "${RHEL_AMI_ID}" \
  --instance-type "${SNO_INSTANCE_TYPE}" \
  --key-name "${SNO_KEY_NAME}" \
  --subnet-id "${PRIVATE_SUBNET_ID}" \
  --private-ip-address "${SNO_PRIVATE_IP}" \
  --security-group-ids "${SNO_SG_ID}" \
  --block-device-mappings '[{"DeviceName":"/dev/sda1","Ebs":{"VolumeSize":120,"VolumeType":"gp3","DeleteOnTermination":true}}]' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=ocp-airgap-sno-master0-dummy}]' \
  --query 'Instances[0].InstanceId' \
  --output text)"

aws ec2 wait instance-running --region "${AWS_REGION}" --instance-ids "${WORK_INSTANCE_ID}" "${SNO_INSTANCE_ID}"

ALLOC_ID="$(aws ec2 allocate-address \
  --region "${AWS_REGION}" \
  --domain vpc \
  --query 'AllocationId' \
  --output text)"

aws ec2 associate-address --region "${AWS_REGION}" --instance-id "${WORK_INSTANCE_ID}" --allocation-id "${ALLOC_ID}"

WORK_PUBLIC_IP="$(aws ec2 describe-instances \
  --region "${AWS_REGION}" \
  --instance-ids "${WORK_INSTANCE_ID}" \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text)"

SNO_MAC="$(aws ec2 describe-instances \
  --region "${AWS_REGION}" \
  --instance-ids "${SNO_INSTANCE_ID}" \
  --query 'Reservations[0].Instances[0].NetworkInterfaces[0].MacAddress' \
  --output text)"

printf 'WORK_INSTANCE_ID=%s\nSNO_INSTANCE_ID=%s\nWORK_PUBLIC_IP=%s\nSNO_MAC=%s\n' \
  "${WORK_INSTANCE_ID}" "${SNO_INSTANCE_ID}" "${WORK_PUBLIC_IP}" "${SNO_MAC}"
```

### 3.6 まとめて確認

```bash
aws ec2 describe-instances \
  --region "${AWS_REGION}" \
  --instance-ids "${WORK_INSTANCE_ID}" "${SNO_INSTANCE_ID}" \
  --query 'Reservations[].Instances[].{Name:Tags[?Key==`Name`]|[0].Value,PrivateIp:PrivateIpAddress,PublicIp:PublicIpAddress,Mac:NetworkInterfaces[0].MacAddress,State:State.Name}' \
  --output table
```

出力例:

```text
ocp-airgap-work                 10.0.0.10   x.x.x.x      xx:xx:xx:xx:xx:xx   running
ocp-airgap-sno-master0-dummy    10.0.1.10   None         yy:yy:yy:yy:yy:yy   running
```

---

## 4. 作業用 EC2 を初期化する

対応:

- 三井田さんログ: RHEL 上に必要パッケージを導入。
- 成功手順 md: 作業ディレクトリ作成、OS パッケージ導入。
- 公式 docs: Agent-based Installer 第3章 3.1、3.2。

### 4.1 Mac から work に入る

Mac 側で実行する。`REPLACE_ME_WORK_PUBLIC_IP` は AWS 作成後の Public IP に置き換える。

```bash
export WORK_KEY_PATH="$HOME/.ssh/ocp-airgap/ocp-airgap.pem"
export WORK_PUBLIC_IP="REPLACE_ME_WORK_PUBLIC_IP"

chmod 400 "${WORK_KEY_PATH}"

ssh -i "${WORK_KEY_PATH}" -o IdentitiesOnly=yes ec2-user@"${WORK_PUBLIC_IP}"
```

### 4.2 作業ディレクトリと環境変数ファイルを作る

work 側で実行する。`SNO_MAC` だけ AWS で取得した MAC に置き換える。クォートは残す。

```bash
mkdir -p ~/ocp-airgap/{config,iso-create,logs,mirror-registry,oc-mirror-work,pkg}
chmod 700 ~/ocp-airgap/logs

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
EOF

source ~/ocp-airgap/config/env.sh
```

`SNO_MAC` を後で修正する場合は、work 側でこうする。

```bash
source ~/ocp-airgap/config/env.sh

sed -i "s/^export SNO_MAC=.*/export SNO_MAC=\"${SNO_MAC}\"/" ~/ocp-airgap/config/env.sh
source ~/ocp-airgap/config/env.sh
```

### 4.3 OS パッケージを入れる

この段階では work がインターネットへ出るために AWS Resolver が使える必要がある。VPC の DNS support / DNS hostnames を無効にしない。

```bash
cd ~/ocp-airgap
source config/env.sh

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

### 4.4 まとめて確認

```bash
cat /etc/redhat-release
hostname
ip -br addr
podman --version
skopeo --version
rpm -q jq nmstate firewalld bind bind-utils chrony tmux
```

出力例:

```text
Red Hat Enterprise Linux release 9.x
eth0 UP 10.0.0.10/24
podman version ...
skopeo version ...
```

---

## 5. DNS / NTP / firewall を先に構成する

対応:

- 三井田さんログ: `agent-config.yaml` で静的 IP / DNS / default route を明示する方針。
- 成功手順 md: 今回ログでは後続で実施した DNS / NTP / firewall を、本手順では前倒し。
- 公式 docs: Agent-based Installer 第1章 1.6 ネットワーク概要、1.7.1 DNS 要件、1.7.2 負荷分散要件。

SNO でも `api.ocp.lab.test`、`api-int.ocp.lab.test`、`*.apps.ocp.lab.test`、`master-0.ocp.lab.test`、`quay.ocp.lab.test` を解決できる状態を先に作る。

### 5.1 hostname と hosts を設定する

```bash
cd ~/ocp-airgap
source config/env.sh

sudo hostnamectl set-hostname "${WORK_HOSTNAME}"

sudo cp -a /etc/hosts /etc/hosts.bak.$(date +%Y%m%d-%H%M%S)
sudo sed -i '/ocp\.lab\.test/d' /etc/hosts

sudo tee -a /etc/hosts >/dev/null <<EOF
${WORK_PRIVATE_IP} ${WORK_HOSTNAME} work ${QUAY_HOSTNAME} quay
${SNO_PRIVATE_IP} ${SNO_HOSTNAME} master-0 api.${CLUSTER_NAME}.${BASE_DOMAIN} api api-int.${CLUSTER_NAME}.${BASE_DOMAIN} api-int
EOF
```

### 5.2 BIND を設定する

```bash
cd ~/ocp-airgap
source config/env.sh

sudo cp -a /etc/named.conf /etc/named.conf.bak.$(date +%Y%m%d-%H%M%S)

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
    forwarders { 10.0.0.2; };
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
cd ~/ocp-airgap
source config/env.sh

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

10  IN PTR ${WORK_HOSTNAME}.
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

### 5.3 named を有効化し、work 自身もこの DNS を使う

```bash
cd ~/ocp-airgap
source config/env.sh

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

### 5.4 NTP と firewall を設定する

```bash
cd ~/ocp-airgap
source config/env.sh

sudo cp -a /etc/chrony.conf /etc/chrony.conf.bak.$(date +%Y%m%d-%H%M%S)

sudo sed -i '/^server 169\.254\.169\.123/d' /etc/chrony.conf
sudo sed -i '/^allow 10\.0\.0\.0\/16/d' /etc/chrony.conf
sudo sed -i '/^local stratum 10/d' /etc/chrony.conf

sudo tee -a /etc/chrony.conf >/dev/null <<EOF
server 169.254.169.123 prefer iburst
allow ${VPC_CIDR}
local stratum 10
EOF

sudo systemctl enable --now chronyd
sudo systemctl restart chronyd

sudo firewall-cmd --permanent --add-service=dns
sudo firewall-cmd --permanent --add-service=ntp
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --permanent --add-port=${QUAY_PORT}/tcp
sudo firewall-cmd --reload
```

### 5.5 まとめて確認

```bash
cd ~/ocp-airgap
source config/env.sh

hostname -f
cat /etc/resolv.conf

dig work.${CLUSTER_NAME}.${BASE_DOMAIN} +short
dig ${QUAY_HOSTNAME} +short
dig api.${CLUSTER_NAME}.${BASE_DOMAIN} +short
dig api-int.${CLUSTER_NAME}.${BASE_DOMAIN} +short
dig ${SNO_HOSTNAME} +short
dig console-openshift-console.apps.${CLUSTER_NAME}.${BASE_DOMAIN} +short
dig -x ${WORK_PRIVATE_IP} +short
dig -x ${SNO_PRIVATE_IP} +short

systemctl is-active named
chronyc tracking
sudo firewall-cmd --list-all
```

出力例:

```text
work.ocp.lab.test
nameserver 10.0.0.10
10.0.0.10
10.0.0.10
10.0.1.10
10.0.1.10
10.0.1.10
10.0.1.10
work.ocp.lab.test.
master-0.ocp.lab.test.
active
Leap status     : Normal
services: cockpit dhcpv6-client dns ntp ssh
ports: 8443/tcp
```

---

## 6. SNO dummy から DNS / Quay / NTP 到達性を確認する

対応:

- 三井田さんログ: `agent-config.yaml` へ入れる MAC / interface / 静的 IP / DNS / route を確認する工程。
- 公式 docs: Agent-based Installer 第1章 1.5 ホストの設定、1.6.2 静的ネットワーキング。

SNO dummy は Agent ISO で起動しない。ここでは閉域ノードから work 上の DNS / NTP / Quay へ届くことを確認する。

### 6.1 Mac から SNO dummy へ入る

Mac 側で実行する。`WORK_PUBLIC_IP` は AWS 作成後の値にする。

```bash
export WORK_PUBLIC_IP="REPLACE_ME_WORK_PUBLIC_IP"
export WORK_KEY_PATH="$HOME/.ssh/ocp-airgap/ocp-airgap.pem"
export SNO_KEY_PATH="$HOME/.ssh/ocp-airgap/ocp-airgap-2.pem"

chmod 400 "${WORK_KEY_PATH}" "${SNO_KEY_PATH}"

ssh -i "${SNO_KEY_PATH}" \
  -o IdentitiesOnly=yes \
  -o "ProxyCommand=ssh -i ${WORK_KEY_PATH} -o IdentitiesOnly=yes -W %h:%p ec2-user@${WORK_PUBLIC_IP}" \
  ec2-user@10.0.1.10
```

### 6.2 SNO dummy 側で確認する

```bash
sudo cp -a /etc/resolv.conf /etc/resolv.conf.bak.$(date +%Y%m%d-%H%M%S)
printf 'nameserver 10.0.0.10\n' | sudo tee /etc/resolv.conf

ip -br link
ip -br addr
ip route

getent hosts work.ocp.lab.test
getent hosts quay.ocp.lab.test
getent hosts api.ocp.lab.test
getent hosts api-int.ocp.lab.test
getent hosts master-0.ocp.lab.test

curl -k https://quay.ocp.lab.test:8443/v2/
sudo chronyd -Q 'server 10.0.0.10 iburst'
```

出力例:

```text
eth0 UP aa:bb:cc:dd:ee:ff
eth0 UP 10.0.1.10/24
default via 10.0.1.1 dev eth0
10.0.0.10 quay.ocp.lab.test
10.0.1.10 master-0.ocp.lab.test
true
chronyd exiting
```

MAC と NIC 名を `config/env.sh` に反映する。work 側へ戻って実行する。

```bash
cd ~/ocp-airgap

sed -i 's/^export SNO_NIC=.*/export SNO_NIC="eth0"/' config/env.sh
sed -i 's/^export SNO_MAC=.*/export SNO_MAC="REPLACE_ME_SNO_ETH0_MAC"/' config/env.sh

source config/env.sh
grep '^export SNO_' config/env.sh
```

---

## 7. OpenShift tools を取得する

対応:

- 三井田さんログ: `openshift-install-fips` で ISO を生成。
- 成功手順 md: `oc` / `kubectl` / `oc-mirror` / `openshift-install-fips` を取得。
- 公式 docs: Agent-based Installer 第3章 3.2、第4章 4.2。

```bash
cd ~/ocp-airgap
source config/env.sh

cd pkg

curl -fL -O "https://mirror.openshift.com/pub/openshift-v4/clients/ocp/${OCP_VERSION}/openshift-client-linux-amd64-rhel9.tar.gz"
curl -fL -O "https://mirror.openshift.com/pub/openshift-v4/clients/ocp/${OCP_VERSION}/openshift-install-rhel9-amd64.tar.gz"
curl -fL -O "https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/latest/oc-mirror.rhel9.tar.gz"

mkdir -p /tmp/ocp-client /tmp/ocp-installer /tmp/ocp-mirror

tar xzf openshift-client-linux-amd64-rhel9.tar.gz -C /tmp/ocp-client
tar xzf openshift-install-rhel9-amd64.tar.gz -C /tmp/ocp-installer
tar xzf oc-mirror.rhel9.tar.gz -C /tmp/ocp-mirror

sudo install -m 0755 /tmp/ocp-client/oc /usr/local/bin/oc
sudo install -m 0755 /tmp/ocp-client/kubectl /usr/local/bin/kubectl
sudo install -m 0755 /tmp/ocp-installer/openshift-install-fips /usr/local/bin/openshift-install-fips
sudo install -m 0755 /tmp/ocp-mirror/oc-mirror /usr/local/bin/oc-mirror

hash -r
```

### 7.1 まとめて確認

```bash
oc version --client
openshift-install-fips version
oc-mirror version || oc-mirror --version
which oc kubectl oc-mirror openshift-install-fips
```

出力例:

```text
Client Version: 4.21.21
openshift-install-fips 4.21.21
oc-mirror 4.22.x
/usr/local/bin/oc
/usr/local/bin/openshift-install-fips
```

---

## 8. mirror-registry / Quay を起動する

対応:

- 三井田さんログ: 作業ノード上に mirror-registry / Quay を起動。
- 成功手順 md: `mirror-registry` 展開、`./mirror-registry install`。
- 公式 docs: Disconnected environments の mirror registry for Red Hat OpenShift。

### 8.1 mirror-registry tar を work へ転送する

Mac 側で実行する。`WORK_PUBLIC_IP` は AWS 作成後の値にする。

```bash
export WORK_PUBLIC_IP="REPLACE_ME_WORK_PUBLIC_IP"
export WORK_KEY_PATH="$HOME/.ssh/ocp-airgap/ocp-airgap.pem"

scp -i "${WORK_KEY_PATH}" \
  "$HOME/Downloads/mirror-registry-amd64.tar.gz" \
  ec2-user@"${WORK_PUBLIC_IP}":/home/ec2-user/ocp-airgap/pkg/
```

### 8.2 work 側で展開する

```bash
cd ~/ocp-airgap

tar xzf pkg/mirror-registry-amd64.tar.gz -C mirror-registry
ls -lh mirror-registry
```

### 8.3 Quay を起動する

`Quay init password:` には、この検証用の一時パスワードを入力する。入力したパスワードは後続の `podman login` でも使う。

```bash
cd ~/ocp-airgap
source config/env.sh

cd mirror-registry

sudo firewall-cmd --permanent --add-port=${QUAY_PORT}/tcp
sudo firewall-cmd --reload

sudo cp -a /etc/hosts /etc/hosts.bak.$(date +%Y%m%d-%H%M%S)
sudo sed -i "/${QUAY_HOSTNAME//./\\.}/d" /etc/hosts
printf '%s %s quay\n' "${WORK_PRIVATE_IP}" "${QUAY_HOSTNAME}" | sudo tee -a /etc/hosts

read -rsp 'Quay init password: ' QUAY_PASSWORD
printf '\n'

./mirror-registry install \
  --quayHostname "${QUAY_HOSTNAME}" \
  --quayRoot "$PWD/mirror" \
  --initUser init \
  --initPassword "${QUAY_PASSWORD}"

unset QUAY_PASSWORD
```

### 8.4 まとめて確認

```bash
cd ~/ocp-airgap
source config/env.sh

podman ps
curl -k "https://${QUAY_FQDN}/health/instance"
```

出力例:

```text
quay-redis
quay-app
"status_code":200
```

---

## 9. Quay CA を信頼し、pull secret を merge する

対応:

- 三井田さんログ: Quay CA を install-config に入れる方針。
- 成功手順 md: Quay CA 信頼、podman login、pull secret merge。
- 公式 docs: Agent-based Installer 第2章 2.2.1 ミラーリングされたイメージを使用するように Agent-based Installer を設定する。

### 9.1 Quay CA を work に信頼させる

```bash
cd ~/ocp-airgap
source config/env.sh

sudo cp mirror-registry/mirror/quay-rootCA/rootCA.pem /etc/pki/ca-trust/source/anchors/quay-ocp-lab-test.crt
sudo update-ca-trust extract

sudo mkdir -p "/etc/containers/certs.d/${QUAY_FQDN}"
sudo cp mirror-registry/mirror/quay-rootCA/rootCA.pem "/etc/containers/certs.d/${QUAY_FQDN}/ca.crt"
```

### 9.2 Quay にログインする

`Username` は `init`。Password は Quay 起動時に入力したものを使う。

```bash
cd ~/ocp-airgap
source config/env.sh

curl "https://${QUAY_FQDN}/health/instance"
curl "https://${QUAY_FQDN}/v2/"

podman login "https://${QUAY_FQDN}"
```

### 9.3 Red Hat pull secret を work へ転送する

Mac 側で実行する。

```bash
export WORK_PUBLIC_IP="REPLACE_ME_WORK_PUBLIC_IP"
export WORK_KEY_PATH="$HOME/.ssh/ocp-airgap/ocp-airgap.pem"

scp -i "${WORK_KEY_PATH}" \
  "$HOME/Downloads/pull-secret.txt" \
  ec2-user@"${WORK_PUBLIC_IP}":/home/ec2-user/ocp-airgap/config/pull-secret.txt
```

### 9.4 pull secret に Quay 認証を merge する

```bash
cd ~/ocp-airgap
source config/env.sh

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
```

### 9.5 まとめて確認

```bash
cd ~/ocp-airgap
source config/env.sh

podman login --get-login "${QUAY_FQDN}"
jq -r '.auths | keys[]' config/pull-secret.json
ls -lh config/pull-secret.json config/pull-secret.original.json "${XDG_RUNTIME_DIR}/containers/auth.json"
```

出力例:

```text
init
cloud.openshift.com
quay.io
quay.ocp.lab.test:8443
registry.connect.redhat.com
registry.redhat.io
```

---

## 10. ImageSetConfiguration を作成する

対応:

- 三井田さんログ: Operator を含むミラー対象を指定。
- 成功手順 md: OCP 4.21.21、Operator、additionalImages を `ImageSetConfiguration` に定義。
- 公式 docs: Disconnected environments の oc-mirror plugin v2。

Operator は Agent ISO に焼き込まれない。ここで行うのは、Operator Catalog と Operator 関連イメージを Quay へ事前ミラーする作業である。ISO には Quay CA、pull secret、ミラー参照設定が入る。

```bash
cd ~/ocp-airgap
source config/env.sh

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
  additionalImages:
    - name: registry.redhat.io/rhel8/support-tools:latest
    - name: registry.redhat.io/rhel10/rhel-guest-image:latest
    - name: registry.redhat.io/rhel8/rhel-guest-image:latest
    - name: registry.redhat.io/rhel9/rhel-guest-image:latest
EOF
```

### 10.1 まとめて確認

```bash
cd ~/ocp-airgap

grep -E 'stable-4.21|redhat-operator-index:v4.21|kubernetes-nmstate-operator|odf-operator|kubevirt-hyperconverged|mtv-operator|cluster-logging|gpu-operator-certified|additionalImages' config/imageset-config.yaml
```

出力例:

```text
name: stable-4.21
catalog: registry.redhat.io/redhat/redhat-operator-index:v4.21
name: kubernetes-nmstate-operator
additionalImages:
```

---

## 11. oc-mirror v2 でミラーする

対応:

- 三井田さんログ: oc-mirror v2 で release / Operator / additionalImages を Quay へミラー。
- 成功手順 md: `oc-mirror --v2` 実行、IDMS / ITMS / CatalogSource / ClusterCatalog 生成。
- 公式 docs: Agent-based Installer 第2章、Disconnected environments の oc-mirror plugin v2。

処理は長いので tmux を使う。

```bash
tmux new -s oc-mirror
```

tmux 内で実行する。

```bash
cd ~/ocp-airgap
source config/env.sh

df -h /
curl "https://${QUAY_FQDN}/v2/"
podman login --get-login "${QUAY_FQDN}"

oc-mirror \
  --config config/imageset-config.yaml \
  --workspace file://oc-mirror-work \
  "docker://${QUAY_FQDN}" \
  --v2 \
  --image-timeout 30m
```

### 11.1 まとめて確認

```bash
cd ~/ocp-airgap

ls -lh oc-mirror-work/working-dir/cluster-resources
```

出力例:

```text
idms-oc-mirror.yaml
itms-oc-mirror.yaml
cs-redhat-operator-index-v4-21.yaml
cc-redhat-operator-index-v4-21.yaml
signature-configmap.yaml
```

tmux から抜ける場合は `Ctrl-b` の後に `d`。

---

## 12. install-config.yaml を作成する

対応:

- 三井田さんログ: `additionalTrustBundle`、pull secret、ミラー設定、SNO 相当の `controlPlane replicas: 1 / compute replicas: 0` を設定。
- 成功手順 md: IDMS から install-config 用のミラー参照を作成。
- 公式 docs: Agent-based Installer 第2章 2.2.1、第4章 4.4、第9章 9.1。

### 12.1 SSH 公開鍵と imageDigestSources を用意する

```bash
cd ~/ocp-airgap
source config/env.sh

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

### 12.2 install-config.yaml を生成する

```bash
cd ~/ocp-airgap
source config/env.sh

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

### 12.3 まとめて確認

```bash
cd ~/ocp-airgap

awk '
  /^pullSecret: \|/ {print "pullSecret: |"; print "  MASKED"; inps=1; next}
  inps && /^[^ ]/ {inps=0}
  inps {next}
  {print}
' config/install-config.yaml | sed -n '1,180p'
```

出力例:

```text
baseDomain: lab.test
metadata:
  name: ocp
additionalTrustBundlePolicy: Always
networking:
  networkType: OVNKubernetes
platform:
  none: {}
pullSecret: |
  MASKED
imageDigestSources:
controlPlane:
  replicas: 1
compute:
  - replicas: 0
```

---

## 13. agent-config.yaml を作成する

対応:

- 三井田さんログ: `rendezvousIP`、hostname、role、interface name、MAC、静的 IP、DNS、default route を明示。
- 成功手順 md: 実行順と ISO 生成前の配置を参照。ただし DHCP 最小構成ではなく、三井田さんログ準拠の静的 NMState を採用。
- 公式 docs: Agent-based Installer 第1章 1.5、1.6.2、第9章 9.2。

公式 docs では `v1beta1` の例もあるが、三井田さんログおよび今回成功実績に合わせて `apiVersion: v1alpha1` を使う。

### 13.1 agent-config.yaml を生成する

```bash
cd ~/ocp-airgap
source config/env.sh

if [ -z "${SNO_MAC}" ] || [ "${SNO_MAC}" = "REPLACE_ME_SNO_MAC_AFTER_AWS_CREATION" ]; then
  echo "config/env.sh の SNO_MAC を実際の SNO eth0 MAC に置き換えてから再実行する"
  exit 1
fi

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

### 13.2 まとめて確認

```bash
cd ~/ocp-airgap

cat config/agent-config.yaml
```

出力例:

```text
rendezvousIP: 10.0.1.10
hostname: master-0.ocp.lab.test
name: eth0
macAddress: xx:xx:xx:xx:xx:xx
ip: 10.0.1.10
server:
  - 10.0.0.10
next-hop-address: 10.0.1.1
```

---

## 14. Agent ISO を生成する

対応:

- 三井田さんログ: `install-config.yaml` と `agent-config.yaml` を ISO 生成ディレクトリへコピーし、`openshift-install-fips agent create image` を実行。
- 成功手順 md: Agent ISO 生成。
- 公式 docs: Agent-based Installer 第3章 3.4、第4章 4.6。

```bash
cd ~/ocp-airgap

rm -rf iso-create
mkdir -p iso-create

cp config/install-config.yaml iso-create/install-config.yaml
cp config/agent-config.yaml iso-create/agent-config.yaml

openshift-install-fips agent create image --dir iso-create
```

### 14.1 まとめて確認

```bash
cd ~/ocp-airgap

ls -lh iso-create
find iso-create -maxdepth 2 -type f -printf '%p %s bytes\n' | sort
cat iso-create/rendezvousIP
```

出力例:

```text
agent.x86_64.iso
auth/kubeadmin-password
auth/kubeconfig
10.0.1.10
```

この時点で Agent ISO 生成は完了。

---

## 15. クラスター起動後に使う oc-mirror 生成リソース

対応:

- 三井田さんログ: oc-mirror 出力の CatalogSource / ClusterCatalog を生成。
- 成功手順 md: IDMS / ITMS / CatalogSource / ClusterCatalog の確認。
- 公式 docs: Disconnected environments の oc-mirror plugin v2、Operator Catalog 関連。

Agent ISO 生成時点ではクラスターがまだ存在しないため、以下はクラスター起動後の工程で使う。

```bash
cd ~/ocp-airgap

export KUBECONFIG=iso-create/auth/kubeconfig

oc apply -f oc-mirror-work/working-dir/cluster-resources/idms-oc-mirror.yaml
oc apply -f oc-mirror-work/working-dir/cluster-resources/itms-oc-mirror.yaml
oc apply -f oc-mirror-work/working-dir/cluster-resources/cs-redhat-operator-index-v4-21.yaml
oc apply -f oc-mirror-work/working-dir/cluster-resources/cc-redhat-operator-index-v4-21.yaml
oc apply -f oc-mirror-work/working-dir/cluster-resources/signature-configmap.yaml
```

注意: `idms-oc-mirror.yaml` 相当は、ISO 生成前の `install-config.yaml` に `imageDigestSources` として反映済みである。インストール後に IDMS / ITMS を適用するかは、実際のクラスター状態を確認してから決める。

---

## 16. 最終成果物

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

## 17. 注意点

Operator は Agent ISO に焼き込まれない。  
Operator 関連イメージと Catalog は Quay に事前ミラーされ、ISO には Quay を参照するための CA、pull secret、ミラー参照設定が入る。

`platform: none` では、DNS、API / Ingress の到達性、必要に応じた LB 相当、ノード間通信、Quay 到達性、NTP、firewall、routing を利用者側で用意する。SNO 検証では `api` / `api-int` / `*.apps` を同じ SNO IP へ向けている。

SNO dummy EC2 を作り直した場合、MAC が変わる。`agent-config.yaml` の `macAddress` と `networkConfig.interfaces[].mac-address` を更新し、Agent ISO も作り直す。

---

## 18. 参照元

### 18.1 AWS 接続メモ

- `ocp_airgap_ec2_connection_plan.md`
- 今回は再実行向けに IP を `10.0.0.10` / `10.0.1.10` へ正規化。
- 実ログの `10.0.0.17` / `10.0.1.169` は今回の実行実績としてのみ扱う。

### 18.2 成功手順 md

- `ocp_airgap_agent_operator_included_procedure.md`
- 参照したもの:
  - tools 取得
  - mirror-registry / Quay 起動
  - Quay CA trust
  - pull secret merge
  - ImageSetConfiguration
  - oc-mirror v2
  - IDMS / ITMS / CatalogSource / ClusterCatalog
  - install-config / agent-config
  - Agent ISO 生成
- ただし成功手順 md 内の環境値は本手順へ直接流用しない。

### 18.3 三井田さんログ

- `AgentBased installログ_三井田さん.docx`
- 参照したもの:
  - オンプレ閉域想定
  - 静的 NMState
  - MAC / IP / DNS / default route / rendezvousIP を明示する `agent-config.yaml`
  - `install-config.yaml` と `agent-config.yaml` を ISO 生成ディレクトリへコピーして Agent ISO を生成する流れ

### 18.4 公式ドキュメント

- OpenShift Container Platform 4.21: Installing an on-premise cluster with the Agent-based Installer
  - 第1章 Agent-based Installer を使用したインストールの準備
  - 1.5 ホストの設定
  - 1.6 ネットワークの概要
  - 1.6.2 静的ネットワーキング
  - 1.7 platform `none` の要件
  - 1.7.1 DNS 要件
  - 1.7.2 負荷分散要件
  - 第2章 切断されたインストールのミラーリングについて
  - 2.2.1 ミラーリングされたイメージを使用するように Agent-based Installer を設定する
  - 第3章 クラスターのインストール
  - 3.2 Agent-based Installer のダウンロード
  - 3.3 設定入力の作成
  - 3.4 エージェントイメージの作成と起動
  - 第4章 カスタマイズによるクラスターのインストール
  - 4.4 優先設定入力の作成
  - 4.6 エージェントイメージの作成と起動
  - 第9章 Agent-based Installer のインストール設定パラメーター
  - 9.1 install-config.yaml の設定パラメーター
  - 9.2 agent-config.yaml の設定パラメーター
- OpenShift Container Platform 4.21: Disconnected environments
  - mirror registry for Red Hat OpenShift
  - oc-mirror plugin v2
  - Operator catalog / CatalogSource / ClusterCatalog 関連

### 18.5 今回実行ログ

今回ログは、本手順を組み立てるための実績確認に使った。  
本手順の本文にはログ取得用コマンドは含めていない。

主な実績:

- Quay 起動成功
- Quay CA trust / podman login 成功
- pull secret merge 成功
- ImageSetConfiguration 作成
- oc-mirror v2 成功
- DNS / NTP / firewall 確認
- install-config / agent-config 作成
- Agent ISO 生成成功
