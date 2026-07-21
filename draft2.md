# OpenShift Container Platform 4.21 Agent-based Installer オフラインインストール手順書

本書は、OpenShift Container Platform 4.21をAgent-based Installerで閉域環境へインストールするための手順書です。今回の簡略SNOテストに必要なパラメーター、コマンド、設定例を本書内に記載します。完全閉域でダウンロード、レジストリー、閉域作業の各ホストを分離する場合の資材搬入とミラー手順も併記し、本書の主経路は1台集約構成とします。このMarkdownファイルを本手順の正本とし、別形式の資料を前提にしません。

> [!NOTE]
> 本書のIPアドレス、ドメイン、MACアドレス、デバイス名はテスト用の例です。実行前に、対象環境の値へ変更してください。

> [!IMPORTANT]
> 本書の実行例は、**SNO（control planeとworkerを1台に集約した構成）をデフォルト**とし、検証を簡略化するため、資材取得、DNS、NTP、ミラーレジストリー、ISO生成を1台の作業用ホストへ集約します。資材取得後は外部接続を遮断します。完全閉域の実環境では、後述の構成図どおり、外部の資材取得ホストと閉域側ホストを分離して読み替えます。

## 目次

大項目は本文の同名の大見出しに対応し、「実際の作業手順」配下の小項目は、本文の同じ番号・見出しに対応します。各項目をクリックすると該当箇所へ移動できます。

- **[OpenShiftのインストール方法について](#installation-methods)**
- **[Agent-based Installerを用いたインストールの手順概要](#procedure-overview)**
- **[事前に必要なもの](#prerequisites)**
- **[今回のテスト構成](#test-environment)**
- **[実際の作業手順](#actual-procedure)**
  - **[1. 環境準備](#step-1)**
    - [1.1 作業用ホストにて設定ファイルを準備する](#step-1-1)
    - [1.2 作業用ホストにてパッケージ、DNS、NTP、FWを準備する](#step-1-2)
  - **[2. OpenShift toolsを取得する](#step-2)**
  - **[3. ミラーレジストリーを構築して起動する](#step-3)**
    - [3.1 ミラーレジストリーのtarファイルを準備する](#step-3-1)
  - **[4. ミラーレジストリーへの接続とイメージを準備する](#step-4)**
    - [4.1 Quay CAを信頼させ、pull secretをマージする](#step-4-1)
    - [4.2 ImageSetConfigurationを作成する](#step-4-2)
    - [4.3 oc-mirror v2でミラーする](#step-4-3)
  - **[5. 各YAMLとAgent ISOを作成する](#step-5)**
    - [5.1 SSH公開鍵とimageDigestSourcesを用意する](#step-5-1)
    - [5.2 install-config.yamlを作成する](#step-5-2)
    - [5.3 agent-config.yamlを作成する](#step-5-3)
    - [5.4 Agent ISOを生成する](#step-5-4)
  - **[6. ノードを起動してインストールを監視する](#step-6)**
  - **[7. インストール後の状態を確認する](#step-7)**

<a id="installation-methods"></a>

## OpenShiftのインストール方法について

OpenShiftには、インフラストラクチャーの準備方法や操作方法が異なる複数のインストール方式があります。

- **IPI（Installer-Provisioned Infrastructure）**: 対応する基盤上で、インストーラーがコンピュート、ネットワーク、ロードバランサーなどを作成します。
- **UPI（User-Provisioned Infrastructure）**: 利用者がインフラストラクチャーを準備し、その上へOpenShiftをインストールします。
- **Assisted Installer**: Web画面でホスト検出と設定を行う方式です。専用のBootstrapノードは不要です。
- **Agent-based Installer**: Assisted Installerの仕組みを起動用ISOへ組み込み、ローカルのCLIで設定とインストールを行う方式です。閉域環境で利用できます。

| 方式 | 外部接続可能な環境 | 閉域環境 |
| --- | --- | --- |
| 従来型 | IPI／UPI | IPI／UPIとミラーレジストリー |
| Installer型 | Assisted Installer | Agent-based Installer |

本書では、閉域環境に対応したAgent-based Installerを使用します。インストール方式全体の説明は、[OpenShift Container Platform 4.21 インストールの概要](https://docs.redhat.com/ja/documentation/openshift_container_platform/4.21/html-single/installation_overview/index)を参照してください。

<a id="procedure-overview"></a>

## Agent-based Installerを用いたインストールの手順概要

| 手順 | 今行うことと目的 | 完了時の成果物・状態 | 公式ドキュメントの対応章 |
| --- | --- | --- | --- |
| 1. 環境準備 | ホスト、名前解決、時刻同期、通信許可を準備する | 共通パラメーターとDNS、NTP、FWが使用可能 | 1.5、1.6、1.6.2、1.7.1、3.1 |
| 2. ツール取得 | ISO生成、ミラー、確認に使うCLIを取得する | 必要なCLI一式を実行可能 | 3.2。ミラー用CLIは2.1 |
| 3. レジストリー構築 | 閉域側でイメージを配布する場所を用意する | ミラーレジストリーがHTTPSで応答 | 第2章と2.1を実施するための本書独自準備 |
| 4. イメージ準備 | OCPと利用予定Operatorのイメージを取得・格納する | ミラーイメージとクラスター登録用定義 | 2.1、2.2、2.2.1、およびoc-mirror v2公式手順 |
| 5. YAML・ISO作成 | クラスター設定と各ホスト設定から起動ISOを作る | クラスター設定ファイル、ホスト設定ファイル、Agent ISO | 3.3、3.4、9.1、9.2 |
| 6. インストール | ノードをISO起動し、完了まで監視する | インストール済みクラスター | 3.4、3.5、3.6 |
| 7. 完了確認 | ミラー定義を適用し、ノードとOperatorを確認する | 全ノードReady、Operator正常 | 3.6。詳細確認は本書独自補足 |

「今回のテスト構成」は本書独自の検証構成であり、公式ドキュメントに直接対応する章はありません。

<a id="prerequisites"></a>

## 事前に必要なもの

ここでは、作業開始前に環境が満たしている必要がある条件だけを示します。ダウンロードするファイルは、必要になる手順の中で説明します。

### 作業用ホスト

#### 今回の簡略SNOテスト

RHEL 9互換のx86_64 Linuxホストを1台用意します。今回のテストでは、この1台が外部からの資材取得、DNS、NTP、ファイアウォール、ミラーレジストリー、Agent ISO生成およびインストール操作を兼務します。イメージ取得が終わるまで一時的に外部へ接続し、取得後は外部接続を遮断します。

1台集約時は、**8 vCPU、32 GBメモリー、500 GB以上の空き領域**を本書の開始値とします。500 GBにはダウンロード領域、`oc-mirror`のworkspace、ミラーレジストリー保存領域およびISO生成領域を含みます。取得するOCPバージョン数とOperatorが増える場合は、ミラー実行前の容量確認結果に応じて増やします。

#### 完全閉域で役割を分離する場合

完全閉域の実環境では、外部側のダウンロード用ホストと閉域側ホストの間にネットワーク経路を設けません。次の3つの役割を別ホストへ分離し、承認済み媒体または承認済みの一方向転送経路で資材を搬入します。DNS／NTPは、既存の閉域サービスを使うか、閉域作業用ホストとは別のインフラサービスへさらに分離できます。

| 役割 | 配置 | 主な作業 | 容量を見積もる対象 |
| --- | --- | --- | --- |
| 外部ダウンロード用ホスト | 外部接続ネットワーク | ツール、Pull secret、レジストリー導入資材の取得、`mirror-to-disk` | ダウンロード、workspace、`mirror-to-disk`出力 |
| ミラーレジストリー用ホスト | 完全閉域ネットワーク | OCPリリースイメージとOperatorイメージをHTTPSで配布 | 現在のミラーイメージ、更新時の追加分、保守用の空き |
| 閉域作業／インストール用ホスト | 完全閉域ネットワーク | 搬入物の受け入れ、`disk-to-mirror`、設定ファイル作成、ISO生成、インストール監視 | 搬入物の一時保管とISO生成。搬入物を別領域へ置く場合はその分も加算 |

上表の3台へ一律に500 GBを割り当てるという意味ではありません。外部ダウンロード用ホストとレジストリー用ホストは、選択するOCPバージョンとOperatorに応じて大きく変動します。閉域作業用ホストは、搬入物をどこへ保持するかにより必要量が変わります。いずれもroot権限または`sudo`権限を使用できるようにします。

完全閉域では、次の順序で資材を扱います。

1. 外部ダウンロード用ホストでツール、Pull secret、レジストリー導入資材を取得し、`oc-mirror mirror-to-disk`を実行する。
2. `mirror-to-disk`が生成したディレクトリー構造を崩さず、取得資材一式を承認済み媒体へ格納する。
3. 媒体を閉域作業用ホストへ搬入し、`oc-mirror disk-to-mirror`で閉域のミラーレジストリーへ投入する。
4. SSH鍵、レジストリーCA、`install-config.yaml`および`agent-config.yaml`は閉域側で作成し、Agent ISOを生成する。

今回の簡略テストでは、上記3役割が同じ作業用ホストにあるため、手順2の物理搬送と`disk-to-mirror`を省略し、`mirror-to-mirror`を使用します。

完全閉域で役割を分離する場合は、本文を1台で手順番号順に通すのではなく、次の役割別の順序で実施します。これにより、外部から閉域への搬入を1回にまとめられます。

| 段階 | 実行する場所 | 本文の読み順 |
| --- | --- | --- |
| 1. 閉域基盤の準備 | 閉域作業用／レジストリー用ホスト | 手順1でDNS、NTP、FWなどを準備する。外部資材を必要としない範囲は先行実施できる |
| 2. 外部資材の準備 | 外部ダウンロード用ホスト | 手順2の取得、手順3.1のレジストリーアーカイブ取得、手順4.1の公開用Pull secret作成、手順4.2、手順4.3 Bのmirror-to-diskを実施する。閉域側のインストール操作はまだ行わない |
| 3. 一括搬入 | 承認済み媒体 | ツール、レジストリーアーカイブ、公開用Pull secret、`imageset-config.yaml`、`oc-mirror-work`全体を閉域へ1回で搬入する |
| 4. 閉域側の構築 | 閉域作業用／レジストリー用ホスト | 手順2のツール導入、手順3のレジストリー導入、手順4.1のCA登録・ログイン・Pull secretマージ、手順4.3 Bのdisk-to-mirrorを実施し、手順5以降へ進む |

| 通信元 | 通信先 | 用途 | 必要な時期 |
| --- | --- | --- | --- |
| 外部ダウンロード用ホスト | Red Hat配布サイト／公開レジストリー | ツール、リリース、Operatorイメージの取得 | 資材取得時 |
| 承認済み媒体 | 閉域作業用ホスト | 閉域への資材搬入 | 資材更新時 |
| 閉域作業用ホスト | ミラーレジストリー | `disk-to-mirror`と接続確認 | ミラー投入時 |
| OpenShiftノード | ミラーレジストリー | インストールおよび運用中のイメージ取得 | 常時 |
| 閉域側の各ホスト | DNS／NTP | 名前解決と時刻同期 | 常時 |
| 閉域作業用ホスト | OpenShiftノード／API | ISO配布、インストール監視、完了確認 | インストール時以降 |

搬入する主なものは次のとおりです。ここでは要件ではなく、完全閉域へ持ち込む対象を整理しています。

| 搬入物 | 外部側での準備 | 閉域側での使用先 |
| --- | --- | --- |
| `oc`、`kubectl`、`openshift-install-fips`、`oc-mirror` | 対象OCP 4.21系に対応するアーカイブを取得 | 閉域作業用ホスト |
| mirror registry for Red Hat OpenShiftのアーカイブ | x86_64用を取得 | ミラーレジストリー用ホスト |
| Pull secret | Red Hat Hybrid Cloud Consoleから取得し、安全に保管 | 閉域作業用ホスト |
| `ImageSetConfiguration`と`mirror-to-disk`生成物一式 | 対象バージョンとOperatorを決めて生成 | 閉域作業用ホストからミラーレジストリーへ投入 |
| 必要なOSパッケージまたは閉域用リポジトリー | 閉域側に未導入のパッケージがある場合のみ準備 | 各閉域ホスト |

### OpenShiftノード

まず、Red Hat資料に記載されている基準値を示します。表の値は**1ノードあたり**です。SNOの8 vCPU／16 GB／120 GBは[Single-node OpenShiftの公式資料](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/installing_on_a_single_node/preparing-to-install-sno)では最小要件、Agent-based Installer公式資料では各トポロジーの推奨リソースとして記載されています。追加Operatorを導入すると、この値より多く必要になります。

| トポロジー | control plane数 | compute数 | 公式基準のvCPU | 公式基準のメモリー | 公式基準のストレージ |
| --- | ---: | ---: | ---: | ---: | ---: |
| SNO | 1 | 0 | 8 | 16 GB | 120 GB |
| HAクラスター | 3から5 | 2以上 | 8 | 16 GB | 120 GB |

120 GBはOpenShiftを成立させるための公式基準であり、ログ、更新時のイメージ、監視データ、追加Operatorおよび検証ワークロードを長期間保持する余裕を示す値ではありません。SNOでは、OpenShift本体、監視・ログ、更新用データ、追加機能およびワークロードが1台へ集中します。

本書の検証割当てと、複数ノードを設計するときの開始値を次に示します。

| 区分 | vCPU | メモリー | インストール先ディスク | 位置づけ・使い分け |
| --- | ---: | ---: | ---: | --- |
| **今回の基本SNO検証（デフォルト）** | **8** | **32 GB** | **300 GB SSD** | OCP本体、標準監視、軽いテストワークロードを確認する構成 |
| 拡張SNO検証 | 16以上 | 64 GB以上 | 500 GB SSDから開始 | 複数Operator、ログ収集、Virtualizationなどを追加して検証する構成 |
| 複数ノードのcontrol plane設計目安 | 8以上／台 | 32 GB以上／台 | 300 GB SSD以上／台 | 本書では未検証。3台用意し、通常のアプリケーションワークロードは配置しない |
| 複数ノードのworker設計目安 | 8以上／台 | 32 GB以上／台 | 300 GB SSD以上／台 | 本書では未検証。2台以上用意し、ワークロードと追加Operator分を加算する |

> [!IMPORTANT]
> **この手順のSNOをそのまま試す場合は、8 vCPU／32 GB／300 GB SSDを使用します。** 500 GBは必須値ではなく、追加Operatorやログなどを同じノードへ載せる拡張検証の開始値です。PV、VMディスク、ログの長期保管領域、バックアップ格納先は300 GB／500 GBへ含めず、別途設計します。

OpenShift Virtualizationでは、OCP基盤分に加えて仮想化基盤のオーバーヘッドと各VMのCPU、メモリーおよびディスクが必要です。したがって「500 GBあればVirtualizationにも十分」とは判断せず、VMディスクはStorageClass／PVとして別に算定します。

各ノードについて、次の値を事前に確認します。

- ホスト名
- 固定IPアドレスとprefix length
- デフォルトゲートウェイ
- 使用するNIC名とMACアドレス
- インストール先ディスクのデバイス名

### ネットワーク、DNS、NTP、ファイアウォール

- Machine Network、Cluster Network、Service Networkが互いに重複しない。
- 全OpenShiftノードからDNS、NTPおよびミラーレジストリーへ到達できる。
- 全OpenShiftノード間でクラスター通信ができる。
- ノードの正引きと逆引きが一致する。
- SNOではAPI、内部API、アプリケーション用のDNS名をSNOの固定IPへ向ける。
- 複数ノードではAPIと内部APIをAPI VIPへ、アプリケーション用ワイルドカードをIngress VIPへ向ける。
- 複数ノードで使用するAPI VIPとIngress VIPは、Machine Network内の未使用IPとする。
- 全ノードが同じNTPソースへ同期できる。
- 作業用ホストでDNS、NTP、SSHおよびミラーレジストリーの待受ポートを許可できる。

| DNS名 | SNO（`platform: none`） | 複数ノード（`platform: baremetal`） |
| --- | --- | --- |
| `api.<cluster>.<baseDomain>` | SNOのIP | API VIP |
| `api-int.<cluster>.<baseDomain>` | SNOのIP | API VIP |
| `*.apps.<cluster>.<baseDomain>` | SNOのIP | Ingress VIP |
| 各ノードのFQDN | 各ノードのIP | 各ノードのIP |
| ミラーレジストリーのFQDN | ミラーレジストリーのIP | ミラーレジストリーのIP |

SNOでは独立したVIPとロードバランサーは使用しません。複数ノードでは`platform: baremetal`のクラスター管理VIPを使用し、外部ロードバランサーは使用しません。

<a id="test-environment"></a>

## 今回のテスト構成

> [!IMPORTANT]
> 完全閉域では、外部の資材取得ホストと閉域側の作業用ホストを分離し、資材を承認済み媒体などで搬入します。ただし、**今回のテストでは簡略化のため1台の作業用ホストへすべての役割を集約します**。以降の主手順は、特記がない限りこの1台で実行します。イメージ取得後に外部接続を遮断し、OpenShiftノードは外部接続を使用しません。

### 完全閉域環境で分離する場合の構成

<!-- 構成図1 差し込み位置 -->
![完全閉域環境の構成](images/airgap-separated.svg)

図1 完全閉域環境でホストを分離する場合の構成

### 今回実施する簡略化構成

<!-- 構成図2 差し込み位置 -->
![今回実施する簡略SNO構成](images/sno-simplified.svg)

図2 今回実施するSNOの簡略化構成

図中の大きな色付き枠はネットワーク、枠内の白い箱はそれぞれ別の物理ホストまたは仮想ホストです。線は接続関係または資材搬送経路であり、作業順序ではありません。今回のテスト対象は図2のSNOです。複数ノードの設定例も後続手順に残しますが、図2のテスト環境へ同時に構築するものではありません。

作業用ホストは、テストを簡略化するため次の役割を兼務します。実環境では既存サービスや別ホストへ分離できます。

- 資材取得
- ミラーレジストリー
- DNS
- NTP
- ファイアウォール
- Agent ISO生成とインストール監視

| 項目 | SNO | 複数ノード |
| --- | --- | --- |
| `platform` | `none` | `baremetal` |
| control plane | 1台 | 3台 |
| worker replica | 0 | 2 |
| API VIP／Ingress VIP | 使用しない | 使用する |
| `rendezvousIP` | SNOのIP | control plane 1台の実IP |
| 専用Bootstrapノード | 不要 | 不要 |

`rendezvousIP`は、インストール中に初期構築処理を実行するノードのIPです。API VIPを指定しません。

<a id="actual-procedure"></a>

## 実際の作業手順

<a id="step-1"></a>

### 1. 環境準備

> **公式ドキュメント対応:** 1.5「ホストの設定」、1.6「ネットワークの概要」、1.6.2「静的ネットワーキング」、1.7.1「プラットフォーム none のDNS要件」、3.1「前提条件」に対応します。NTPの具体値は9.2.2、ファイアウォールの具体コマンドは本書の検証用補足です。

<a id="step-1-1"></a>

#### 1.1 作業用ホストにて設定ファイルを準備する

> **公式ドキュメント対応:** 3.3「設定入力の作成」、9.1「インストール設定パラメーター」、9.2「エージェント設定パラメーター」で使用する値を、本書用に先に整理します。

OpenShiftのミラー、設定YAMLおよびインストールで繰り返し使用する値を`env.sh`へまとめます。**SNO用と複数ノード用のどちらか一方だけ**を選び、コード枠全体をコピーして入力値を変更します。2つの内容を結合しません。

DNSサーバー構築用の逆引きゾーンとforwarder、NTPサーバーの上位時刻源、firewalldのゾーンは`env.sh`へ含めません。これらはホストごとのインフラ設定であるため、手順1.2の各設定箇所で入力します。

最初に、以降で使用する作業ディレクトリーを作成します。

```bash
mkdir -p ~/ocp-airgap/pkg ~/ocp-airgap/config
cd ~/ocp-airgap
```

#### A. SNO用env.sh（デフォルト）

`RENDEZVOUS_IP`は条件分岐で自動設定せず、`SNO_IP`と同じ値を手入力します。作業用ホスト、DNS、NTPおよびレジストリーのIPは、手順1.2の各ネットワーク設定で入力します。

**作成ファイル：`~/ocp-airgap/env.sh`（SNO用）**

```bash
# ファイル名: ~/ocp-airgap/env.sh
# このファイル内の値を対象環境に合わせて変更する

# OpenShift共通
export OCP_VERSION="4.21.21"
export OCP_ARCH="amd64"
export OC_MIRROR_VERSION="4.21.21"
export OC_MIRROR_URL="https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.21.21/oc-mirror.rhel9.tar.gz"

# クラスター共通
export CLUSTER_NAME="ocp"
export BASE_DOMAIN="lab.example.com"
export MACHINE_NETWORK_CIDR="192.168.100.0/24"
export CLUSTER_NETWORK_CIDR="10.128.0.0/14"
export CLUSTER_HOST_PREFIX="23"
export SERVICE_NETWORK_CIDR="172.30.0.0/16"
export NODE_PREFIX_LENGTH="24"
export DEFAULT_GATEWAY="192.168.100.1"

# ミラーレジストリー
export REGISTRY_HOSTNAME="registry.ocp.lab.example.com"
export REGISTRY_PORT="8443"               # mirror registryの既定値
export REGISTRY_FQDN="${REGISTRY_HOSTNAME}:${REGISTRY_PORT}"
export REGISTRY_AUTH_FILE="$HOME/ocp-airgap/config/containers-auth.json"

# SNO（デフォルト）
export SNO_HOSTNAME="sno-0.ocp.lab.example.com"
export SNO_IP="192.168.100.20"
export SNO_NIC="ens192"
export SNO_MAC="02:00:00:00:00:20"
export SNO_INSTALL_DISK="/dev/sda"
export RENDEZVOUS_IP="192.168.100.20"     # SNO_IPと同じ値
```

#### B. 複数ノード用env.sh（`platform: baremetal`）

複数ノードでは、次の枠全体を`env.sh`として保存します。SNO用変数は含めません。`RENDEZVOUS_IP`は、最初のcontrol planeノードである`MASTER0_IP`と同じ値を手入力します。

**作成ファイル：`~/ocp-airgap/env.sh`（複数ノード用）**

```bash
# ファイル名: ~/ocp-airgap/env.sh
# このファイル内の値を対象環境に合わせて変更する

# OpenShift共通
export OCP_VERSION="4.21.21"
export OCP_ARCH="amd64"
export OC_MIRROR_VERSION="4.21.21"
export OC_MIRROR_URL="https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.21.21/oc-mirror.rhel9.tar.gz"

# クラスター共通
export CLUSTER_NAME="ocp"
export BASE_DOMAIN="lab.example.com"
export MACHINE_NETWORK_CIDR="192.168.100.0/24"
export CLUSTER_NETWORK_CIDR="10.128.0.0/14"
export CLUSTER_HOST_PREFIX="23"
export SERVICE_NETWORK_CIDR="172.30.0.0/16"
export NODE_PREFIX_LENGTH="24"
export DEFAULT_GATEWAY="192.168.100.1"

# ミラーレジストリー
export REGISTRY_HOSTNAME="registry.ocp.lab.example.com"
export REGISTRY_PORT="8443"
export REGISTRY_FQDN="${REGISTRY_HOSTNAME}:${REGISTRY_PORT}"
export REGISTRY_AUTH_FILE="$HOME/ocp-airgap/config/containers-auth.json"

# 複数ノード
export API_VIP="192.168.100.15"
export INGRESS_VIP="192.168.100.16"
export MASTER0_HOSTNAME="master-0.ocp.lab.example.com"
export MASTER0_IP="192.168.100.21"
export MASTER0_NIC="ens192"
export MASTER0_MAC="02:00:00:00:00:21"
export MASTER0_INSTALL_DISK="/dev/sda"
export MASTER1_HOSTNAME="master-1.ocp.lab.example.com"
export MASTER1_IP="192.168.100.22"
export MASTER1_NIC="ens192"
export MASTER1_MAC="02:00:00:00:00:22"
export MASTER1_INSTALL_DISK="/dev/sda"
export MASTER2_HOSTNAME="master-2.ocp.lab.example.com"
export MASTER2_IP="192.168.100.23"
export MASTER2_NIC="ens192"
export MASTER2_MAC="02:00:00:00:00:23"
export MASTER2_INSTALL_DISK="/dev/sda"
export WORKER0_HOSTNAME="worker-0.ocp.lab.example.com"
export WORKER0_IP="192.168.100.31"
export WORKER0_NIC="ens192"
export WORKER0_MAC="02:00:00:00:00:31"
export WORKER0_INSTALL_DISK="/dev/sda"
export WORKER1_HOSTNAME="worker-1.ocp.lab.example.com"
export WORKER1_IP="192.168.100.32"
export WORKER1_NIC="ens192"
export WORKER1_MAC="02:00:00:00:00:32"
export WORKER1_INSTALL_DISK="/dev/sda"
export RENDEZVOUS_IP="192.168.100.21"     # MASTER0_IPと同じ値
```

`env.sh`は後続のシェルコマンドで使用する値です。YAMLコードブロック内の`${...}`は自動展開されないため、保存時に実値へ置き換えます。OCPバージョンを変更する場合は、手順4.2の`minVersion`、`maxVersion`およびチャネルも同時に変更します。

完全閉域で3役割を分離する場合は、同じ設計値の`env.sh`を外部ダウンロード用ホスト、ミラーレジストリー用ホスト、閉域作業用ホストの`~/ocp-airgap/`へ配置します。各ホストは自分の担当手順で必要な変数だけを使用します。Pull secretやレジストリーのパスワードは`env.sh`へ記載しません。

**実行:** 作成した設定ファイルを読み込みます。

```bash
cd ~/ocp-airgap
source env.sh
```

`export`の値は現在のシェルだけに有効です。再ログイン、再起動、または別のターミナルで手順を再開するたびに、`source ~/ocp-airgap/env.sh`を実行します。

選択した構成に対応する確認だけを実行します。

**SNOの確認:** `SNO_IP`と`RENDEZVOUS_IP`が同じ値であることを確認します。

```bash
printf 'SNO_IP=%s\nRENDEZVOUS_IP=%s\n' \
  "$SNO_IP" "$RENDEZVOUS_IP"
```

**出力例:**

```text
SNO_IP=192.168.100.20
RENDEZVOUS_IP=192.168.100.20
```

**複数ノードの確認:** `MASTER0_IP`と`RENDEZVOUS_IP`が同じ値であり、API VIPとIngress VIPが空でないことを確認します。

```bash
printf 'MASTER0_IP=%s\nRENDEZVOUS_IP=%s\nAPI_VIP=%s\nINGRESS_VIP=%s\n' \
  "$MASTER0_IP" "$RENDEZVOUS_IP" "$API_VIP" "$INGRESS_VIP"
```

**出力例:**

```text
MASTER0_IP=192.168.100.21
RENDEZVOUS_IP=192.168.100.21
API_VIP=192.168.100.15
INGRESS_VIP=192.168.100.16
```

<a id="step-1-2"></a>

#### 1.2 作業用ホストにてパッケージ、DNS、NTP、FWを準備する

> **公式ドキュメント対応:** 1.5、1.6、1.6.2、1.7.1に対応します。BIND、chrony、firewalldの具体コマンドは、本書の検証環境用補足です。

以下は、今回のテストで作業用ホスト自身にDNS、NTP、ファイアウォールを設定する例です。既存のDNS、NTP、ファイアウォールを利用する場合は、同じ名前解決、時刻同期、通信許可を満たすように各環境の設定へ読み替えます。後で省略する可能性がある作業も、検証できるよう本版ではコマンドを残します。

以降は、`~/ocp-airgap`を作業ディレクトリーとし、`pkg`、`config`、`mirror-registry`をその配下に配置します。

作業に必要なパッケージをインストールし、firewalldとchronydを起動します。

```bash
sudo dnf install -y \
  openssl jq curl tar gzip \
  nmstate podman skopeo \
  firewalld tmux \
  bind bind-utils \
  chrony

sudo systemctl enable --now firewalld
sudo systemctl enable --now chronyd

rpm -q openssl jq curl tar gzip nmstate podman skopeo \
  firewalld tmux bind bind-utils chrony
systemctl is-active firewalld chronyd
```

**出力例:**

```text
openssl-<version>
jq-<version>
curl-<version>
...
chrony-<version>
active
active
```

完全閉域でホストを分離する場合は、上の一式を閉域作業用ホストへ導入し、外部ダウンロード用ホストとミラーレジストリー用ホストには次の担当分を導入します。閉域側でOSリポジトリーを参照できない場合は、事前に搬入したRPMまたは組織内の閉域リポジトリーを使用します。

```bash
# 外部ダウンロード用ホスト
sudo dnf install -y jq curl tar gzip tmux
rpm -q jq curl tar gzip tmux
```

**出力例:**

```text
jq-<version>
curl-<version>
tar-<version>
gzip-<version>
tmux-<version>
```

```bash
# ミラーレジストリー用ホスト
sudo dnf install -y curl tar gzip podman firewalld chrony
sudo systemctl enable --now firewalld
rpm -q curl tar gzip podman firewalld chrony
systemctl is-active firewalld
```

**出力例:**

```text
curl-<version>
tar-<version>
gzip-<version>
podman-<version>
firewalld-<version>
chrony-<version>
active
```

##### ホスト名とhostsファイルを設定する

作業用ホストのホスト名を設定し、BIND起動前でも作業用ホストとミラーレジストリーを解決できるようにします。

次の`sed`は`/etc/hosts`全体を置き換えるものではありません。初回は削除対象がないため何も削除せず、再実行時は前回追加した`# ocp-airgap-start`から`# ocp-airgap-end`までだけを削除します。その後、現在の入力値で管理範囲を作り直します。既存のその他の行は残ります。

構成に対応する一方のコード枠だけを実行します。

**SNO用 — 変更対象ファイル：`/etc/hosts`**

```bash
# 変更対象ファイル: /etc/hosts
# /etc/hosts設定用の入力値
export WORK_IP="192.168.100.10"
export WORK_HOSTNAME="work.ocp.lab.example.com"
export REGISTRY_IP="192.168.100.10"  # 分離構成ではレジストリーホストのIP

sudo hostnamectl set-hostname "$WORK_HOSTNAME"
sudo cp -a /etc/hosts "/etc/hosts.bak.$(date +%Y%m%d-%H%M%S)"
sudo sed -i '/^# ocp-airgap-start$/,/^# ocp-airgap-end$/d' /etc/hosts
sudo sh -c 'cat >> /etc/hosts' <<EOF
# ocp-airgap-start
${WORK_IP} ${WORK_HOSTNAME} work
${REGISTRY_IP} ${REGISTRY_HOSTNAME} registry
${SNO_IP} ${SNO_HOSTNAME} sno-0 api.${CLUSTER_NAME}.${BASE_DOMAIN} api api-int.${CLUSTER_NAME}.${BASE_DOMAIN} api-int
# ocp-airgap-end
EOF

hostname -f
getent hosts "$REGISTRY_HOSTNAME"
getent hosts "api.${CLUSTER_NAME}.${BASE_DOMAIN}"
```

**出力例:**

```text
work.ocp.lab.example.com
192.168.100.10  registry.ocp.lab.example.com registry
192.168.100.20  sno-0.ocp.lab.example.com sno-0 api.ocp.lab.example.com api api-int.ocp.lab.example.com api-int
```

**複数ノード用 — 変更対象ファイル：`/etc/hosts`**

```bash
# 変更対象ファイル: /etc/hosts
# /etc/hosts設定用の入力値
export WORK_IP="192.168.100.10"
export WORK_HOSTNAME="work.ocp.lab.example.com"
export REGISTRY_IP="192.168.100.10"  # 分離構成ではレジストリーホストのIP

sudo hostnamectl set-hostname "$WORK_HOSTNAME"
sudo cp -a /etc/hosts "/etc/hosts.bak.$(date +%Y%m%d-%H%M%S)"
sudo sed -i '/^# ocp-airgap-start$/,/^# ocp-airgap-end$/d' /etc/hosts
sudo sh -c 'cat >> /etc/hosts' <<EOF
# ocp-airgap-start
${WORK_IP} ${WORK_HOSTNAME} work
${REGISTRY_IP} ${REGISTRY_HOSTNAME} registry
${API_VIP} api.${CLUSTER_NAME}.${BASE_DOMAIN} api api-int.${CLUSTER_NAME}.${BASE_DOMAIN} api-int
${INGRESS_VIP} console-openshift-console.apps.${CLUSTER_NAME}.${BASE_DOMAIN}
${MASTER0_IP} ${MASTER0_HOSTNAME}
${MASTER1_IP} ${MASTER1_HOSTNAME}
${MASTER2_IP} ${MASTER2_HOSTNAME}
${WORKER0_IP} ${WORKER0_HOSTNAME}
${WORKER1_IP} ${WORKER1_HOSTNAME}
# ocp-airgap-end
EOF

hostname -f
getent hosts "$REGISTRY_HOSTNAME"
getent hosts "api.${CLUSTER_NAME}.${BASE_DOMAIN}"
```

**出力例:**

```text
work.ocp.lab.example.com
192.168.100.10  registry.ocp.lab.example.com registry
192.168.100.15  api.ocp.lab.example.com api api-int.ocp.lab.example.com api-int
```

##### DNSを設定する

このBIND例では、作業用ホスト、ミラーレジストリー、各OpenShiftノードのFQDNを`${CLUSTER_NAME}.${BASE_DOMAIN}`ゾーン内に置き、FQDNの先頭ラベルからAレコード名を作成します。

最初に、このDNS設定だけで使用する値を入力します。`REVERSE_ZONE`はMachine Networkの逆引きゾーン名です。本書の例は`192.168.100.0/24`のため、`100.168.192.in-addr.arpa`を指定します。上位DNSを利用する場合は`UPSTREAM_DNS`へforwarderのIPを設定し、完全閉域で使用しない場合は空文字にします。

```bash
# DNS設定用の入力値。このシェル内だけで使用する
export DNS_LISTEN_IP="192.168.100.10"
export DNS_HOSTNAME="work.ocp.lab.example.com"
export DNS_SERVER_IP="192.168.100.10"
export DNS_ALLOW_CIDR="192.168.100.0/24"
export REGISTRY_IP="192.168.100.10"  # 分離構成ではレジストリーホストのIP
export REVERSE_ZONE="100.168.192.in-addr.arpa"
export UPSTREAM_DNS="192.168.100.1"  # 使用しない場合は空文字
```

**作成ファイル：`/etc/named.conf`**

```bash
# 作成ファイル: /etc/named.conf
FORWARDERS_BLOCK=""
if [ -n "$UPSTREAM_DNS" ]; then
  FORWARDERS_BLOCK="forwarders { ${UPSTREAM_DNS}; };"
fi

sudo cp -a /etc/named.conf "/etc/named.conf.bak.$(date +%Y%m%d-%H%M%S)"

sudo sh -c 'cat > /etc/named.conf' <<EOF
options {
  listen-on port 53 { 127.0.0.1; ${DNS_LISTEN_IP}; };
  listen-on-v6 port 53 { none; };
  directory "/var/named";
  dump-file "/var/named/data/cache_dump.db";
  statistics-file "/var/named/data/named_stats.txt";
  memstatistics-file "/var/named/data/named_mem_stats.txt";
  allow-query { localhost; ${DNS_ALLOW_CIDR}; };
  recursion yes;
  allow-recursion { localhost; ${DNS_ALLOW_CIDR}; };
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

zone "${REVERSE_ZONE}" IN {
  type master;
  file "${REVERSE_ZONE}.zone";
  allow-update { none; };
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
EOF
```

正引きゾーンを作成します。SNO用と複数ノード用はそれぞれファイル全体を作成する独立したコード枠です。構成に対応する一方だけを実行します。

**SNO用 — 作成ファイル：`/var/named/${CLUSTER_NAME}.${BASE_DOMAIN}.zone`**

```bash
# 作成ファイル: /var/named/${CLUSTER_NAME}.${BASE_DOMAIN}.zone
FORWARD_ZONE_FILE="/var/named/${CLUSTER_NAME}.${BASE_DOMAIN}.zone"
SERIAL="$(date +%Y%m%d)01"
WORK_RECORD="${DNS_HOSTNAME%%.*}"
REGISTRY_RECORD="${REGISTRY_HOSTNAME%%.*}"
SNO_RECORD="${SNO_HOSTNAME%%.*}"

sudo sh -c "cat > \"$FORWARD_ZONE_FILE\"" <<EOF
\$TTL 300
@ IN SOA ${DNS_HOSTNAME}. root.${CLUSTER_NAME}.${BASE_DOMAIN}. (
  ${SERIAL}
  1H
  15M
  1W
  300 )
  IN NS ${DNS_HOSTNAME}.

${WORK_RECORD}     IN A ${DNS_LISTEN_IP}
${REGISTRY_RECORD} IN A ${REGISTRY_IP}
${SNO_RECORD} IN A ${SNO_IP}
api      IN A ${SNO_IP}
api-int  IN A ${SNO_IP}
*.apps   IN A ${SNO_IP}
EOF
```

**複数ノード用 — 作成ファイル：`/var/named/${CLUSTER_NAME}.${BASE_DOMAIN}.zone`**

```bash
# 作成ファイル: /var/named/${CLUSTER_NAME}.${BASE_DOMAIN}.zone
FORWARD_ZONE_FILE="/var/named/${CLUSTER_NAME}.${BASE_DOMAIN}.zone"
SERIAL="$(date +%Y%m%d)01"
WORK_RECORD="${DNS_HOSTNAME%%.*}"
REGISTRY_RECORD="${REGISTRY_HOSTNAME%%.*}"
MASTER0_RECORD="${MASTER0_HOSTNAME%%.*}"
MASTER1_RECORD="${MASTER1_HOSTNAME%%.*}"
MASTER2_RECORD="${MASTER2_HOSTNAME%%.*}"
WORKER0_RECORD="${WORKER0_HOSTNAME%%.*}"
WORKER1_RECORD="${WORKER1_HOSTNAME%%.*}"

sudo sh -c "cat > \"$FORWARD_ZONE_FILE\"" <<EOF
\$TTL 300
@ IN SOA ${DNS_HOSTNAME}. root.${CLUSTER_NAME}.${BASE_DOMAIN}. (
  ${SERIAL}
  1H
  15M
  1W
  300 )
  IN NS ${DNS_HOSTNAME}.

${WORK_RECORD}     IN A ${DNS_LISTEN_IP}
${REGISTRY_RECORD} IN A ${REGISTRY_IP}
api      IN A ${API_VIP}
api-int  IN A ${API_VIP}
*.apps   IN A ${INGRESS_VIP}
${MASTER0_RECORD} IN A ${MASTER0_IP}
${MASTER1_RECORD} IN A ${MASTER1_IP}
${MASTER2_RECORD} IN A ${MASTER2_IP}
${WORKER0_RECORD} IN A ${WORKER0_IP}
${WORKER1_RECORD} IN A ${WORKER1_IP}
EOF
```

逆引きゾーンを作成します。SNO用と複数ノード用のうち、構成に対応する一方だけを実行します。このコピー用のBIND例は、上で入力したIPv4 `/24`ネットワーク専用です。別の`/24`を使用する場合は`REVERSE_ZONE`と各IPを変更します。`/23`や`/16`などを使用する場合は、対象ネットワークに合わせて逆引きゾーン名とPTRのowner生成方法も変更してください。

**SNO用 — 作成ファイル：`/var/named/${REVERSE_ZONE}.zone`**

```bash
# 作成ファイル: /var/named/${REVERSE_ZONE}.zone
REVERSE_ZONE_FILE="/var/named/${REVERSE_ZONE}.zone"
SERIAL="$(date +%Y%m%d)01"

sudo sh -c "cat > \"$REVERSE_ZONE_FILE\"" <<EOF
\$TTL 300
@ IN SOA ${DNS_HOSTNAME}. root.${CLUSTER_NAME}.${BASE_DOMAIN}. (
  ${SERIAL}
  1H
  15M
  1W
  300 )
  IN NS ${DNS_HOSTNAME}.

${DNS_LISTEN_IP##*.} IN PTR ${DNS_HOSTNAME}.
${REGISTRY_IP##*.} IN PTR ${REGISTRY_HOSTNAME}.
${SNO_IP##*.} IN PTR ${SNO_HOSTNAME}.
${SNO_IP##*.} IN PTR api.${CLUSTER_NAME}.${BASE_DOMAIN}.
${SNO_IP##*.} IN PTR api-int.${CLUSTER_NAME}.${BASE_DOMAIN}.
EOF
```

**複数ノード用 — 作成ファイル：`/var/named/${REVERSE_ZONE}.zone`**

```bash
# 作成ファイル: /var/named/${REVERSE_ZONE}.zone
REVERSE_ZONE_FILE="/var/named/${REVERSE_ZONE}.zone"
SERIAL="$(date +%Y%m%d)01"

sudo sh -c "cat > \"$REVERSE_ZONE_FILE\"" <<EOF
\$TTL 300
@ IN SOA ${DNS_HOSTNAME}. root.${CLUSTER_NAME}.${BASE_DOMAIN}. (
  ${SERIAL}
  1H
  15M
  1W
  300 )
  IN NS ${DNS_HOSTNAME}.

${DNS_LISTEN_IP##*.} IN PTR ${DNS_HOSTNAME}.
${REGISTRY_IP##*.} IN PTR ${REGISTRY_HOSTNAME}.
${API_VIP##*.} IN PTR api.${CLUSTER_NAME}.${BASE_DOMAIN}.
${API_VIP##*.} IN PTR api-int.${CLUSTER_NAME}.${BASE_DOMAIN}.
${MASTER0_IP##*.} IN PTR ${MASTER0_HOSTNAME}.
${MASTER1_IP##*.} IN PTR ${MASTER1_HOSTNAME}.
${MASTER2_IP##*.} IN PTR ${MASTER2_HOSTNAME}.
${WORKER0_IP##*.} IN PTR ${WORKER0_HOSTNAME}.
${WORKER1_IP##*.} IN PTR ${WORKER1_HOSTNAME}.
EOF
```

ゾーンファイルの所有者、権限、SELinuxコンテキストを整え、設定を検査してBINDを起動します。

```bash
FORWARD_ZONE_FILE="/var/named/${CLUSTER_NAME}.${BASE_DOMAIN}.zone"
REVERSE_ZONE_FILE="/var/named/${REVERSE_ZONE}.zone"

sudo chown root:named "$FORWARD_ZONE_FILE" "$REVERSE_ZONE_FILE"
sudo chmod 640 "$FORWARD_ZONE_FILE" "$REVERSE_ZONE_FILE"
sudo restorecon -v "$FORWARD_ZONE_FILE" "$REVERSE_ZONE_FILE"

sudo named-checkconf /etc/named.conf \
  && echo "OK: named.conf"
sudo named-checkzone \
  "${CLUSTER_NAME}.${BASE_DOMAIN}" "$FORWARD_ZONE_FILE"
sudo named-checkzone "$REVERSE_ZONE" "$REVERSE_ZONE_FILE"

sudo systemctl enable --now named
sudo systemctl restart named
```

**出力例:**

```text
OK: named.conf
zone ocp.lab.example.com/IN: loaded serial YYYYMMDD01
OK
zone 100.168.192.in-addr.arpa/IN: loaded serial YYYYMMDD01
OK
```

作業用ホスト自身もこのDNSを使用するように設定します。

```bash
DEV="$(ip route show default | awk '{print $5; exit}')"
CON="$(nmcli -t -f NAME,DEVICE connection show --active \
  | awk -F: -v dev="$DEV" '$2==dev{print $1; exit}')"

printf 'DEV=%s\nCON=%s\n' "$DEV" "$CON"
test -n "$CON" && echo "OK: active connection found"
sudo nmcli connection modify \
  "$CON" ipv4.dns "$DNS_SERVER_IP" ipv4.ignore-auto-dns yes
sudo nmcli connection up "$CON"
```

**出力例:**

```text
DEV=ens192
CON=<接続プロファイル名>
OK: active connection found
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/<番号>)
```

接続を再有効化すると、SSHセッションが一時的に切断される場合があります。

完全閉域でホストを分離する場合は、ミラーレジストリー用ホストでも`env.sh`を読み込み、同ホストのリゾルバーを閉域DNSへ向けます。

```bash
cd ~/ocp-airgap
source env.sh
export DNS_SERVER_IP="192.168.100.10"  # 閉域DNSサーバーのIP

DEV="$(ip route show default | awk '{print $5; exit}')"
CON="$(nmcli -t -f NAME,DEVICE connection show --active \
  | awk -F: -v dev="$DEV" '$2==dev{print $1; exit}')"

printf 'DEV=%s\nCON=%s\n' "$DEV" "$CON"
test -n "$CON" && echo "OK: active connection found"
sudo nmcli connection modify \
  "$CON" ipv4.dns "$DNS_SERVER_IP" ipv4.ignore-auto-dns yes
sudo nmcli connection up "$CON"
getent hosts "$REGISTRY_HOSTNAME"
```

**出力例:**

```text
DEV=ens192
CON=<接続プロファイル名>
OK: active connection found
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/<番号>)
192.168.100.10  registry.ocp.lab.example.com
```

OpenShiftノードのDNSは、手順5.3の`agent-config.yaml`へ同じDNSサーバーIPを入力します。

正引きと逆引きを確認します。ワイルドカードは、実際のルートを想定した名前で確認します。

**SNO用の確認:** 

```bash
dig "$REGISTRY_HOSTNAME" +short
dig "api.${CLUSTER_NAME}.${BASE_DOMAIN}" +short
dig "api-int.${CLUSTER_NAME}.${BASE_DOMAIN}" +short
dig "console-openshift-console.apps.${CLUSTER_NAME}.${BASE_DOMAIN}" +short
dig -x "$DNS_LISTEN_IP" +short
dig -x "$REGISTRY_IP" +short
dig -x "$SNO_IP" +short
```

**出力例:**

```text
192.168.100.10
192.168.100.20
192.168.100.20
192.168.100.20
work.ocp.lab.example.com.
registry.ocp.lab.example.com.
work.ocp.lab.example.com.
registry.ocp.lab.example.com.
sno-0.ocp.lab.example.com.
api.ocp.lab.example.com.
api-int.ocp.lab.example.com.
```

**複数ノード用の確認:** 

```bash
dig "$REGISTRY_HOSTNAME" +short
dig "api.${CLUSTER_NAME}.${BASE_DOMAIN}" +short
dig "api-int.${CLUSTER_NAME}.${BASE_DOMAIN}" +short
dig "console-openshift-console.apps.${CLUSTER_NAME}.${BASE_DOMAIN}" +short
dig "$MASTER0_HOSTNAME" +short
dig "$MASTER1_HOSTNAME" +short
dig "$MASTER2_HOSTNAME" +short
dig "$WORKER0_HOSTNAME" +short
dig "$WORKER1_HOSTNAME" +short
dig -x "$API_VIP" +short
dig -x "$MASTER0_IP" +short
dig -x "$MASTER1_IP" +short
dig -x "$MASTER2_IP" +short
dig -x "$WORKER0_IP" +short
dig -x "$WORKER1_IP" +short
```

**出力例:**

```text
192.168.100.10
192.168.100.15
192.168.100.15
192.168.100.16
192.168.100.21
192.168.100.22
192.168.100.23
192.168.100.31
192.168.100.32
api.ocp.lab.example.com.
api-int.ocp.lab.example.com.
master-0.ocp.lab.example.com.
master-1.ocp.lab.example.com.
master-2.ocp.lab.example.com.
worker-0.ocp.lab.example.com.
worker-1.ocp.lab.example.com.
```

##### NTPを設定する

作業用ホストをOpenShiftノードのNTPサーバーとして設定します。外部NTPを使用できる間はそこへ同期し、外部接続を遮断した後はローカルクロックを配信します。既存NTPを使用する場合は、このブロックを読み替えます。

このNTP設定だけで使用する上位時刻源を入力します。完全閉域で上位NTPを使用しない場合は空文字にします。

```bash
# NTP設定用の入力値。このシェル内だけで使用する
export NTP_SERVER_IP="192.168.100.10"
export NTP_ALLOW_CIDR="192.168.100.0/24"
export UPSTREAM_NTP="192.168.100.1"  # 使用しない場合は空文字
```

**追記先ファイル：`/etc/chrony.conf`**

```bash
# 追記先ファイル: /etc/chrony.conf
sudo cp -a /etc/chrony.conf "/etc/chrony.conf.bak.$(date +%Y%m%d-%H%M%S)"
sudo sed -i \
  '/^# ocp-airgap-ntp-start$/,/^# ocp-airgap-ntp-end$/d' \
  /etc/chrony.conf
sudo sed -Ei \
  's/^(pool|server)[[:space:]]+/# disabled-by-ocp-airgap: &/' \
  /etc/chrony.conf

{
  echo '# ocp-airgap-ntp-start'
  if [ -n "$UPSTREAM_NTP" ]; then
    echo "server ${UPSTREAM_NTP} prefer iburst"
  fi
  echo "allow ${NTP_ALLOW_CIDR}"
  echo 'local stratum 10'
  echo '# ocp-airgap-ntp-end'
} | sudo tee -a /etc/chrony.conf >/dev/null

sudo systemctl enable --now chronyd
sudo systemctl restart chronyd

chronyc sources -v
chronyc tracking
```

**出力例:**

```text
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^* <上位NTPサーバー>             2   6   377    24   <時刻差>

Reference ID    : <参照先>
Stratum         : <階層>
Leap status     : Normal

# 外部遮断後にローカルクロックを使用する場合
Reference ID    : 7F7F0101
Stratum         : 10
Leap status     : Normal
```

完全閉域でホストを分離する場合は、ミラーレジストリー用ホストを閉域作業用ホストのNTPへ同期させます。

**変更対象ファイル：`/etc/chrony.conf`（ミラーレジストリー用ホスト）**

```bash
# 変更対象ファイル: /etc/chrony.conf
export NTP_SERVER_IP="192.168.100.10"  # 閉域NTPサーバーのIP

sudo cp -a /etc/chrony.conf "/etc/chrony.conf.bak.$(date +%Y%m%d-%H%M%S)"
sudo sed -i \
  '/^# ocp-airgap-client-ntp-start$/,/^# ocp-airgap-client-ntp-end$/d' \
  /etc/chrony.conf
sudo sed -Ei \
  's/^(pool|server)[[:space:]]+/# disabled-by-ocp-airgap: &/' \
  /etc/chrony.conf
sudo sh -c 'cat >> /etc/chrony.conf' <<EOF
# ocp-airgap-client-ntp-start
server ${NTP_SERVER_IP} prefer iburst
# ocp-airgap-client-ntp-end
EOF

sudo systemctl enable --now chronyd
sudo systemctl restart chronyd
chronyc sources -v
chronyc tracking
```

**出力例:**

```text
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^* work.ocp.lab.example.com     10   6   377    18   <時刻差>

Reference ID    : <閉域NTPサーバー>
Leap status     : Normal
```

OpenShiftノードのNTPは、手順5.3の`additionalNTPSources`へ同じIPを入力します。

##### ファイアウォールを設定する

構成に対応する一方を実行します。`FW_ZONE`は`env.sh`の値ではありません。各ホストで`firewall-cmd --get-active-zones`を確認し、そのホストのactive zoneをコード枠の先頭へ入力します。

**A. 今回の1台集約テスト:** 作業用ホストが提供するDNS、NTP、SSH、ミラーレジストリーのポートを許可します。

```bash
export FW_ZONE="public"  # このホストのactive zoneへ変更

sudo systemctl enable --now firewalld
sudo firewall-cmd --permanent --zone="$FW_ZONE" --add-service=dns
sudo firewall-cmd --permanent --zone="$FW_ZONE" --add-service=ntp
sudo firewall-cmd --permanent --zone="$FW_ZONE" --add-service=ssh
sudo firewall-cmd --permanent --zone="$FW_ZONE" \
  --add-port="${REGISTRY_PORT}/tcp"
sudo firewall-cmd --reload
```

サービスと許可状態を確認します。

```bash
systemctl is-active named
systemctl is-active chronyd
systemctl is-active firewalld
chronyc tracking

sudo firewall-cmd --get-active-zones
sudo firewall-cmd --zone="$FW_ZONE" --query-service=dns
sudo firewall-cmd --zone="$FW_ZONE" --query-service=ntp
sudo firewall-cmd --zone="$FW_ZONE" --query-service=ssh
sudo firewall-cmd --zone="$FW_ZONE" \
  --query-port="${REGISTRY_PORT}/tcp"
sudo firewall-cmd --zone="$FW_ZONE" --list-all
```

**出力例:**

```text
success
success
success
success
success
active
active
active
Reference ID    : <参照先>
Leap status     : Normal
public
  interfaces: ens192
yes
yes
yes
yes
public (active)
  interfaces: ens192
  services: dns ntp ssh
  ports: 8443/tcp
```

OpenShiftノード間とAPI/VIPに必要な通信は、作業用ホストのfirewalldへ混在させず、スイッチ、ルーターまたは各ノード側のネットワーク設計で許可します。

**B. 完全閉域で3役割を分離する場合:** 閉域作業用ホストではDNS、NTP、SSHを、ミラーレジストリー用ホストではSSHとレジストリーポートを許可します。外部ダウンロード用ホストには閉域側への通信を許可しません。

閉域作業用ホストで実行します。

```bash
export FW_ZONE="public"  # 閉域作業用ホストのactive zoneへ変更

sudo systemctl enable --now firewalld
sudo firewall-cmd --permanent --zone="$FW_ZONE" --add-service=dns
sudo firewall-cmd --permanent --zone="$FW_ZONE" --add-service=ntp
sudo firewall-cmd --permanent --zone="$FW_ZONE" --add-service=ssh
sudo firewall-cmd --reload

sudo firewall-cmd --zone="$FW_ZONE" --query-service=dns
sudo firewall-cmd --zone="$FW_ZONE" --query-service=ntp
sudo firewall-cmd --zone="$FW_ZONE" --query-service=ssh
```

**出力例:**

```text
success
success
success
success
yes
yes
yes
```

ミラーレジストリー用ホストで、同じ`env.sh`を読み込んだ後に実行します。

```bash
cd ~/ocp-airgap
source env.sh
export FW_ZONE="public"  # レジストリーホストのactive zoneへ変更

sudo systemctl enable --now firewalld
sudo firewall-cmd --permanent --zone="$FW_ZONE" --add-service=ssh
sudo firewall-cmd --permanent --zone="$FW_ZONE" \
  --add-port="${REGISTRY_PORT}/tcp"
sudo firewall-cmd --reload

sudo firewall-cmd --zone="$FW_ZONE" --query-service=ssh
sudo firewall-cmd --zone="$FW_ZONE" \
  --query-port="${REGISTRY_PORT}/tcp"
```

**出力例:**

```text
success
success
success
yes
yes
```

レジストリーの8443/TCPは閉域ネットワークからだけ到達可能にし、外部ダウンロード用ホストとのルーティングは作成しません。

この時点では、OpenShiftのCLIはまだインストールしていないため、`oc version`は実行しません。

<a id="step-2"></a>

### 2. OpenShift toolsを取得する

> **公式ドキュメント対応:** 3.2「Agent-based Installerのダウンロード」。`oc-mirror`の取得は2.1「Agent-based Installerによる非接続インストールのイメージのミラーリング」を参照します。

この手順では、次のツールを取得します。ここで初めてツール名と用途を定義します。

| ツール | 用途 |
| --- | --- |
| `oc`／`kubectl` | OpenShiftおよびKubernetesの操作 |
| `openshift-install-fips` | Agent ISOの生成とインストール監視 |
| `oc-mirror` | 公開レジストリーからのイメージ取得と閉域ミラーへの格納 |

本書で取得するRHEL 9用インストーラーアーカイブには、`openshift-install-fips`という名前のバイナリーが含まれます。後続手順でもこのコマンド名を統一して使用します。クラスター自体のFIPSモードを有効にするかどうかは別の設計項目です。

今回のテストで兼務している作業用ホストから実行します。この時点では外部接続を維持します。

```bash
cd ~/ocp-airgap/pkg

curl -fL -O "https://mirror.openshift.com/pub/openshift-v4/clients/ocp/${OCP_VERSION}/openshift-client-linux-amd64-rhel9.tar.gz"
curl -fL -O "https://mirror.openshift.com/pub/openshift-v4/clients/ocp/${OCP_VERSION}/openshift-install-rhel9-amd64.tar.gz"
curl -fL -o oc-mirror.rhel9.tar.gz "${OC_MIRROR_URL}"
```

完全閉域でホストを分離する場合は、外部ダウンロード用ホストで取得したアーカイブ一式を、手順4.3 Bの`mirror-to-disk`生成物と一緒に一括搬入します。この時点ではまだ媒体を閉域へ搬入しません。外部ダウンロード用ホストでは`oc-mirror`を使用できるようにし、一括搬入後は閉域作業用ホストで4コマンドすべてを導入します。今回のテストでは同じ作業用ホストが両方を兼ねるため、次の解凍とインストールを1回だけ実施します。

ダウンロードが完了したら解凍します。

```bash
mkdir -p \
  "extract/${OCP_VERSION}/client" \
  "extract/${OCP_VERSION}/installer" \
  "extract/${OC_MIRROR_VERSION}/oc-mirror"

tar xzf openshift-client-linux-amd64-rhel9.tar.gz -C "extract/${OCP_VERSION}/client"
tar xzf openshift-install-rhel9-amd64.tar.gz -C "extract/${OCP_VERSION}/installer"
tar xzf oc-mirror.rhel9.tar.gz -C "extract/${OC_MIRROR_VERSION}/oc-mirror"
```

解凍されたツールをインストールします。

```bash
sudo install -m 0755 "extract/${OCP_VERSION}/client/oc" /usr/local/bin/oc
sudo install -m 0755 "extract/${OCP_VERSION}/client/kubectl" /usr/local/bin/kubectl
sudo install -m 0755 "extract/${OCP_VERSION}/installer/openshift-install-fips" /usr/local/bin/openshift-install-fips
sudo install -m 0755 "extract/${OC_MIRROR_VERSION}/oc-mirror/oc-mirror" /usr/local/bin/oc-mirror
```

インストールが完了した後に、初めてバージョンを確認します。

```bash
oc version --client
openshift-install-fips version
oc-mirror version || oc-mirror --version
which oc kubectl oc-mirror openshift-install-fips
```

**出力例:**

```text
Client Version: 4.21.21
Kustomize Version: v5.x.x
openshift-install 4.21.21
...
oc-mirror version ...
/usr/local/bin/oc
/usr/local/bin/kubectl
/usr/local/bin/oc-mirror
/usr/local/bin/openshift-install-fips
```

`oc`とインストーラーが指定した4.21.zであることを確認します。`oc version`には`--client`を付けているため、この時点ではクラスターのバージョンは表示されません。

<a id="step-3"></a>

### 3. ミラーレジストリーを構築して起動する

> **公式ドキュメント対応:** 第2章および2.1を実施するための環境準備です。mirror registry for Red Hat OpenShiftの構築コマンド自体は、Agent-based Installer公式資料の直接対応手順ではありません。

本書では、テスト環境用のミラーレジストリーとしてmirror registry for Red Hat OpenShiftを使用します。組織内に既存のOCI互換レジストリーがある場合は、そのレジストリーのFQDN、ポート、CAおよび認証情報へ読み替えます。

<a id="step-3-1"></a>

#### 3.1 ミラーレジストリーのtarファイルを準備する

> **公式ドキュメント対応:** 第2章および2.1のミラーリングを実行するための本書独自の事前準備です。mirror registry for Red Hat OpenShiftの導入自体は、Agent-based Installer公式資料の直接対応手順ではありません。

外部接続可能な端末で[Red Hat Hybrid Cloud ConsoleのDownloadsページ](https://console.redhat.com/openshift/downloads)を開き、mirror registry for Red Hat OpenShiftのx86_64用アーカイブをダウンロードします。本書では、ダウンロードしたファイル名を`mirror-registry-amd64.tar.gz`とします。

今回の簡略テストでは、`mirror-registry-amd64.tar.gz`を作業用ホストの`~/ocp-airgap/pkg/`へ配置します。完全閉域でホストを分離する場合は、外部ダウンロード用ホストで取得して保持し、手順4.3 Bの資材と一緒に一括搬入してから、ミラーレジストリー用ホストの`~/ocp-airgap/pkg/`へ配置します。

次のコマンドで解凍します。完全閉域でホストを分離する場合、このコマンド以降は一括搬入後にミラーレジストリー用ホストで実行します。

```bash
cd ~/ocp-airgap
mkdir -p mirror-registry
tar xzf pkg/mirror-registry-amd64.tar.gz -C mirror-registry
ls -lh mirror-registry
```

**出力例:**

```text
total ...
-rwxr-xr-x. 1 user user  ... mirror-registry
-rw-r--r--. 1 user user  ... execution-environment.tar
-rw-r--r--. 1 user user  ... image-archive.tar
```

`$REGISTRY_HOSTNAME`が手順1.2のDNS設定で指定したレジストリーIPへ名前解決できることを確認してから、レジストリーを起動します。今回の簡略テストでは作業用ホストで実行します。完全閉域で分離する場合は、同じ資材をミラーレジストリー用ホストへ配置し、そのホストで実行します。次の`passwd123`は本書のテスト値です。実行前に対象環境の初期パスワードへ変更します。

```bash
getent hosts "$REGISTRY_HOSTNAME"

cd ~/ocp-airgap/mirror-registry
./mirror-registry install \
  --quayHostname "$REGISTRY_HOSTNAME" \
  --quayRoot "$HOME/ocp-airgap/mirror-registry/mirror" \
  --initUser init \
  --initPassword passwd123 \
  && echo "OK: mirror registry install completed"

mkdir -p "$HOME/ocp-airgap/config"
cp mirror/quay-rootCA/rootCA.pem \
  "$HOME/ocp-airgap/config/quay-rootCA.pem"
test -s "$HOME/ocp-airgap/config/quay-rootCA.pem" \
  && echo "OK: quay root CA"
```

**出力例:**

```text
192.168.100.10  registry.ocp.lab.example.com
...
PLAY RECAP *********************************************************************
localhost                  : ok=... changed=... unreachable=0 failed=0 ...
OK: mirror registry install completed
OK: quay root CA
```

導入処理の表示はバージョンにより異なります。`unreachable=0`、`failed=0`であることを確認します。

完全閉域でホストを分離する場合は、作成した`quay-rootCA.pem`を閉域内の承認済み転送方法で閉域作業用ホストの`~/ocp-airgap/config/`へコピーします。以降、閉域作業用ホストではこの固定パスを使用します。

起動状態を確認します。この段階の`-k`は、次の手順でCAを信頼させる前の一時確認に限って使用します。

```bash
podman ps
curl -k "https://${REGISTRY_FQDN}/health/instance"
curl -kI "https://${REGISTRY_FQDN}/v2/"
```

**出力例:**

```text
CONTAINER ID  IMAGE                                  ...  STATUS       NAMES
...           quay.io/projectquay/quay:...           ...  Up ...       quay-app
...           quay.io/sclorg/postgresql-...          ...  Up ...       quay-postgres
...           quay.io/redis/redis-...                ...  Up ...       quay-redis
{"data":{"services":{...}},"status_code":200}
HTTP/2 401
```

`/v2/`は認証前のため`401 Unauthorized`でも正常です。接続拒否、タイムアウトまたは5xx応答がある場合は、先へ進まず原因を解消します。

<a id="step-4"></a>

### 4. ミラーレジストリーへの接続とイメージを準備する

> **公式ドキュメント対応:** 2.1、2.2「切断されたレジストリーのOpenShift Container Platformイメージリポジトリーのミラーリング」、2.2.1「ミラーリングされたイメージを使用するようにAgent-based Installerを設定する」に対応します。

<a id="step-4-1"></a>

#### 4.1 Quay CAを信頼させ、pull secretをマージする

> **公式ドキュメント対応:** 2.2.1「ミラーリングされたイメージを使用するようにAgent-based Installerを設定する」の、pull secretと追加信頼バンドの準備に対応します。

今回の簡略テストでは、本節を上から順に実行します。完全閉域でホストを分離する場合は、外部ダウンロード用ホストで先に本節後半の`pull-secret-redhat.json`作成までを実施し、手順4.2と4.3 Bのmirror-to-diskへ進みます。一括搬入と手順3のレジストリー導入後に本節の先頭へ戻り、閉域作業用ホストでCA登録、レジストリーログインおよびPull secretマージを実施します。

前の手順で起動したQuayのCA証明書を、作業用ホストのOSとコンテナー関連ツールから信頼できるようにします。

```bash
cd ~/ocp-airgap

sudo cp config/quay-rootCA.pem \
  /etc/pki/ca-trust/source/anchors/quay-ocp.crt
sudo update-ca-trust extract

sudo mkdir -p "/etc/containers/certs.d/${REGISTRY_FQDN}"
sudo cp config/quay-rootCA.pem \
  "/etc/containers/certs.d/${REGISTRY_FQDN}/ca.crt"
```

TLS検証を有効にした状態で確認します。

```bash
curl "https://${REGISTRY_FQDN}/health/instance"
```

**出力例:**

```text
{"data":{"services":{...}},"status_code":200}
```

証明書検証エラーが表示される場合は、ログインへ進まずCAの配置先と`REGISTRY_FQDN`を見直します。

レジストリーへログインします。認証情報は、再ログイン後も使えるように`env.sh`で指定したファイルへ保存します。

```bash
podman login \
  --authfile "$REGISTRY_AUTH_FILE" \
  "$REGISTRY_FQDN" \
  -u init
```

**出力例:**

```text
Password:
Login Succeeded!
```

続いて、外部接続可能な端末で[Red Hat Hybrid Cloud ConsoleのPull secretページ](https://console.redhat.com/openshift/install/pull-secret)を開き、Pull secretをダウンロードします。本書では、ダウンロードしたファイルを外部ダウンロード用ホストの`~/ocp-airgap/config/pull-secret.txt`へ配置します。

外部ダウンロード用ホストで、公開レジストリー用の認証ファイルを作成します。今回の簡略テストでは、同じ作業用ホストで実行します。

```bash
cd ~/ocp-airgap
mkdir -p config
jq . config/pull-secret.txt > config/pull-secret-redhat.json
chmod 600 config/pull-secret-redhat.json
```

完全閉域でホストを分離する場合は、この`pull-secret-redhat.json`を手順4.3の`mirror-to-disk`で使用し、その生成物と一緒に閉域作業用ホストの`~/ocp-airgap/config/`へ一括搬入します。今回の簡略テストでは転送せず、そのまま次へ進みます。

以降は閉域作業用ホストで実行します。公開レジストリー用Pull secretをコピーし、先ほどのPodman認証ファイルからQuayの認証情報を追加します。

```bash
cp -a config/pull-secret-redhat.json config/pull-secret.json

AUTHFILE="${REGISTRY_AUTH_FILE}"
REGISTRY="${REGISTRY_FQDN}"

test -f "${AUTHFILE}" \
  && echo "OK: registry auth file"
jq -e --arg reg "${REGISTRY}" '.auths[$reg]' "${AUTHFILE}" >/dev/null \
  && echo "OK: registry credentials"

jq --arg reg "${REGISTRY}" \
  --argjson entry "$(jq --arg reg "${REGISTRY}" '.auths[$reg]' "${AUTHFILE}")" \
  '.auths[$reg] = $entry' \
  config/pull-secret.json > config/pull-secret.merged.json

mv config/pull-secret.merged.json config/pull-secret.json
cp config/pull-secret.json "${AUTHFILE}"
jq -c . config/pull-secret.json > config/pull-secret-compact.json
chmod 600 \
  config/pull-secret-redhat.json \
  config/pull-secret.json \
  config/pull-secret-compact.json \
  "${AUTHFILE}"

test -s config/pull-secret.json \
  && test -s config/pull-secret-compact.json \
  && echo "OK: merged pull secret files"
```

**出力例:**

```text
OK: registry auth file
OK: registry credentials
OK: merged pull secret files
```

認証情報が含まれることを確認します。

```bash
podman login \
  --authfile "$REGISTRY_AUTH_FILE" \
  --get-login "${REGISTRY_FQDN}"
jq -r '.auths | keys[]' config/pull-secret.json
```

**出力例:**

```text
init
cloud.openshift.com
quay.io
registry.ocp.lab.example.com:8443
registry.redhat.io
```

キーの一覧には、ミラーレジストリーと元のPull secretにあった公開レジストリーの両方が残ります。秘密情報そのものは表示されません。

別のターミナルまたは再ログイン後に再開する場合は、`source ~/ocp-airgap/env.sh`を実行します。`$REGISTRY_AUTH_FILE`が存在しない場合は、このログインとpull secretマージの手順を再実行してから`oc-mirror`へ進みます。

<a id="step-4-2"></a>

#### 4.2 ImageSetConfigurationを作成する

> **公式ドキュメント対応:** 2.1「Agent-based Installerによる非接続インストールのイメージのミラーリング」から案内される、oc-mirrorプラグインv2のImageSetConfiguration作成手順に対応します。

ImageSetConfigurationは、次のミラー処理で取得するOpenShiftリリース、アーキテクチャーおよびOperatorを定義するYAMLです。ここでファイル名を`imageset-config.yaml`とします。

今回の簡略テストでは作業用ホストで作成します。完全閉域でホストを分離する場合は、外部ダウンロード用ホストで同じファイルを作成し、`mirror-to-disk`生成物と一緒に閉域作業用ホストへ搬入します。生成コマンドは使用せず、次の内容をエディターで保存します。バージョンとOperatorは対象環境に合わせて変更します。

Operatorを選ぶ目安は次のとおりです。

| 利用する機能 | 事前ミラーするpackage | 選択時の注意 |
| --- | --- | --- |
| ログを外部へ転送する | `cluster-logging` | 初期運用から使う場合に含める |
| ログをLokiStackへ保存・検索する | `cluster-logging`、`loki-operator` | 保存期間と永続ストレージを事前に設計する。SNOでは容量圧迫に注意する |
| アプリケーション／PVをバックアップ・リストアする | `redhat-oadp-operator` | S3互換などのバックアップ先を用意する場合に含める |
| VMを実行する | `kubevirt-hyperconverged` | CPU、メモリー、VM用PVを別途見積もる |
| NMStateでノードNIC、Linux bridge、bondなどを設定する | `kubernetes-nmstate-operator` | NMStateによるホストネットワーク設定を使用する場合に含める |
| 既存仮想基盤からVMを移行する | `mtv-operator` | VM移行を行う場合に含める |
| ODFを利用する | `odf-operator` | 対応トポロジー、必要ノード数、専用ディスクを別途設計した場合のみ含める |
| ノードのローカルディスクをPVとして使用する | `local-storage-operator` | 対象ディスクを決めている場合に含める |
| NVIDIA GPUを使用する | `nfd`、`gpu-operator-certified` | GPU搭載ノードで利用する場合に含める |
| 閉域内で更新グラフを提供する | `cincinnati-operator` | 初回導入だけなら不要。利用時は`graph: true`へ変更する |
| ハブクラスターとして複数クラスターを管理する | `multicluster-engine` | ハブ用途の場合に含める |

> [!IMPORTANT]
> 閉域では後からOperatorを追加すると、イメージを再取得して再搬入する必要があります。インストール直後から次回の資材搬入までに使用する予定のOperatorは、あらかじめミラー対象へ含めます。ただし、**ミラーすることとOperatorをインストールすることは別作業**です。使わないOperatorまで全て含めると、取得容量、転送時間およびミラーレジストリー容量が増えます。

次の例は、用途表に挙げたOperatorをミラー対象として有効にした記載例です。**実際に使用しないpackageは、ミラー実行前に該当する`- name`からそのpackageの末尾までを削除してください。** OCP本体だけをミラーする場合は、`operators:`ブロック全体を削除します。行ごとのコメント解除は不要です。チャネル名は、対象の4.21.zで利用できるものを確認してから確定します。`graph: false`は本書の既定値です。`cincinnati-operator`を使って閉域更新サービスを構成する場合は、`graph: true`に変更します。

**作成ファイル：`~/ocp-airgap/config/imageset-config.yaml`**

```yaml
# ファイル名: ~/ocp-airgap/config/imageset-config.yaml
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
        - name: loki-operator
        - name: local-storage-operator
        - name: cincinnati-operator
          defaultChannel: v1
          channels:
            - name: v1
        - name: multicluster-engine
        - name: nfd
          defaultChannel: stable
          channels:
            - name: stable
    - catalog: registry.redhat.io/redhat/certified-operator-index:v4.21
      packages:
        - name: gpu-operator-certified
          defaultChannel: stable
          channels:
            - name: stable
```

<a id="step-4-3"></a>

#### 4.3 oc-mirror v2でミラーする

> **公式ドキュメント対応:** 2.1から案内されるoc-mirrorプラグインv2の手順に対応します。今回の簡略テストはmirror-to-mirror、完全閉域での役割分離はmirror-to-diskとdisk-to-mirrorを使用します。

次のAまたはBのうち、構成に対応する一方だけを実行します。長時間実行されるため、必要に応じて`tmux`を使用します。必要量は選択するOCPバージョン数とOperatorで変わるため、一律の容量は設定しません。

##### A. 今回の簡略テスト：mirror-to-mirror

1台の作業用ホストから公開レジストリーのイメージを取得し、同じホスト上のQuayへ直接ミラーします。ミラー対象、workspaceおよびミラーレジストリー保存分を含めた空き容量を確認します。

```bash
df -h "$HOME/ocp-airgap"
```

**出力例:**

```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/mapper/...  1.0T  120G  880G  13% /home
```

`Avail`が、選択したOCPリリースとOperatorの見積容量に余裕を加えた値以上であることを確認します。

必要な空き容量を確保できたら、ミラーを実行します。`oc-mirror`は`env.sh`で設定した`REGISTRY_AUTH_FILE`を認証情報として使用するため、実行前にファイルが存在することも確認します。

```bash
cd ~/ocp-airgap
test -s "$REGISTRY_AUTH_FILE" && echo "OK: registry auth file"

oc-mirror \
  --config config/imageset-config.yaml \
  --workspace file://oc-mirror-work \
  "docker://${REGISTRY_FQDN}" \
  --v2 \
  && echo "OK: mirror-to-mirror completed"
```

**出力例:**

```text
OK: registry auth file
...
OK: mirror-to-mirror completed
```

最後に`OK: mirror-to-mirror completed`が表示されない場合は、直前のエラーを解消して再実行します。

実行完了後、今回の作業用ホストの外部接続を遮断します。以降は、作業用ホスト、ミラーレジストリー、DNS、NTP、OpenShiftノードだけで処理を続けます。

##### B. 完全閉域で3役割を分離する場合：mirror-to-disk／disk-to-mirror

**外部ダウンロード用ホストで実行:** 公開レジストリー用の`pull-secret-redhat.json`を使い、ディスクへ取得します。この時点で閉域側のミラーレジストリーへは接続しません。

```bash
cd ~/ocp-airgap
export MIRROR_DATA_DIR="$HOME/ocp-airgap/oc-mirror-work"

df -h "$HOME/ocp-airgap"
test -s config/pull-secret-redhat.json \
  && echo "OK: Red Hat pull secret"
mkdir -p "$MIRROR_DATA_DIR"

oc-mirror \
  --config config/imageset-config.yaml \
  --authfile config/pull-secret-redhat.json \
  "file://${MIRROR_DATA_DIR}" \
  --v2 \
  && echo "OK: mirror-to-disk completed"

find "$MIRROR_DATA_DIR" -maxdepth 3 -type f -print | head -50
```

**出力例:**

```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/mapper/...  1.0T  120G  880G  13% /home
OK: Red Hat pull secret
...
OK: mirror-to-disk completed
/home/<user>/ocp-airgap/oc-mirror-work/working-dir/...
```

`OK: mirror-to-disk completed`と、最後の`find`による取得ファイル一覧が表示されます。

`oc-mirror-work`ディレクトリー全体、`config/imageset-config.yaml`、`config/pull-secret-redhat.json`および手順2・3で取得した閉域側用資材を承認済み媒体へ格納します。`oc-mirror-work`の一部だけを選ばず、生成されたディレクトリー構造を維持したまま、閉域作業用ホストの`~/ocp-airgap/`へ搬入します。本書ではチェックサム確認を必須手順にしません。

**閉域作業用ホストで実行:** 手順3のミラーレジストリーが起動し、手順4.1のマージ済み認証ファイルがあることを確認してから、搬入したデータをレジストリーへ投入します。

```bash
cd ~/ocp-airgap
export MIRROR_DATA_DIR="$HOME/ocp-airgap/oc-mirror-work"

df -h "$HOME/ocp-airgap"
test -d "$MIRROR_DATA_DIR" && echo "OK: mirror data directory"
test -s "$REGISTRY_AUTH_FILE" && echo "OK: registry auth file"

oc-mirror \
  --config config/imageset-config.yaml \
  --authfile "$REGISTRY_AUTH_FILE" \
  --from "file://${MIRROR_DATA_DIR}" \
  "docker://${REGISTRY_FQDN}" \
  --v2 \
  && echo "OK: disk-to-mirror completed"
```

**出力例:**

```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/mapper/...  1.0T  300G  700G  30% /home
OK: mirror data directory
OK: registry auth file
...
OK: disk-to-mirror completed
```

最後に`OK: disk-to-mirror completed`が表示されない場合は、直前のエラーを解消して再実行します。

完了すると、後続のクラスター設定で使用するミラー定義やOperatorカタログ定義が`cluster-resources`ディレクトリーへ生成されます。

```bash
ls -lh oc-mirror-work/working-dir/cluster-resources
```

**出力例:**

```text
total ...
-rw-r--r--. 1 user user ... idms-oc-mirror.yaml
-rw-r--r--. 1 user user ... release-signatures.yaml
-rw-r--r--. 1 user user ... cs-redhat-operator-index-v4-21.yaml
```

`idms-oc-mirror.yaml`はOCPリリースのミラー時に生成されます。カタログ定義や`itms-oc-mirror.yaml`は、選択したミラー対象に応じて生成されます。

<a id="step-5"></a>

### 5. 各YAMLとAgent ISOを作成する

> **公式ドキュメント対応:** 3.3「設定入力の作成」、3.4「エージェントイメージの作成と起動」、9.1「使用可能なインストール設定パラメーター」、9.2「使用可能なエージェント設定パラメーター」に対応します。

ここから、クラスター全体の設定を記述する`install-config.yaml`と、インストール対象ホストの設定を記述する`agent-config.yaml`を作成します。両方のYAMLはAgent ISOを生成するディレクトリへ配置します。

<a id="step-5-1"></a>

#### 5.1 SSH公開鍵とimageDigestSourcesを用意する

> **公式ドキュメント対応:** 2.2.1「ミラーリングされたイメージを使用するようにAgent-based Installerを設定する」および3.3「設定入力の作成」に対応します。

> [!NOTE]
> 添付のOCP 4.21公式PDFの2.2.1および9.1では互換キー`imageContentSources`が記載されています。[OpenShift Installer 4.21の型定義](https://github.com/openshift/installer/blob/release-4.21/pkg/types/installconfig.go)では、このキーは引き続き使用できますが非推奨であり、`imageDigestSources`の使用が推奨されています。本書は4.21.21向けの新規手順として`imageDigestSources`を使用します。

障害調査時にRHCOSノードへ接続できるように、SSH公開鍵を用意します。既存の管理用公開鍵がある場合は、そのファイルを使用できます。

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
ssh-keygen -t ed25519 -N '' -C "ocp-node-debug" -f ~/.ssh/ocp-node-debug
chmod 600 ~/.ssh/ocp-node-debug
chmod 644 ~/.ssh/ocp-node-debug.pub
```

次に、`oc-mirror`が生成したImageDigestMirrorSet（IDMS）定義の`idms-oc-mirror.yaml`を確認します。

```bash
cd ~/ocp-airgap
cat oc-mirror-work/working-dir/cluster-resources/idms-oc-mirror.yaml
```

**出力例:**

```text
apiVersion: config.openshift.io/v1
kind: ImageDigestMirrorSet
metadata:
  name: idms-oc-mirror
spec:
  imageDigestMirrors:
    - source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
      mirrors:
        - registry.ocp.lab.example.com:8443/openshift/release
    - source: quay.io/openshift-release-dev/ocp-release
      mirrors:
        - registry.ocp.lab.example.com:8443/openshift/release-images
```

`mirrors`が対象環境の`${REGISTRY_FQDN}`配下を指していない場合は、`install-config.yaml`を作る前に手順4.3を見直します。

表示内容と、`install-config.yaml`へ貼り付ける内容は形式が異なります。次の2つを混同しないでください。

| 項目 | 意味 |
| --- | --- |
| `source` | OpenShiftが通常参照する外部の取得元。例では`quay.io` |
| `mirrors` | 今回の閉域環境で参照させるミラーレジストリー。例では`registry.ocp.lab.example.com:8443` |

**A. `oc-mirror`生成値（確認のみ・このYAML全体はコピーしない）**

**確認するファイル：`~/ocp-airgap/oc-mirror-work/working-dir/cluster-resources/idms-oc-mirror.yaml`**

```yaml
# ファイル名: ~/ocp-airgap/oc-mirror-work/working-dir/cluster-resources/idms-oc-mirror.yaml
apiVersion: config.openshift.io/v1
kind: ImageDigestMirrorSet
metadata:
  name: idms-oc-mirror
spec:
  imageDigestMirrors:
    - source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
      mirrors:
        - registry.ocp.lab.example.com:8443/openshift/release
    - source: quay.io/openshift-release-dev/ocp-release
      mirrors:
        - registry.ocp.lab.example.com:8443/openshift/release-images
```

**B. `install-config.yaml`へ貼り付ける値（コピー対象）**

`spec.imageDigestMirrors`配下のリストを転記し、トップキーを`imageDigestSources`にします。`apiVersion`、`kind`、`metadata`および`spec`は貼り付けません。実際のレジストリー名とパスは、必ず自分の`idms-oc-mirror.yaml`の出力に合わせます。IDMSの各項目に`mirrorSourcePolicy`がある場合は、同じ値を`sourcePolicy`という名前で転記します。出力にない場合は追加しません。

**貼り付け先ファイル：`~/ocp-airgap/install-config.yaml`の`imageDigestSources`部分**

```yaml
# 貼り付け先: ~/ocp-airgap/install-config.yaml
imageDigestSources:
  - source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
    mirrors:
      - registry.ocp.lab.example.com:8443/openshift/release
  - source: quay.io/openshift-release-dev/ocp-release
    mirrors:
      - registry.ocp.lab.example.com:8443/openshift/release-images
```

`install-config.yaml`で使用する値の参照先は次のとおりです。エディターで各ファイルを開き、次節のプレースホルダーを手動で置き換えます。pull secretは標準出力へ表示しません。

| 設定項目 | 転記元ファイル |
| --- | --- |
| `additionalTrustBundle` | `~/ocp-airgap/config/quay-rootCA.pem`の`BEGIN CERTIFICATE`と`END CERTIFICATE`を含む全内容 |
| `pullSecret` | `~/ocp-airgap/config/pull-secret-compact.json`（マージ済みの1行JSON） |
| `sshKey` | `~/.ssh/ocp-node-debug.pub` |
| `imageDigestSources` | 手順5.1 Bで整形したブロック |

<a id="step-5-2"></a>

#### 5.2 install-config.yamlを作成する

> **公式ドキュメント対応:** 3.3「設定入力の作成」および9.1「使用可能なインストール設定パラメーター」に対応します。

`install-config.yaml`では、クラスター名、ネットワーク、ノード数、プラットフォーム、ミラーレジストリーのCAと認証情報を設定します。SNOと複数ノードでは`platform`、replica数およびVIPの設定が異なります。

次のうち、作成する構成に対応する一方の内容をエディターで`install-config.yaml`として保存します。生成コマンドは使用しません。`${...}`と`<...>`の箇所は、保存前に`env.sh`の実値および手順5.1で確認した内容へ置き換えます。`imageDigestSources`のコメント行も、手順5.1 Bのブロックへ置き換えます。

##### SNO（デフォルト、`platform: none`）

**作成ファイル：`~/ocp-airgap/install-config.yaml`**

```yaml
# ファイル名: ~/ocp-airgap/install-config.yaml
apiVersion: v1
baseDomain: ${BASE_DOMAIN}
metadata:
  name: ${CLUSTER_NAME}

compute:
  - name: worker
    architecture: ${OCP_ARCH}
    replicas: 0
    platform: {}

controlPlane:
  name: master
  architecture: ${OCP_ARCH}
  replicas: 1
  platform: {}

networking:
  networkType: OVNKubernetes
  machineNetwork:
    - cidr: ${MACHINE_NETWORK_CIDR}
  clusterNetwork:
    - cidr: ${CLUSTER_NETWORK_CIDR}
      hostPrefix: ${CLUSTER_HOST_PREFIX}
  serviceNetwork:
    - ${SERVICE_NETWORK_CIDR}

platform:
  none: {}

additionalTrustBundlePolicy: Always
additionalTrustBundle: |
  <rootCA.pemのBEGIN CERTIFICATEからEND CERTIFICATEまでを、各行の先頭に空白2文字を付けて貼り付ける>

# ここへ手順5.1 BのimageDigestSourcesブロックをそのまま貼り付ける

pullSecret: '<マージ済みpull secretのJSON 1行>'
sshKey: '<SSH公開鍵 1行>'
```

SNOでは`apiVIPs`と`ingressVIPs`を設定しません。`api`、`api-int`、`*.apps`のDNS名はSNOのIPを返すようにします。

##### 複数ノード（必要な場合のみ、`platform: baremetal`）

> [!CAUTION]
> **SNOでは以下を使用しません。** SNO版と複数ノード版のどちらか一方だけを`install-config.yaml`として作成し、2つの内容を連結しないでください。

**作成ファイル：`~/ocp-airgap/install-config.yaml`（SNO版の代わりに使用）**

```yaml
# ファイル名: ~/ocp-airgap/install-config.yaml
# 複数ノード用。SNOでは使用しない
apiVersion: v1
baseDomain: ${BASE_DOMAIN}
metadata:
  name: ${CLUSTER_NAME}

compute:
  - name: worker
    architecture: ${OCP_ARCH}
    replicas: 2
    platform: {}

controlPlane:
  name: master
  architecture: ${OCP_ARCH}
  replicas: 3
  platform: {}

networking:
  networkType: OVNKubernetes
  machineNetwork:
    - cidr: ${MACHINE_NETWORK_CIDR}
  clusterNetwork:
    - cidr: ${CLUSTER_NETWORK_CIDR}
      hostPrefix: ${CLUSTER_HOST_PREFIX}
  serviceNetwork:
    - ${SERVICE_NETWORK_CIDR}

platform:
  baremetal:
    apiVIPs:
      - ${API_VIP}
    ingressVIPs:
      - ${INGRESS_VIP}

additionalTrustBundlePolicy: Always
additionalTrustBundle: |
  <rootCA.pemのBEGIN CERTIFICATEからEND CERTIFICATEまでを、各行の先頭に空白2文字を付けて貼り付ける>

# ここへ手順5.1 BのimageDigestSourcesブロックをそのまま貼り付ける

pullSecret: '<マージ済みpull secretのJSON 1行>'
sshKey: '<SSH公開鍵 1行>'
```

複数ノードでは、`apiVIPs`と`ingressVIPs`を必ず設定します。API VIPとIngress VIPは各ノードのIPと重複させません。

保存後、クラスター名、replica数、platform、VIP、`imageDigestSources`、CAおよびpull secretが選択した構成と一致することを確認します。pull secretの内容は画面へ表示しません。

<a id="step-5-3"></a>

#### 5.3 agent-config.yamlを作成する

> **公式ドキュメント対応:** 3.3「設定入力の作成」および9.2「使用可能なエージェント設定パラメーター」に対応します。

`agent-config.yaml`では、Agent ISOで起動する各ホストの役割、NIC、MACアドレス、固定IP、DNS、デフォルトルート、NTP、インストール先ディスクを設定します。

次のうち、作成する構成に対応する一方の内容をエディターで`agent-config.yaml`として保存します。生成コマンドは使用しません。`${...}`の箇所は`env.sh`の実値へ、`<DNSサーバーIP>`と`<NTPサーバーIP>`は手順1.2で各環境に設定した値へ置き換えます。

##### SNO（デフォルト、ホスト1台）

**作成ファイル：`~/ocp-airgap/agent-config.yaml`**

```yaml
# ファイル名: ~/ocp-airgap/agent-config.yaml
apiVersion: v1beta1
kind: AgentConfig
metadata:
  name: ${CLUSTER_NAME}
rendezvousIP: ${RENDEZVOUS_IP}
additionalNTPSources:
  - <NTPサーバーIP>
hosts:
  - hostname: ${SNO_HOSTNAME}
    role: master
    interfaces:
      - name: ${SNO_NIC}
        macAddress: "${SNO_MAC}"
    rootDeviceHints:
      deviceName: ${SNO_INSTALL_DISK}
    networkConfig:
      interfaces:
        - name: ${SNO_NIC}
          type: ethernet
          state: up
          mac-address: "${SNO_MAC}"
          ipv4:
            enabled: true
            address:
              - ip: ${SNO_IP}
                prefix-length: ${NODE_PREFIX_LENGTH}
            dhcp: false
      dns-resolver:
        config:
          server:
            - <DNSサーバーIP>
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: ${DEFAULT_GATEWAY}
            next-hop-interface: ${SNO_NIC}
            table-id: 254
```

SNOでは、`rendezvousIP`にSNO自身の固定IPを指定します。

##### 複数ノード（必要な場合のみ、control plane 3台、worker 2台）

> [!CAUTION]
> **SNOでは以下を使用しません。** SNO版と複数ノード版のどちらか一方だけを`agent-config.yaml`として作成し、2つの内容を連結しないでください。

**作成ファイル：`~/ocp-airgap/agent-config.yaml`（SNO版の代わりに使用）**

```yaml
# ファイル名: ~/ocp-airgap/agent-config.yaml
# 複数ノード用。SNOでは使用しない
apiVersion: v1beta1
kind: AgentConfig
metadata:
  name: ${CLUSTER_NAME}
rendezvousIP: ${RENDEZVOUS_IP}
additionalNTPSources:
  - <NTPサーバーIP>
hosts:
  - hostname: ${MASTER0_HOSTNAME}
    role: master
    interfaces:
      - name: ${MASTER0_NIC}
        macAddress: "${MASTER0_MAC}"
    rootDeviceHints:
      deviceName: ${MASTER0_INSTALL_DISK}
    networkConfig:
      interfaces:
        - name: ${MASTER0_NIC}
          type: ethernet
          state: up
          mac-address: "${MASTER0_MAC}"
          ipv4:
            enabled: true
            address:
              - ip: ${MASTER0_IP}
                prefix-length: ${NODE_PREFIX_LENGTH}
            dhcp: false
      dns-resolver:
        config:
          server:
            - <DNSサーバーIP>
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: ${DEFAULT_GATEWAY}
            next-hop-interface: ${MASTER0_NIC}
            table-id: 254

  - hostname: ${MASTER1_HOSTNAME}
    role: master
    interfaces:
      - name: ${MASTER1_NIC}
        macAddress: "${MASTER1_MAC}"
    rootDeviceHints:
      deviceName: ${MASTER1_INSTALL_DISK}
    networkConfig:
      interfaces:
        - name: ${MASTER1_NIC}
          type: ethernet
          state: up
          mac-address: "${MASTER1_MAC}"
          ipv4:
            enabled: true
            address:
              - ip: ${MASTER1_IP}
                prefix-length: ${NODE_PREFIX_LENGTH}
            dhcp: false
      dns-resolver:
        config:
          server:
            - <DNSサーバーIP>
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: ${DEFAULT_GATEWAY}
            next-hop-interface: ${MASTER1_NIC}
            table-id: 254

  - hostname: ${MASTER2_HOSTNAME}
    role: master
    interfaces:
      - name: ${MASTER2_NIC}
        macAddress: "${MASTER2_MAC}"
    rootDeviceHints:
      deviceName: ${MASTER2_INSTALL_DISK}
    networkConfig:
      interfaces:
        - name: ${MASTER2_NIC}
          type: ethernet
          state: up
          mac-address: "${MASTER2_MAC}"
          ipv4:
            enabled: true
            address:
              - ip: ${MASTER2_IP}
                prefix-length: ${NODE_PREFIX_LENGTH}
            dhcp: false
      dns-resolver:
        config:
          server:
            - <DNSサーバーIP>
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: ${DEFAULT_GATEWAY}
            next-hop-interface: ${MASTER2_NIC}
            table-id: 254

  - hostname: ${WORKER0_HOSTNAME}
    role: worker
    interfaces:
      - name: ${WORKER0_NIC}
        macAddress: "${WORKER0_MAC}"
    rootDeviceHints:
      deviceName: ${WORKER0_INSTALL_DISK}
    networkConfig:
      interfaces:
        - name: ${WORKER0_NIC}
          type: ethernet
          state: up
          mac-address: "${WORKER0_MAC}"
          ipv4:
            enabled: true
            address:
              - ip: ${WORKER0_IP}
                prefix-length: ${NODE_PREFIX_LENGTH}
            dhcp: false
      dns-resolver:
        config:
          server:
            - <DNSサーバーIP>
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: ${DEFAULT_GATEWAY}
            next-hop-interface: ${WORKER0_NIC}
            table-id: 254

  - hostname: ${WORKER1_HOSTNAME}
    role: worker
    interfaces:
      - name: ${WORKER1_NIC}
        macAddress: "${WORKER1_MAC}"
    rootDeviceHints:
      deviceName: ${WORKER1_INSTALL_DISK}
    networkConfig:
      interfaces:
        - name: ${WORKER1_NIC}
          type: ethernet
          state: up
          mac-address: "${WORKER1_MAC}"
          ipv4:
            enabled: true
            address:
              - ip: ${WORKER1_IP}
                prefix-length: ${NODE_PREFIX_LENGTH}
            dhcp: false
      dns-resolver:
        config:
          server:
            - <DNSサーバーIP>
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: ${DEFAULT_GATEWAY}
            next-hop-interface: ${WORKER1_NIC}
            table-id: 254
```

複数ノードでは、全5台を省略せず`hosts`へ記載します。`rendezvousIP`はcontrol plane 0の実IPであり、API VIPではありません。

次を確認します。

- `apiVersion`が`v1beta1`である。
- `metadata.name`が`install-config.yaml`のクラスター名と一致する。
- SNOではホスト1台、複数ノードではmaster 3台とworker 2台がある。
- MACアドレスとIPアドレスが各ホストで一意である。
- NIC名とインストール先ディスクが実機に存在する。
- `rendezvousIP`が`master`ロールのホストに割り当てられている。

本書のテスト値ではインストール先を`/dev/sda`としています。複数ディスクを搭載する実機では、誤消去を防ぐため`/dev/disk/by-path/...`などの安定したデバイス名を使用します。

<a id="step-5-4"></a>

#### 5.4 Agent ISOを生成する

> **公式ドキュメント対応:** 3.4「エージェントイメージの作成と起動」に対応します。

`openshift-install-fips`は`--dir`で指定したディレクトリーから設定ファイルを読み込みます。`install-config.yaml`と`agent-config.yaml`を`iso-create`へコピーしてからISOを生成します。

```bash
cd ~/ocp-airgap
mkdir -p iso-create
cp install-config.yaml iso-create/install-config.yaml
cp agent-config.yaml iso-create/agent-config.yaml
openshift-install-fips agent create image --dir iso-create \
  && echo "OK: Agent ISO creation completed"
```

**出力例:**

```text
...
OK: Agent ISO creation completed
```

この処理では、2つのYAML、ノード数、platform、VIP、MACアドレス、ネットワーク設定およびミラーからのrelease image取得が検証されます。エラーが表示された場合はISOを加工せず、設定値を修正して新しく生成します。

次のファイルが生成されていることを確認します。

- Agent ISO：`~/ocp-airgap/iso-create/agent.x86_64.iso`
- kubeconfig：`~/ocp-airgap/iso-create/auth/kubeconfig`
- kubeadmin password：`~/ocp-airgap/iso-create/auth/kubeadmin-password`

```bash
ls -lh \
  iso-create/agent.x86_64.iso \
  iso-create/auth/kubeconfig \
  iso-create/auth/kubeadmin-password

test -s iso-create/agent.x86_64.iso \
  && test -s iso-create/auth/kubeconfig \
  && test -s iso-create/auth/kubeadmin-password \
  && echo "OK: Agent ISO and auth files"
```

**出力例:**

```text
-rw-r--r--. 1 user user 1.3G ... iso-create/agent.x86_64.iso
-rw-------. 1 user user 8.8K ... iso-create/auth/kubeconfig
-rw-------. 1 user user   23 ... iso-create/auth/kubeadmin-password
OK: Agent ISO and auth files
```

ISO生成時に`ERROR`が表示された場合は、そのまま次へ進まず設定を修正して再生成します。

ISO、kubeconfig、kubeadmin passwordには機密情報が含まれるため、公開リポジトリーへ登録しません。

<a id="step-6"></a>

### 6. ノードを起動してインストールを監視する

> **公式ドキュメント対応:** 3.4「エージェントイメージの作成と起動」、3.5「現在のインストールホストがリリースイメージをプルできることを確認する」、3.6「インストールの進行状況の追跡と確認」に対応します。ISO生成後の作業を公式手順で補っています。

生成した`~/ocp-airgap/iso-create/agent.x86_64.iso`を、BMCの仮想メディアまたは起動用メディアとして各OpenShiftノードへ割り当てます。専用のBootstrapノードは用意しません。

**SNOの場合:**

1. SNOノードへISOを割り当てる。
2. SNOノードをISOから起動する。
3. SNOノード自身がランデブーホストとなり、インストール中の初期構築処理を実行する。

**複数ノードの場合:**

1. 5台すべてへ同じISOを割り当てる。
2. `MASTER0_IP`を設定したcontrol plane 0を起動する。このノードがランデブーホストとなる。
3. 残りのcontrol plane 2台とworker 2台を起動する。

各ノードの起動画面で、次を確認します。

- 想定したホスト名、NIC、固定IP、デフォルトゲートウェイが表示される。
- DNSとNTPへ到達できる。
- インストール先ディスクが正しい。
- ミラーレジストリーからrelease imageを取得できる。

**実行:** 作業用ホストでBootstrap完了を待機します。

```bash
cd ~/ocp-airgap
openshift-install-fips agent wait-for bootstrap-complete \
  --dir iso-create \
  --log-level=info \
  && echo "OK: bootstrap complete"
```

**出力例:**

```text
INFO ... Bootstrap is complete
OK: bootstrap complete
```

タイムアウトやホスト未検出のエラーが表示された場合は、インストール完了待機へ進みません。

Bootstrap完了後、インストール完了を待機します。

```bash
openshift-install-fips agent wait-for install-complete \
  --dir iso-create \
  --log-level=info \
  && echo "OK: install complete"
```

**出力例:**

```text
INFO Install complete!
INFO To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=...'
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.ocp.lab.example.com
OK: install complete
```

処理が停止した場合は、DNS、NTP、ミラーレジストリー、MACアドレス、インストール先ディスクおよび`rendezvousIP`を確認します。

<a id="step-7"></a>

### 7. インストール後の状態を確認する

> **公式ドキュメント対応:** 3.6「インストールの進行状況の追跡と確認」に対応します。`cluster-resources`の適用はoc-mirrorプラグインv2手順に対応し、その後の確認コマンドは本書の運用確認です。

**設定:** 生成されたkubeconfigを使用するため、環境変数を読み込みます。

```bash
export KUBECONFIG="$HOME/ocp-airgap/iso-create/auth/kubeconfig"
```

**設定:** 閉域クラスターが外部の既定Operatorカタログへ接続しないように、既定ソースを無効化します。手順4.2でOperatorを残した場合は、この後に`oc-mirror`生成のカタログを適用します。`operators:`ブロックを削除してOCP本体だけをミラーした場合は、使用可能なカタログが表示されない状態で正常です。

```bash
oc patch OperatorHub cluster --type json \
  -p '[{"op":"add","path":"/spec/disableAllDefaultSources","value":true}]'
```

**出力例:**

```text
operatorhub.config.openshift.io/cluster patched

# 再実行時
operatorhub.config.openshift.io/cluster patched (no change)
```

**適用:** `oc-mirror`が生成したミラー定義、release signature、およびOperatorを選択した場合のCatalogSourceをクラスターへ適用します。`install-config.yaml`の`imageDigestSources`はインストール中のミラー参照に使い、この手順はインストール後のクラスターに生成済みリソースを登録するために必要です。

```bash
oc apply -f \
  "$HOME/ocp-airgap/oc-mirror-work/working-dir/cluster-resources"
```

**出力例:**

```text
imagedigestmirrorset.config.openshift.io/idms-oc-mirror created
configmap/oc-mirror-signature-... created
catalogsource.operators.coreos.com/cs-redhat-operator-index-v4-21 created

# 再実行時
imagedigestmirrorset.config.openshift.io/idms-oc-mirror unchanged
configmap/oc-mirror-signature-... unchanged
catalogsource.operators.coreos.com/cs-redhat-operator-index-v4-21 configured
```

まず、`oc-mirror v2`が生成したImageDigestMirrorSetの名前が1件以上表示されることを確認します。既存リソースと混同しないように、`createdBy: oc-mirror v2`アノテーションで絞り込みます。

```bash
oc get imagedigestmirrorset.config.openshift.io -o json \
  | jq -r '.items[] | select(.metadata.annotations.createdBy == "oc-mirror v2") | .metadata.name'
```

**出力例:**

```text
idms-oc-mirror
```

生成バージョンによって名前が異なる場合は、`cluster-resources`内の定義名と一致することを確認します。

Operatorまたはタグミラーを含めた場合は、生成された種類だけ、次の確認コマンドの`#`を外します。oc-mirror v2の出力方式によってCatalogSourceまたはClusterCatalogが生成されるため、`cluster-resources`に実在する種類を確認します。

```bash
# oc get imagetagmirrorset.config.openshift.io -o json \
#   | jq -r '.items[] | select(.metadata.annotations.createdBy == "oc-mirror v2") | .metadata.name'
# oc get catalogsource.operators.coreos.com -n openshift-marketplace -o json \
#   | jq -r '.items[] | select(.metadata.annotations.createdBy == "oc-mirror v2") | .metadata.name'
# oc get clustercatalog.olm.operatorframework.io -o json \
#   | jq -r '.items[] | select(.metadata.annotations.createdBy == "oc-mirror v2") | .metadata.name'
```

**出力例:**

```text
itms-oc-mirror
cs-redhat-operator-index-v4-21
redhat-operator-index-v4-21
```

実際には、コメントを外して実行した種類のうち、`cluster-resources`に生成されたリソースだけが表示されます。

**確認:** クラスターとOperatorの状態を確認します。

```bash
oc whoami
oc get nodes -o wide
oc get clusterversion
oc get clusteroperators
```

**出力例（SNO）:**

```text
kube:admin

NAME    STATUS   ROLES                         AGE   VERSION    INTERNAL-IP      ...
sno-0   Ready    control-plane,master,worker   ...   v1.34...   192.168.100.20   ...

NAME      VERSION   AVAILABLE   PROGRESSING   SINCE   STATUS
version   4.21.21   True        False         ...     Cluster version is 4.21.21

NAME                         VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE   MESSAGE
authentication               4.21.21   True        False         False      ...
console                      4.21.21   True        False         False      ...
...
```

**出力例（複数ノード）:**

```text
kube:admin

NAME       STATUS   ROLES                  AGE   VERSION    INTERNAL-IP      ...
master-0   Ready    control-plane,master   ...   v1.34...   192.168.100.21   ...
master-1   Ready    control-plane,master   ...   v1.34...   192.168.100.22   ...
master-2   Ready    control-plane,master   ...   v1.34...   192.168.100.23   ...
worker-0   Ready    worker                 ...   v1.34...   192.168.100.31   ...
worker-1   Ready    worker                 ...   v1.34...   192.168.100.32   ...

NAME      VERSION   AVAILABLE   PROGRESSING   SINCE   STATUS
version   4.21.21   True        False         ...     Cluster version is 4.21.21

NAME                         VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE   MESSAGE
authentication               4.21.21   True        False         False      ...
console                      4.21.21   True        False         False      ...
...
```

異常なPodが残っていないか確認します。

```bash
oc get pods -A \
  --field-selector=status.phase!=Running,status.phase!=Succeeded
```

**出力例:**

```text
No resources found
```

Pod名が表示された場合は、そのPodの状態とイベントを確認してから完了とします。

最後に、Web Consoleへアクセスできることを確認します。

```text
https://console-openshift-console.apps.<cluster>.<baseDomain>
```

本書のテスト値では、次のURLです。

```text
https://console-openshift-console.apps.ocp.lab.example.com
```

以上で、Agent-based Installerを使用した閉域環境へのOpenShift Container Platform 4.21インストールは完了です。

本手順の技術内容は、次のOpenShift Container Platform 4.21公式資料を基準にしています。

- [Agent-based Installerを使用したオンプレミスクラスターのインストール](https://docs.redhat.com/ja/documentation/openshift_container_platform/4.21/html-single/installing_an_on-premise_cluster_with_the_agent-based_installer/index)
- [oc-mirrorプラグインv2を使用した非接続インストールのイメージミラーリング](https://docs.redhat.com/ja/documentation/openshift_container_platform/4.21/html/disconnected_environments/about-installing-oc-mirror-v2)
- [OpenShift Installer release-4.21のinstall-config型定義](https://github.com/openshift/installer/blob/release-4.21/pkg/types/installconfig.go)
- [Single-node OpenShiftのインストール前要件](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/installing_on_a_single_node/preparing-to-install-sno)
- [OpenShift Virtualizationのインストール要件](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/virtualization/installing)
