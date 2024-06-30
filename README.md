# SISF Device Tracking

<!--
[!NOTE]
[!TIP]
[!IMPORTANT]
[!WARNING]
[!CAUTION]
-->

Cisco SISFベースのデバイストラッキングを用いて、端末でのIPアドレス重複を検知できるか、の観点で動作検証を行いました。

その結果をまとめます。

<br>

# 検証結果のまとめ

先に結果を書きます。

SISFベースのデバイストラッキングで実現できること、できないことを要約します。

<br>

## できること

- スイッチの配下に存在する端末の情報（MACアドレス/IPアドレス/接続ポート/接続VLAN）を知ることができます

- 端末でIPアドレスの重複が発生したときに、スイッチでそれを検知できます

- アドレス重複を検知した際、それをSYSLOGで通知できます

- アドレス重複が発生した際、どちらか一方の端末の通信を保護できます（SISFがない場合は、両方ともスローダウン）

<br>

## できないこと

- スイッチの配下に存在する端末の情報（バインディングテーブルの情報）を外部にエクスポートするには、何らかの工夫が必要です

<br>

## 検討すべきこと

IPアドレスの重複を検知すると `%SISF-4-IP_THEFT: IP Theft IP=192.168.10.102` で始まるログをSYSLOGに送信します。
SYSLOGサーバでこれを受け取ったときに、次にどういうアクションを取るべきか、は検討項目の一つです。
少なくとも、検知したスイッチはどれか、どのポートか、どのIPアドレスか、どのMACアドレスか、は分かります。
アドレスが重複した場合にどっちが正しいのか、までは判断つかないと思いますので、
これら情報を抽出して管理者に通報する、というのが望まれるアクションのように思います。

インベントリ管理の観点で、どのようなMACアドレスのデバイスがネットワーク上に存在するのか、を調べる必要があります。
デバイストラッキングのバインディングテーブルがまさにそれなのですが、個々のスイッチのメモリ上にしか存在しませんので、何とかして定期的に収集する必要があります。

> [!TIP]
> DNA Essentialライセンスを持ったCatalyst 9300のようにEEMをサポートした機種であれば、定期的に装置自ら外部のサーバに転送することができます。
> そうでない装置は、Ansibleのような自動化ツールを用いて外から装置に乗り込んで収集するしかなさそうです。

<br>

## 推奨される設計

- コアスイッチでSISFを動かす必要はありません

- 端末がつながるリーフスイッチ（エッジスイッチ）でSISFを有効にします

- リーフスイッチではアップリンクとアクセスポートが存在しますので、デバイストラッキングポリシーを２つ作成します

- IPv6を使っていない場合は、IPv6での情報収集をしないよう、ポリシーを変更します

- スイッチまたぎで発生するIPアドレス重複を検知するには、アップリンクから来る情報も収集します（trusted-portを設定しません）

- SISFで検知したイベントはデフォルトで通知されませんので、アドレス重複を通知したい場合は `device-tracking logging thef` を設定します

- L2スイッチは端末がつながるVLANに足を出していないケースがほとんどですので、`device-tracking tracking auto-source` を設定します

> [!WARNING]
> アップリンクから来る情報を収集すると、自装置配下に**いない**端末の情報もバインディングテーブルにエントリされ、CPUとメモリを多く消費します

<br><br><br>

# マニュアル

**設定ガイド**

https://www.cisco.com/c/ja_jp/td/docs/switches/lan/catalyst9200/software/release/17-2/configuration_guide/sec/b_172_sec_9200_cg/configuring_sisf_based_device_tracking.html


**トラブルシューティング**

https://www.cisco.com/c/ja_jp/support/docs/switches/catalyst-9300-series-switches/221562-troubleshoot-sisf-on-catalyst-9000-serie.html

<br><br>

> [!TIP]
> たどり方
>
> cisco.com → 上部のSupportをクリック → View all product support → Switches → Catalyst 9200 Series Switches → Configuration Guidesのところにバージョンごとのコンフィグガイドがあります。コンフィグガイドの Security → Configuring Switch Integrated Security Features の順にたどります。

> [!NOTE]
> 2024年6月現在　Feature Navigatorで探しても出てきません。なぜ？

<br>

## SISF Device Trackingとは

SISFは<u>S</u>witch <u>I</u>ntegrated <u>S</u>ecurity <u>F</u>eaturesの略です。

エンドノードの存在、位置、移動を検出する仕組みです。
かつてIP Device Trackingという名前で実装されていたものと、IPv6のセキュリティ機能を統合したものです。

> [!NOTE]
>
> 古いCatalystでは SISF は動きませんが、旧体系の設定を変換することはできそうです。
>
> 逆に新しいCatalystでは、IP Device Trackingの設定コマンドは無くなっています。

スイッチは受信したトラフィックを分析してデバイスを識別する情報としてMACアドレスとIP アドレスを抽出して、バインディングテーブルに保存します。

SISFが構築するバインディングテーブルを必要とする機能もあり、それらをSISFクライアントと呼びます。
代表的なSISFクライアントはこれらです。

- LISP/EVPN
- 802.1X
- Web認証
- Cisco TrustSec
- DHCPスヌーピング

SISFクライアントを有効にすると、バインディングテーブルを作るのに必要なSISFポリシーが自動で作成されます。

SISFはIPv4とIPv6の両方で動作します。

> [!NOTE]
> IPv6を使っていない環境では、SISFポリシーの設定を変更してIPv6での情報収集を停止した方がいいです。無用なログが表示されます。

SISFではDHCP、ARP、ND、RAなどのパケットを収集してテーブルを作成します。

これらパケットを送信しないサイレントノードがいる場合は、スタティックにエントリを設定することもできます。

バインディングテーブルには、次のような情報が含まれます。

- IPv4/IPv6アドレス
- MACアドレス
- ホストが存在するインタフェース、VLAN
- 到達可能性情報

到達可能性はいくつかの状態があり、代表的なのは`Reachable`、`Stale`、`Down`などです。

<dl>
    <dt>Reachable</dt>
    <dd>端末から来た制御パケットがスイッチで正しく処理されていることが確認されている状態です。このステートのライフタイムは <strong>5分</strong> です。</dd>
    <dt>Stale</dt>
    <dd>Reachableステートのライフフタイムが切れ、パケットの送信が確認できていない状態です。このステートのライフタイムは <strong>24時間</strong> です。</dd>
    <dt>Down</dt>
    <dd>ホストが繋がっているインタフェースがダウンしている状態です。このステートのライフタイムは <strong>24時間</strong> です。</dd>
</dl>

ライフタイムは調整できますが、デフォルトでは端末の存在を確認してから概ね24時間、テーブルにエントリが残ります。

バインディングテーブルのエントリを手動で消すこともできます。一括で全て消すこともできますし、IPアドレスやMACアドレス、インタフェースを指定して個別にエントリを消すこともできます。

SISFは**デフォルトで無効**になっています。

SISFを有効化するには、デバイストラッキングポリシーをインタフェースもしくはVLANにアタッチします。

<br>

> [!NOTE]
> ポリシーを指定せずに有効化することもできますが、その場合は **default** という名前のポリシーが作られてアタッチされます。
> defaultポリシーの内容は変更できません。

<br>

SISFで収集した情報をどのように扱うか、を選択できます。選択肢は `Glean` と `Inspect` と `Guard` です。

<dl>
<dt>Glean</dt>
<dd>収集するだけ</dd>
<dt>Inspect</dt>
<dd>内容を分析して場合によっては通知</dd>
<dt>Guard</dt>
<dd>保護動作をし、不正なパケットは破棄します</dd>
</dl>

デフォルトは`Guard`です。

<br>

## 設計項目

どこでSISFデバイストラッキングを有効にするか、を設計します。

コアスイッチのように直接端末が繋がらない装置ではSISFを動かす必要はありません。
端末を接続するスイッチで有効にします。

SISFを有効にする、というのはSISFポリシーをアタッチする、ということです。
アタッチする対象はインタフェースもしくはVLANです。

同じスイッチでも上位のスイッチと接続するいわゆるアップリンクのポートと、端末が接続するアクセスポートが存在するのが一般的だと思います。
アップリンクとアクセスポートで、デバイストラッキングの動作を変えるために、ポリシーを２個作ってポートごとにアタッチするポリシーを使い分けるのが良さそうです。

通常、L2スイッチには管理用としてIPアドレスを割り当て、端末が接続するVLANに対してはIPアドレスを持っていない場合の方が多いと思います。
すると、スイッチが送信するプローブのパケットの送信元IPアドレスが0.0.0.0となり、周辺環境に悪影響が出がちです。
SVIにIPアドレスを割り当てるように設計してもよいのですが、スイッチ内に多数のVLANが存在する場合、それぞれにアドレスを持たせるのは現実的ではありません。
`device-tracking tracking auto-source` を設定すれば悪影響を回避できる、とマニュアルに記載されていますので、これを忘れずに設定するようにします。

<br>

## 設定方法

デフォルト動作は無効ですので、手動で有効化します。

ポリシーを作成して、インタフェース（もしくはVLAN）にポリシーをアタッチします。

### ポリシーの作成方法

`device-tracking policy <名前>` で新規に作成します。

ポリシーの中で設定できる項目はこれらです。

```
leaf-sw4(config-device-tracking)#?
device-tracking policy configuration mode:
  data-glean            binding recovery by data traffic source address
                        gleaning
  default               Set a command to its defaults
  destination-glean     binding recovery by data traffic destination address
                        gleaning
  device-role           Sets the role of the device attached to the port
  distribution-switch   Distribution switch to sync with
  exit                  Exit from device-tracking policy configuration mode
  limit                 Specifies a limit
  medium-type-wireless  Force medium type to wireless
  no                    Negate a command or set its defaults
  prefix-glean          Glean prefixes in RA and DHCP-PD traffic
  protocol              Sets the protocol to glean (default all)
  security-level        setup security level
  tracking              Override default tracking behavior
  trusted-port          setup trusted port
  vpc                   setup vpc port
```




**destination-glean**

ネットワーク内の送信元からスヌーピングされたデータパケットからのアドレスの学習を有効にし、データトラフィックの送信元アドレスとともにバインディングテーブルを読み込みます。

**prefix-glean**

IPv6 RA、もしくはDHCP-PDのどちらかでプレフィクスを学習します

**device-role node**

これがデフォルトです。接続している装置が端末である、という前提で動作します。

**device-role switch**

ポートに接続しているのがスイッチの場合は、これを設定します。
そのポートでのバインディングエントリの作成は行いません。

<br>

> [!NOTE]
> **destination-switch** という設定が？キーで表示されるコマンド一覧に表示されますが、実装されていないので設定しても何もおこりません。

<br>

**protocol**

protocolに続く設定はこれらです。デフォルトで全て有効になっています。

```text
  arp    Glean addresses in ARP packets
  dhcp4  Glean addresses in DHCPv4 packets
  dhcp6  Glean addresses in DHCPv6 packets
  ndp    Glean addresses in NDP packets
  udp    Gleaning from UDP packets
```

<br>

> [!NOTE]
> **protocol udp** という設定が？キーで表示されるコマンド一覧に表示されますが、実装されていないので設定しても何もおこりません。

<br>

**security-level**

security-levelに続く設定はこれらです。デフォルトはguardです。不正なパケットは破棄してくれます。

```text
leaf-sw4(config-device-tracking)#security-level ?
  glean    glean addresses passively
  guard    inspect and drop un-authorized messages (default)
  inspect  glean and Validate message
```

**tracking**

trackingに続く設定はこれらです。デフォルトでは**トラッキングしません**。

```text
leaf-sw4(config-device-tracking)#tracking ?
  disable  Tracking off (default)
  enable   Tracking on
```

**trusted-port**

信頼できるポートであることを示します。
他のポートで学習したバインディング情報よりも優先されます。
このポートは信頼できるので、ガードする動作はしなくなります。

<br>

## 検証構成

CMLで完結するように構成します。

![検証構成図](/asset/expt-sisf-1.png)

spine-sw1, spine-sw2 は中継だけを行うコアスイッチで、端末が直接接続されることはありません。
実態はCatalyst 9000vです。

leaf-sw3, leaf-sw4は 端末が直接つながるスイッチです。
実態はCatalyst 9000vです。

場合によってはノンインテリスイッチでポートを増やしてから端末がつながることもありえますし、無線LANのアクセスポイントの先に端末がいるかもしれません。
ここでは管理外スイッチを接続して、その先に端末を接続しています。

これらスイッチ sw1 - sw4 間はVLANをトランクします。

sw1-sw3-sw2-sw4-sw1というように8の字でループを形成しますので、スパニングツリーを有効にしてL2ループを排除します。
スパニングツリーのモードはMSTにします。インスタンス0に全VLANを紐づけます（デフォルト動作です）。
sw1がroot primary、sw2がroot secondaryとなるように設定します。
その結果、リーフスイッチのアップリンクのどれか一つがブロックポートになります。

host-6001, host-6002, host-6003, host-6004 は端末です。実態はIOSvです。固定でIPアドレスを設定することもあればDHCPでアドレスをもらうこともあります。

r5は外部との境界線に置かれたルータです。
端末からみるとこれがデフォルトゲートウェイになります。自分以外のVLANはすべてこのルータの先にあるからです。
DHCPサーバもこのルータが担います。

sv-6005 はUbuntuで、SYSLOGサーバを担います。
全てのスイッチはSYSLOGでこのサーバにログを転送しますので、
このサーバでログファイルを`tail-f`しておけば全てのスイッチのログを即時で確認できます。

VLAN 10 は端末が接続するVLANで、サブネットは 192.168.**10**.0/24 です。

VLAN 254 はスイッチの管理用VLANで、サブネットは 192.168.**254**.0/24 です。

r5はVLAN 254とVLAN 10の両方に足を出しています。

<br>

## Ubuntuの設定

Ubuntuの初期設定、特にログインアカウントやネットワーク設定はCMLで設定してしまった方が楽です。

CMLで以下の設定に変更します。

- ens2のIPアドレスは192.168.100.100/24
- デフォルトゲートウェイは192.168.100.5
- DNSの参照先は192.168.254.5

```yaml
#cloud-config
hostname: inserthostname-here
manage_etc_hosts: True
system_info:
  default_user:
    name: cisco
password: cisco
chpasswd: { expire: False }
ssh_pwauth: True
ssh_authorized_keys:
  - your-ssh-pubkey-line-goes-here
timezone: Asia/Tokyo
locale: ja_JP.utf8
write_files:
  - path: /etc/netplan/50-cloud-init.yaml
    content: |
      network:
        ethernets:
          ens2:
            addresses:
              - 192.168.100.100/24
            gateway4: 192.168.100.5
            dhcp4: false
            nameservers:
              addresses:
                - 192.168.254.5
        version: 2
runcmd:
  - sudo netplan apply
```

Ubuntuを起動してログインします。

シリアルコンソールで操作しているときにviを使うと画面サイズが正しく反映されないので、以下を流し込んで対策します。

```bash
cat - << 'EOS' >> ~/.bashrc
rsz () if [[ -t 0 ]]; then local escape r c prompt=$(printf '\e7\e[r\e[999;999H\e[6n\e8'); IFS='[;' read -sd R -p "$prompt" escape r c; stty cols $c rows $r; fi
rsz
EOS
```

> [!NOTE]
> 元ネタ
> https://wiki.archlinux.org/title/working_with_the_serial_console

<br>

### TFTPサーバの設定

```bash
sudo su
sudo apt update
sudo apt install tftpd-hpq
```

設定ファイル `/etc/default/tftpd-hpq` を書き換えます。

```text
TFTP_USERNAME="tftp"
TFTP_DIRECTORY="/var/lib/tftpboot"
TFTP_ADDRESS=":69"
# TFTP_OPTIONS="--secure"
```

設定ファイルで指定したディレクトリを作成します。

```bash
sudo mkdir /var/lib/tftpboot
sudo chmod -R 777 /var/lib/tftpboot
sudo chown -R nobody:nogroup /var/lib/tftpboot
```

systemctlで再起動します。自動起動の設定もやっておきます。

```bash
sudo systemctl restart tftpd-hpa
sudo systemctl enable tftpd-hpa
```

Cisco装置からファイルを書き込むには、先にファイルを作成し、書き込み権限を付与しなければいけません。

TFTPサーバで下準備。

```bash
touch /var/lib/tftpboot/sw3-dt-database.txt
chmod a+w /var/lib/tftpboot/sw3-dt-database.txt
```

Cisco装置から、そのファイルにコピーするには、こうします。

```
show device-tracking database | redirect tftp://192.168.100.100//var/lib/tftpboot/sw3-dt-database.txt
```

<br>

### rsyslog設定(Ubuntu)

Ubuntuには最初からrsyslogがインストールされていますので、設定を追加してネットワーク機器からのsyslogを受け取れるようにします。

`/etc/rsyslog.conf` を変更します。
以下の2行のコメントを外すと、外部からのUDP syslogを受けとるようになります。

```text
#module(load="imudp")
#input(type="imudp" port="514")
```

rsyslogは `/etc/rsyslog.d/` ディレクトリにファイルを置くと、追加の設定ファイルとして読みこんでくれます。

ファシリティはlocal7（シスコのデフォルト）を使うことにして、以下を流し込みます。

```bash
sudo su
cat - << 'EOS' > /etc/rsyslog.d/local7.conf
local7.* /var/log/local7.log
EOS
```

Ubuntuのrsyslog設定は以上なので、ここで一度、再起動しておきます。

<br>

### logging設定(Cisco)

Cisco機器ではloggingコマンドでSYSLOGの設定をします。

全機器共通で以下を流し込みます。

```text
logging buffered 512000
logging host 192.168.100.100
logging facility local7
logging trap informational
```

> [!TIP]
> `logging facility local7` はデフォルトの設定なので、省略しても構いません。

> [!TIP]
> `logging trap informational` はデフォルト設定なので、省略しても構いません。

デフォルトでは `informational` (レベル値 6) 以上がロギングされます。
つまりデバッグメッセージ以外はすべてSYSLOGに送信されますので、
多すぎる場合は `logging trap` 設定でレベルを変更します。
まず欲しい情報がどのレベル値で飛んでいているか確認して、送信レベルを決めるとよいでしょう。

| level設定     | 値  | 意味                | UNIX相当    |
| ------------- | -- | ------------------- | ----------- |
| emergencies   | 0  | システムが不安定      | LOG_EMERG  |
| alerts        | 1  | ただちに対応が必要    | LOG_ALERT   |
| critical      | 2  | クリティカル          | LOG_CRIT   |
| errors        | 3  | エラー               | LOG_ERR     |
| warnings      | 4  | 警告                 | LOG_WARNING |
| notifications | 5  | 正常動作内だが、要注意 | LOG_NOTICE |
| informational | 6  | 情報メッセージ         | LOG_INFO   |
| debugging     | 7  | デバッグメッセージ     | LOG_DEBUG  |


以下の作業では、Ubuntuのコンソールを開いて、SYSLOGを表示しながら進めていきます。

```bash
tail -f /var/log/local7.log
```

<br>

# 各装置の設定

## spine-sw1

関係しないところを省略しています。

```bash
!
! Last configuration change at 16:35:02 JST Sat Jun 29 2024
!
version 17.10
service timestamps debug datetime msec localtime show-timezone
service timestamps log datetime msec localtime show-timezone
!
hostname spine-sw1
!
logging buffered 512000
clock timezone JST 9 0
switch 1 provision c9kv-q200-8p
!
ip name-server 192.168.254.5
ip domain name iida.local
!
vtp mode transparent
!
spanning-tree mode mst
spanning-tree extend system-id
!
spanning-tree mst configuration
 name all_region
 revision 1
!
spanning-tree mst 0 priority 24576
!
vlan dot1q tag native
!
vlan 10,254
!
interface GigabitEthernet1/0/1
 switchport mode trunk
 switchport nonegotiate
!
interface GigabitEthernet1/0/2
 switchport mode trunk
 switchport nonegotiate
!
interface Vlan1
 no ip address
!
interface Vlan254
 ip address 192.168.254.1 255.255.255.0
!
ip default-gateway 192.168.254.5
!
logging host 192.168.100.100
!
ntp source Vlan254
ntp server 192.168.254.5
!
```

> [!TIP]
> `spanning-tree mst 0 priority 24576` は自分で設定したものではありません。
> `spanning-tree mst 0 root primary` として設定したものが変換されたものです。

sw1はスパニングツリーのルートブリッジです。
`show spanning-tree`の表示で `This bridge is the root` となっていれば期待通りです。

```text
spine-sw1#show spanning-tree

MST0
  Spanning tree enabled protocol mstp
  Root ID    Priority    24576
             Address     5254.0018.a228
             This bridge is the root
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    24576  (priority 24576 sys-id-ext 0)
             Address     5254.0018.a228
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec

Interface           Role Sts Cost      Prio.Nbr Type
------------------- ---- --- --------- -------- --------------------------------
Gi1/0/1             Desg FWD 20000     128.1    P2p
Gi1/0/2             Desg FWD 20000     128.2    P2p
Gi1/0/3             Desg FWD 20000     128.3    P2p
Gi1/0/4             Desg FWD 20000     128.4    P2p
Gi1/0/5             Desg FWD 20000     128.5    P2p
Gi1/0/6             Desg FWD 20000     128.6    P2p
Gi1/0/7             Desg FWD 20000     128.7    P2p
Gi1/0/8             Desg FWD 20000     128.8    P2p
```

sw1はL2スイッチですので、デフォルトゲートウェイとしてVLAN 254上にいる r5 (192.168.254.5) を指しています。
VLAN 10のアドレスは持っていません。

```text
spine-sw1#show ip route
Extended Host Mode is enabled
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2, m - OMP
       n - NAT, Ni - NAT inside, No - NAT outside, Nd - NAT DIA
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       H - NHRP, G - NHRP registered, g - NHRP registration summary
       o - ODR, P - periodic downloaded static route, l - LISP
       a - application route
       + - replicated route, % - next hop override, p - overrides from PfR
       & - replicated local route overrides by connected

Gateway of last resort is 192.168.254.5 to network 0.0.0.0

S*    0.0.0.0/0 [0/0] via 192.168.254.5
      192.168.254.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.254.0/24 is directly connected, Vlan254
L        192.168.254.1/32 is directly connected, Vlan254
spine-sw1#
```

VTPはtransparentモードにしていますので、vlan 10と254を手動で作成しています。

```text
spine-sw1#show vlan id 10

VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
10   VLAN0010                         active    Gi1/0/1, Gi1/0/2

VLAN Type  SAID       MTU   Parent RingNo BridgeNo Stp  BrdgMode Trans1 Trans2
---- ----- ---------- ----- ------ ------ -------- ---- -------- ------ ------
10   enet  100010     1500  -      -      -        -    -        0      0

Primary Secondary Type              Ports
------- --------- ----------------- ------------------------------------------

spine-sw1#
spine-sw1#show vlan id 254

VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
254  VLAN0254                         active    Gi1/0/1, Gi1/0/2

VLAN Type  SAID       MTU   Parent RingNo BridgeNo Stp  BrdgMode Trans1 Trans2
---- ----- ---------- ----- ------ ------ -------- ---- -------- ------ ------
254  enet  100254     1500  -      -      -        -    -        0      0

Primary Secondary Type              Ports
------- --------- ----------------- ------------------------------------------
```

デバイストラッキング関連のデフォルト設定は以下の通りです。
ポリシーデバイストラッキングは動作していません。
初期状態では、"default" という名前のポリシーも存在していません。
SISF機能を有効にしたときに作成されます。

```text
spine-sw1#show run all | inc device-tracking
no device-tracking logging packet drop
no device-tracking logging theft
no device-tracking logging resolution-veto
no device-tracking tracking
device-tracking binding reachable-lifetime 300 stale-lifetime 86400 down-lifetime 86400
no device-tracking binding logging
no device-tracking binding max-entries
```

<br>

## leaf-sw3

アップリンク向けのデバイストラッキングポリシーを作ります。
このポリシーをアタッチするのは、この装置のアップリンクのポートだけです。

IPv6での情報収集は無効にしています。

```text
!
device-tracking policy sisf_uplink_policy
 trusted-port
 security-level inspect
 device-role switch
 no protocol ndp
 no protocol dhcp6
 no protocol udp
!
```

<br>

> [!WARNING]
> show running-configで表示される `no protocol udp` は、サポートされていないコマンドですので、そのまま流し込むと警告メッセージがでます。

<br>

アクセスポート向けのポリシーを作ります。
このポリシーは端末が接続するポートにアタッチされます。

IPv6での情報収集を停止します。

デフォルトの動作レベルは`guard`になっていますので、誤検知によるパケット破棄を避けるために`inspect`にします。

```text
!
device-tracking policy sisf_access_policy
 security-level inspect
 no protocol ndp
 no protocol dhcp6
 no protocol udp
!
```

装置からポーリングするときの送信元IPの設定をします。

```
device-tracking tracking auto-source
```

ポリシーを作っただけでは何も動作しませんので、インタフェースにアタッチします。

まずアップリンク側から。

```text
conf t
interface range GigabitEthernet 1/0/1-2
device-tracking attach-policy sisf_uplink_policy
end
```

続いてダウンリンク側。

```text
conf t
interface range GigabitEthernet 1/0/1-8
device-tracking attach-policy sisf_access_policy
end
```

<br>

> [!NOTE]
> Catalyst 9000vは初期状態で4つのGigインタフェースを持ちますが、4個追加しています。

<br>

`show device-tracking policies` コマンドでポリシーのアタッチ状態を確認します。

```
leaf-sw3#show device-tracking policies
Target               Type  Policy               Feature        Target range
Gi1/0/1              PORT  sisf_uplink_policy   Device-tracking vlan all
Gi1/0/2              PORT  sisf_uplink_policy   Device-tracking vlan all
Gi1/0/3              PORT  sisf_access_policy   Device-tracking vlan all
Gi1/0/4              PORT  sisf_access_policy   Device-tracking vlan all
Gi1/0/5              PORT  sisf_access_policy   Device-tracking vlan all
Gi1/0/6              PORT  sisf_access_policy   Device-tracking vlan all
Gi1/0/7              PORT  sisf_access_policy   Device-tracking vlan all
Gi1/0/8              PORT  sisf_access_policy   Device-tracking vlan all
```

`show device-tracking database` で収集したバインディングテーブルを確認します。

```text
leaf-sw3#show device-tracking database

```

最初は何もありません。

自装置の配下にいるhost-6001とhost-6002の間でpingしてみます。

```text
host6001#ping 192.168.10.102
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.10.102, timeout is 2 seconds:
.!!!!
Success rate is 80 percent (4/5), round-trip min/avg/max = 1/1/2 ms
```

もう一度、バインディングテーブルを表示します。

```text
leaf-sw3#show device-tracking database
Binding Table has 1 entries, 1 dynamic (limit 200000)
Codes: L - Local, S - Static, ND - Neighbor Discovery, ARP - Address Resolution Protocol, DH4 - IPv4 DHCP, DH6 - IPv6 DHCP, PKT - Other Packet, API - API created
Preflevel flags (prlvl):
0001:MAC and LLA match     0002:Orig trunk            0004:Orig access
0008:Orig trusted trunk    0010:Orig trusted access   0020:DHCP assigned
0040:Cga authenticated     0080:Cert authenticated    0100:Statically assigned


    Network Layer Address                    Link Layer Address     Interface  vlan       prlvl      age        state      Time left
ARP 192.168.10.101                           5254.0002.7482         Gi1/0/3    10         0005       9s         REACHABLE  299 s
```

sw3の配下にいる端末**だけ**がバインディングテーブルに入りました。pingの宛先192.168.10.102はバインディングされませんでした。

今度はhost-6001からhost-6003にpingしてみます。このときの経路は host-6001 → sw3 → sw1(もしくはsw2) → sw4 → host-6002 です。

```text
host6001#ping 192.168.10.103
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.10.103, timeout is 2 seconds:
.!!!!
Success rate is 80 percent (4/5), round-trip min/avg/max = 47/53/59 ms
```

バインディングテーブルを確認してみます。

```
leaf-sw3#show device-tracking database
Binding Table has 1 entries, 1 dynamic (limit 200000)
Codes: L - Local, S - Static, ND - Neighbor Discovery, ARP - Address Resolution Protocol, DH4 - IPv4 DHCP, DH6 - IPv6 DHCP, PKT - Other Packet, API - API created
Preflevel flags (prlvl):
0001:MAC and LLA match     0002:Orig trunk            0004:Orig access
0008:Orig trusted trunk    0010:Orig trusted access   0020:DHCP assigned
0040:Cga authenticated     0080:Cert authenticated    0100:Statically assigned


    Network Layer Address                    Link Layer Address     Interface  vlan       prlvl      age        state      Time left
ARP 192.168.10.101                           5254.0002.7482         Gi1/0/3    10         0005       105s       REACHABLE  197 s
```

やはり宛先端末の情報はエントリに加わりませんね。

同様にsw4で確認してみます。

```
leaf-sw4#show device-tracking database
Binding Table has 1 entries, 1 dynamic (limit 200000)
Codes: L - Local, S - Static, ND - Neighbor Discovery, ARP - Address Resolution Protocol, DH4 - IPv4 DHCP, DH6 - IPv6 DHCP, PKT - Other Packet, API - API created
Preflevel flags (prlvl):
0001:MAC and LLA match     0002:Orig trunk            0004:Orig access
0008:Orig trusted trunk    0010:Orig trusted access   0020:DHCP assigned
0040:Cga authenticated     0080:Cert authenticated    0100:Statically assigned


    Network Layer Address                    Link Layer Address     Interface  vlan       prlvl      age        state      Time left
ARP 192.168.10.103                           5254.0003.c787         Gi1/0/3    10         0005       114s       REACHABLE  189 s
```

host-6003のアドレス 192.168.10.103 がバインディングテーブルにエントリされました。

<br>

## ここまでのまとめ

device-trackingをアクセススイッチに設定することで、自装置の配下にいる端末のMACアドレス、IPアドレス、接続ポートの情報がデータベース化されます。

<br>

## IPアドレスを重複させてみると何が起きる？

デバイストラッキングで発生するイベントをログに出してみます。アドレス重複を検知したいので `theft` を有効にします。

```text
leaf-sw3(config)#device-tracking logging ?
  packet           Packet events
  resolution-veto  Resolution veto events
  theft            IP or MAC theft events
  <cr>             <cr>

leaf-sw3(config)#device-tracking logging theft
leaf-sw3(config)#
```

host-6001のIPアドレスを変更して、わざとhost-6002と同じIPアドレスにしてみます。

```
host6001#conf t
host6001(config)#int gig 0/0
host6001(config-if)#ip address 192.168.10.102 255.255.255.0
*Jun 29 08:36:31.299: %IP-4-DUPADDR: Duplicate address 192.168.10.102 on GigabitEthernet0/0, sourced by 5254.001d.836d
```

host-6001の実態はIOSvです。
さすが、賢いですね。アドレスが重複したことをコンソールに出してくれました。

<br>

> [!NOTE]
> IOSvがアドレス重複を検出する仕組みはGratuitous ARPです。

<br>

また、接続しているsw3のコンソールにこのようなログが表示されました。

```text
leaf-sw3#
Jun 29 17:58:14.868 JST: %SISF-4-IP_THEFT: IP Theft IP=192.168.10.102 VLAN=10 MAC=5254.001d.836d IF=Gi1/0/3 New(Spoof) MAC=5254.0002.7482 New I/F=Gi1/0/3
Jun 29 17:58:20.181 JST: %SISF-4-IP_THEFT: IP Theft IP=192.168.10.102 VLAN=10 MAC=5254.001d.836d IF=Gi1/0/3 New(Spoof) MAC=5254.0002.7482 New I/F=Gi1/0/3
Jun 29 17:58:26.211 JST: %SISF-4-IP_THEFT: IP Theft IP=192.168.10.102 VLAN=10 MAC=5254.001d.836d IF=Gi1/0/3 New(Spoof) MAC=5254.0002.7482 New I/F=Gi1/0/3
```

もちろん、SYSLOGサーバにも同じメッセージが飛んでいます。

```text
Jun 29 17:58:14 192.168.254.3 101: Jun 29 17:58:14.868 JST: %SISF-4-IP_THEFT: IP Theft IP=192.168.10.102 VLAN=10 MAC=5254.001d.836d IF=Gi1/0/3 New(Spoof) MAC=5254.0002.7482 New I/F=Gi1/0/3
Jun 29 17:58:19 192.168.254.3 102: Jun 29 17:58:20.181 JST: %SISF-4-IP_THEFT: IP Theft IP=192.168.10.102 VLAN=10 MAC=5254.001d.836d IF=Gi1/0/3 New(Spoof) MAC=5254.0002.7482 New I/F=Gi1/0/3
Jun 29 17:58:25 192.168.254.3 103: Jun 29 17:58:26.211 JST: %SISF-4-IP_THEFT: IP Theft IP=192.168.10.102 VLAN=10 MAC=5254.001d.836d IF=Gi1/0/3 New(Spoof) MAC=5254.0002.7482 New I/F=Gi1/0/3
Jun 29 18:00:02 192.168.254.3 104: Jun 29 18:00:02.621 JST: %SISF-4-IP_THEFT: IP Theft IP=192.168.10.102 VLAN=10 MAC=5254.001d.836d IF=Gi1/0/3 New(Spoof) MAC=5254.0002.7482 New I/F=Gi1/0/3
```

IPアドレスが重複した場合、どっちが通信できるのかを確認してみます。

重複したアドレスを持つ host-6001 と host-6002 で同時にpingを打ってみます。

```text
host6001#
ping 192.168.254.5 repeat 100
Type escape sequence to abort.
Sending 100, 100-byte ICMP Echos to 192.168.254.5, timeout is 2 seconds:
!!!!!!!!.!!!!!!!!!!!!.!!!!!!!!!!!!.!!!!!!!!!!!!.!!!!!!!!!!!!!.!!!!!!!!
!!!!.!!!!!!!!!!!!!.!!!!!!!!!!!
Success rate is 93 percent (93/100), round-trip min/avg/max = 55/79/106 ms
host6001#
```

```text
host6002#ping 192.168.254.5 repeat 100
Type escape sequence to abort.
Sending 100, 100-byte ICMP Echos to 192.168.254.5, timeout is 2 seconds:
!!.!!!!!!!!!!!!!.!!!!!!!!!!!!.!!!!!!!!!!!!!.!!!!!!!!!!!!.!!!!!!!!!!!!.
!!!!!!!!!!!!!.!!!!!!!!!!!!.!!!
```

おもしろい結果になりました。

どちらかが通信できない期間（pingの結果が . の期間）はもう一方が通信している状態です。

どちらの端末も、通信できないわけではないけど、とてつもなく遅い、という状態になるでしょう。

<br>

## IPアドレス重複の影響を軽減してみる

IPアドレスが重複した場合、恐らく一方は被害者です。片方だけでも救済したいものです。

デバイストラッキングのポリシー設定で、セキュリティレベルを `inspect` としていた部分を `guard` に変えてみます。

```text
leaf-sw3(config)#device-tracking policy sisf_access_policy
leaf-sw3(config-device-tracking)#security-level guard
leaf-sw3(config-device-tracking)#end
```

もう一度、host-6001のIPアドレスを変更して host-6002 のアドレスと重複させてみます。

```text
host6002#ping 192.168.254.5 repeat 100
Type escape sequence to abort.
Sending 100, 100-byte ICMP Echos to 192.168.254.5, timeout is 2 seconds:
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
Success rate is 100 percent (100/100), round-trip min/avg/max = 63/79/125 ms
```

もともとそのアドレスを使っていた側は通信を継続でき、アドレスを重複させた側は通信できない、という結果になりました。

セキュリティレベルは `guard` にしておいた方が良さそうです。

<br>

## スイッチまたぎでのアドレス重複が起こるとどうなる？

sw3とsw4のセキュリティレベルは `guard` にしておきます。

この状態でhost-6001のアドレスを、sw4配下にいるhost-6003のアドレス 192.168.10.103 に変更し、意図的にアドレスを重複させます。

```text
host6003#ping 192.168.254.5 repeat 100
Type escape sequence to abort.
Sending 100, 100-byte ICMP Echos to 192.168.254.5, timeout is 2 seconds:
!!!!!!!!!!!!!!!!!!!!!!!!!!!!
*Jun 29 09:19:02.434: %IP-4-DUPADDR: Duplicate address 192.168.10.103 on GigabitEthernet0/0, sourced by 5254.0002.7482..........................................
..........
```

あら、残念・・・

今度はアドレス重複された側の通信が停止してしまいました。
どっちが正しいアドレスの所有者か、までは分かりませんので、これは致し方ないと思います。
一方が救済されるだけでも良し、とすべきでしょう。

ですが、もっと大きな問題が。

sw3とsw4はどちらもIPアドレスの重複を検知できていません。

<br>

## スイッチまたぎのアドレス重複を検知するには？

これはアップリンク側のポリシー設定を見直す必要がありそうです。

```text
device-tracking policy sisf_uplink_policy
 trusted-port !★ここが怪しい
 device-role switch
 no protocol ndp
 no protocol dhcp6
```

**trusted-port** という設定にするとガード動作が行われなくなります。

この設定を外してみます。

sw3とsw4の設定をこのようにしてみます。

```text
!
device-tracking policy sisf_access_policy
 no protocol ndp
 no protocol dhcp6
!
device-tracking policy sisf_uplink_policy
 security-level inspect
 device-role switch
 no protocol ndp
 no protocol dhcp6
!
```

そして、先程と同じようにhost-6001のIPアドレスをhost-6003と同じものに変更してみます。

すると、SYSLOGサーバに、sw3とsw4からほぼ同時に `%SISF-4-IP_THEFT` のログが表示されました。

```text
Jun 29 18:32:24 192.168.254.4 100: Jun 29 18:32:24.802 JST: %SISF-4-IP_THEFT: IP Theft IP=192.168.10.103 VLAN=10 MAC=5254.0003.c787 IF=Gi1/0/3 New(Spoof) MAC=5254.0002.7482 New I/F=Gi1/0/1
Jun 29 18:32:24 192.168.254.3 178: Jun 29 18:32:24.853 JST: %SISF-4-IP_THEFT: IP Theft IP=192.168.10.103 VLAN=10 MAC=5254.0002.7482 New(Spoof) MAC=5254.0003.c787 IF=Gi1/0/3 New IF=Gi1/0/1
```

どちらか一方のホストが通信を継続し、どちらか一方がブロックされる、という結果になりました。

アクセス側のポリシーでセキュリティレベルをguardにしていているのが有効に働いています。

ただし、アップリンク側にいる端末の情報もバインディングテーブルにエントリされるようになります。
VLAN上に多数の端末がいる場合、テーブルが消費するメモリが気になるかもしれません。

```
leaf-sw3#show device-tracking database
Binding Table has 4 entries, 4 dynamic (limit 200000)
Codes: L - Local, S - Static, ND - Neighbor Discovery, ARP - Address Resolution Protocol, DH4 - IPv4 DHCP, DH6 - IPv6 DHCP, PKT - Other Packet, API - API created
Preflevel flags (prlvl):
0001:MAC and LLA match     0002:Orig trunk            0004:Orig access
0008:Orig trusted trunk    0010:Orig trusted access   0020:DHCP assigned
0040:Cga authenticated     0080:Cert authenticated    0100:Statically assigned


    Network Layer Address                    Link Layer Address     Interface  vlan       prlvl      age        state      Time left
ARP 192.168.10.103                           5254.0002.7482         Gi1/0/3    10         0005       4mn        REACHABLE  47 s try 0
ARP 192.168.10.102                           5254.001d.836d         Gi1/0/3    10         0005       55s        REACHABLE  255 s try 0
ARP 192.168.10.101                           5254.0002.7482         Gi1/0/3    10         0005       53mn       STALE     try 0 85251 s
ARP 192.168.10.5                             5254.0003.6a0d         Gi1/0/2    10         0003       49mn       STALE     try 0 87141 s
```

<br>

## トラッキングするとどうなる？

デフォルトではトラッキング動作は無効になっています。

sw3とsw4のアクセス側のポリシーをこのようにしてみます。

```text
!
device-tracking policy sisf_access_policy
 no protocol ndp
 no protocol dhcp6
 no protocol udp
 tracking enable
!
```

host-6001のIPアドレスをhost-6003と重複させます。

すると、sw3とsw4は次のようなログをしばらくの間、吐き続けます。

```text
Jun 29 19:31:53.760 JST: %SISF-4-IP_THEFT: IP Theft IP=192.168.10.103 VLAN=10 MAC=5254.0002.7482 New(Spoof) MAC=5254.0003.c787 IF=Gi1/0/3 New IF=Gi1/0/1
Jun 29 19:31:55.695 JST: %SISF-4-IP_THEFT: IP Theft IP=192.168.10.103 VLAN=10 MAC=5254.0002.7482 New(Spoof) MAC=5254.0003.c787 IF=Gi1/0/3 New IF=Gi1/0/1
Jun 29 19:31:56.719 JST: %SISF-4-IP_THEFT: IP Theft IP=192.168.10.103 VLAN=10 MAC=5254.0002.7482 New(Spoof) MAC=5254.0003.c787 IF=Gi1/0/3 New IF=Gi1/0/1
Jun 29 19:31:57.739 JST: %SISF-4-IP_THEFT: IP Theft IP=192.168.10.103 VLAN=10 MAC=5254.0002.7482 New(Spoof) MAC=5254.0003.c787 IF=Gi1/0/3 New IF=Gi1/0/1
Jun 29 19:31:59.757 JST: %SISF-4-IP_THEFT: IP Theft IP=192.168.10.103 VLAN=10 MAC=5254.0002.7482 New(Spoof) MAC=5254.0003.c787 IF=Gi1/0/3 New IF=Gi1/0/1
Jun 29 19:32:01.760 JST: %SISF-4-IP_THEFT: IP Theft IP=192.168.10.103 VLAN=10 MAC=5254.0002.7482 New(Spoof) MAC=5254.0003.c787 IF=Gi1/0/3 New IF=Gi1/0/1
Jun 29 19:32:02.772 JST: %SISF-4-IP_THEFT: IP Theft IP=192.168.10.103 VLAN=10 MAC=5254.0002.7482 New(Spoof) MAC=5254.0003.c787 IF=Gi1/0/3 New IF=Gi1/0/1
Jun 29 19:32:03.795 JST: %SISF-4-IP_THEFT: IP Theft IP=192.168.10.103 VLAN=10 MAC=5254.0002.7482 New(Spoof) MAC=5254.0003.c787 IF=Gi1/0/3 New IF=Gi1/0/1
Jun 29 19:32:05.295 JST: %SISF-4-IP_THEFT: IP Theft IP=192.168.10.103 VLAN=10 MAC=5254.0002.7482 New(Spoof) MAC=5254.0003.c787 IF=Gi1/0/3 New IF=Gi1/0/1
Jun 29 19:32:06.805 JST: %SISF-4-IP_THEFT: IP Theft IP=192.168.10.103 VLAN=10 MAC=5254.0002.7482 New(Spoof) MAC=5254.0003.c787 IF=Gi1/0/3 New IF=Gi1/0/1
Jun 29 19:32:08.819 JST: %SISF-4-IP_THEFT: IP Theft IP=192.168.10.103 VLAN=10 MAC=5254.0002.7482 New(Spoof) MAC=5254.0003.c787 IF=Gi1/0/3 New IF=Gi1/0/1
Jun 29 19:32:09.833 JST: %SISF-4-IP_THEFT: IP Theft IP=192.168.10.103 VLAN=10 MAC=5254.0002.7482 New(Spoof) MAC=5254.0003.c787 IF=Gi1/0/3 New IF=Gi1/0/1

```

また、guard機能もうまく働いていないようです。

```
host6001#ping 192.168.254.5 repeat 100
Type escape sequence to abort.
Sending 100, 100-byte ICMP Echos to 192.168.254.5, timeout is 2 seconds:
!!!!!!!!!!!!!!!.!!!!!!!!!!!!.!!!!!!!!!!!.!!!!!!!!!!!!.!!!!!!!!!!!...!.
!!!!!!!!!!!!
```

仕組みは分かりませんが、トラッキングはデフォルトのまま無効にしておく方が良さそうです。

<br>

## 端末が移動するとどうなる？

CMLの操作でhost-6001をsw3の配下からsw4の配下に移動します。

結果的には、特に何も起きません。ログもでません。

sw3のバインディングテーブルはこうなります。

```text
leaf-sw3#show device-tracking database

    Network Layer Address                    Link Layer Address     Interface  vlan       prlvl      age        state      Time left
ARP 192.168.10.101                           5254.0002.7482         Gi1/0/1    10         0003       5s         REACHABLE  303 s try 0
```

接続先がGi1/0/1、つまりアップリンクの先にいる、というように変わりました。

<br>

## いらないVLANの情報を排除するには？

ここまでの設定では、sw3とsw4はアップリンク側から流れてくるパケットの情報も使ってバインディングテーブルを構築します。

したがって、自装置に直接つながっている端末だけでなく、一見無関係なスイッチの情報まで見えてしまいます。

```text
leaf-sw3#show device-tracking database
Binding Table has 7 entries, 7 dynamic (limit 200000)
Codes: L - Local, S - Static, ND - Neighbor Discovery, ARP - Address Resolution Protocol, DH4 - IPv4 DHCP, DH6 - IPv6 DHCP, PKT - Other Packet, API - API created
Preflevel flags (prlvl):
0001:MAC and LLA match     0002:Orig trunk            0004:Orig access
0008:Orig trusted trunk    0010:Orig trusted access   0020:DHCP assigned
0040:Cga authenticated     0080:Cert authenticated    0100:Statically assigned


    Network Layer Address                    Link Layer Address     Interface  vlan       prlvl      age        state      Time left
ARP 192.168.254.5                            5254.000c.3e3a         Gi1/0/2    254        0003       19mn       STALE     try 0 87363 s
ARP 192.168.254.4                            5254.000b.ebfc         Gi1/0/1    254        0003       19mn       STALE     try 0 87421 s
ARP 192.168.254.2                            5254.0019.21ee         Gi1/0/2    254        0003       19mn       STALE     try 0 88137 s
ARP 192.168.254.1                            5254.0018.a2a3         Gi1/0/1    254        0003       19mn       STALE     try 0 89873 s
ARP 192.168.10.102                           5254.001d.836d         Gi1/0/3    10         0005       13mn       STALE     try 0 87319 s
ARP 192.168.10.101                           5254.0002.7482         Gi1/0/3    10         0005       3mn        REACHABLE  69 s try 0
ARP 192.168.10.5                             5254.0003.6a0d         Gi1/0/2    10         0003       18mn       STALE     try 0 85843 s
```

このように192.168.254.xのアドレスはスイッチの管理用のアドレスですので、バインディングテーブルに入ってほしくないエントリかもしれません。

「デバイストラッキングをしない」というポリシーは作れませんので、「情報収集しない」というポリシを作ります。

```text
!
device-tracking policy sisf_vlan254_policy
 trusted-port
 device-role switch
 no protocol ndp
 no protocol dhcp6
 no protocol arp
 no protocol dhcp4
 no protocol udp
!
```

このポリシーをVLAN 254にアタッチしてみます。

```text
!
vlan configuration 254
 device-tracking attach-policy sisf_vlan254_policy
!
```

すると・・・

バインディングテーブルはこのようになります。

```text
leaf-sw3#show device-tracking database
Binding Table has 4 entries, 3 dynamic (limit 200000)
Codes: L - Local, S - Static, ND - Neighbor Discovery, ARP - Address Resolution Protocol, DH4 - IPv4 DHCP, DH6 - IPv6 DHCP, PKT - Other Packet, API - API created
Preflevel flags (prlvl):
0001:MAC and LLA match     0002:Orig trunk            0004:Orig access
0008:Orig trusted trunk    0010:Orig trusted access   0020:DHCP assigned
0040:Cga authenticated     0080:Cert authenticated    0100:Statically assigned


    Network Layer Address                    Link Layer Address     Interface  vlan       prlvl      age        state      Time left
ARP 192.168.254.4                            5254.000b.ebfc         Gi1/0/1    254        0003       3mn        REACHABLE  125 s
L   192.168.254.3                            5254.0014.5bdc         Vl254      254        0100       5mn        REACHABLE
ARP 192.168.254.2                            5254.0019.21ee         Gi1/0/2    254        0003       3mn        REACHABLE  108 s
ARP 192.168.254.1                            5254.0018.a2a3         Gi1/0/1    254        0003       3mn        REACHABLE  117 s
```

これではダメですね。VLAN 254上の端末もエントリされています。

アップリンクポートに設定されたポリシーが優先されてしまっています。特定のVLANではバインディングテーブルを作らない、というのはできなそうです。

<br>

## 大量のIPアドレス、MACアドレスが存在したら？

バインディングテーブルがあふれるかもしれません。

バインディングテーブルの最大サイズはマニュアルに記載がありませんが、実機があれば確認できます。

```text
leaf-sw4(config)#device-tracking binding max-entries ?
  <1-1000000>  Number of entries
```

設定できるのは100万(1000000)エントリまでですが、実際に設定すると警告が表示されます。

```
leaf-sw4(config)# device-tracking binding max-entries 1000000

% Warning: max-entries is greater than default (200000), Please check memory availability
```

デフォルトで **バインディングテーブルの上限は20万エントリに設定されている** ということが分かります。

一つのポートあたりのMACアドレス、IPアドレスに制限をつけることもできますが、デフォルトではリミットは設定されていません。
これはそのままでよいでしょう。

では、sw3とsw4のポリシーで上限を3個に設定してみます。

```text
leaf-sw3(config)#device-tracking binding max-entries 3
```

これでエントリ数の上限は3個になりました。

すると、端末同士のpingで疎通できない相手がでてきます。

このケースではhost-6002からhost-6004がダメになっていますが、どの区間が通信不可になるかは動的に変わります。

```text
host6002#ping 192.168.10.101
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.10.101, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/2 ms

host6002#ping 192.168.10.102
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.10.102, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/1 ms

host6002#ping 192.168.10.103
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.10.103, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 99/109/116 ms

host6002#ping 192.168.10.104
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.10.104, timeout is 2 seconds:
.....
Success rate is 0 percent (0/5)
```

**バインディングテーブルにエントリがないと通信できない** という結果になりました。

デフォルトの上限20万エントリという数字は十分大きく設定されています。SDMテンプレートを考えれば心配する必要はないと思います。

<br>

## バインディングテーブルを外部に保存するには？

IOS-XEはコマンド出力をリダイレクトできますので、外部のTFTPサーバや、フラッシュメモリに出力を保存できます。

たとえば、このようにリダイレクトすればブートフラッシュ直下に `dt_database.txt` というファイル名で記録されます。
これはテキストファイルですので more コマンドで中身を閲覧できます。

```text
show device-tracking database | redirect bootflash:dt_database.txt
```

> [!TIP]
> 外部のTFTPサーバにリダイレクトすることも可能ですが、TFTPサーバ側で事前にファイルを準備し、書き込みできるようにしておく必要があります。

<br>

## 定期的に実行するには？

EEM(Embedded Event Manager)が動く装置であれば定期的に実行することもできます。

> [!WARNING]
>
> Catalyst 9200ではEEMは動かないかも？？？（要確認）
>
> Catalyst 9300であれば、DNA EssentilsライセンスでEEM4.0が動きます（Feature Navigatorで確認）

たとえば、このようなEEMアプレットを作れば、5分ごとに定期的にコマンドが実行されてブートフラッシュのファイルが更新されます。

```text
event manager applet EXPORT-DEVICE-TRACKING-DATABASE
 event timer cron cron-entry "*/5 * * * *"
 action 1.0 cli command "enable"
 action 2.0 cli command "show device-tracking database | redirect bootflash:dt_database.txt"
```

cron-entryはUNIXのcronと同じ指定です。

> [!TIP]
> 分 時 日 月 曜日
> *  *  *  *  *

**毎分** は `* * * * *` です。

**毎時0分** は `0 * * * *` です。0分は一日に24回来ますので、毎時実行されます。

**毎日 2時0分** は `0 2 * * *` です。2時0分は一日に1回しか来ませんので、一日１回実行されます。

**10分ごと** は `*/10 * * * *` です。

デバイストラッキングで作成するバインディングテーブルのエントリは少なくとも一日ありますので、毎日指定の時刻に実行すればよいと思います。

<br>

# まとめ

SISFベースのデバイストラッキングを使うと、IPアドレスの重複を検出できます。

適用すべき場所は、端末を継続する末端のスイッチです。

末端のスイッチには上位スイッチと接続するアップリンクとなるポートと、端末が接続されるアクセスポートがありますので、デバイストラッキングの動作ポリシーも分ける必要があります。

アップリンクのポリシーはこのようにします。

端末側から来るトラフィックで防御をかければよいので、アップリンク側のセキュリティレベルは `inspect` に変更しています。
接続相手は上位のスイッチですので、device-roleを`switch`に変えています。

```text
!
device-tracking policy sisf_uplink_policy
 security-level inspect
 device-role switch
 no protocol ndp
 no protocol dhcp6
 no protocol udp !★ゴミ
!
```

アクセスポートのポリシーはこのようにします。

```text
!
device-tracking policy sisf_access_policy
 no protocol ndp
 no protocol dhcp6
 no protocol udp !★ゴミ
!
```

設定上は見えていませんが、このポリシーのセキュリティレベルは `guard` です。

こうすることで、IPアドレスの重複が起こったとき、どちらか一方は通信を継続できます。
アドレス重複した端末が共にスローダウンする、という事象よりはずっとよいと思います。
また、guardしないと状態が安定しないので %SISF-4-IP_THEFT のログが出続けることになります。

IPv6を使った検知は、本当にIPv6を使っている環境でない限り停止しておいたほうがよいです。
無用なログメッセージがでてきます。

デバイストラッキングで何か異常な事象を発見してもデフォルトでは何も表示されません。
下記を設定しておくことと、IPアドレスの重複をログとして出力できます。

```
device-tracking logging theft
```
