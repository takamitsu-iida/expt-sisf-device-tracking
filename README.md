# SISF Device Tracking

<!--
[!NOTE]
[!TIP]
[!IMPORTANT]
[!WARNING]
[!CAUTION]
-->

Cisco SISFベースのデバイストラッキングについて、動作検証の結果をまとめます。

<br>

## マニュアル

> [!note]
> Feature NavigatorでSISFを検索しても出てきません。なぜ？

- https://www.cisco.com/c/ja_jp/td/docs/switches/lan/catalyst9200/software/release/17-2/configuration_guide/sec/b_172_sec_9200_cg/configuring_sisf_based_device_tracking.html

- https://www.cisco.com/c/ja_jp/support/docs/switches/catalyst-9300-series-switches/221562-troubleshoot-sisf-on-catalyst-9000-serie.html

<br>

## SISF Device Trackingとは

Switch Integrated Security Features

エンドノードの存在、位置、移動を検出する仕組みです。
かつてIP Device Trackingという名前で古くから実装されていたものと、IPv6のセキュリティ機能を統合したものです。


> [!NOTE]
> 古いCatalystでは SISF は動きません。

> [!NOTE]
> 逆に新しいCatalystでは、IP Device Trackingの設定コマンド、IPv6 snoopingの設定コマンドは無くなっています。

スイッチは受信したトラフィックを分析し、デバイスを識別する情報としてMACアドレスとIP アドレスを抽出して、バインディングテーブルに保存します。

SISFが作るバインディングテーブルを必要とする機能もあり、それらをSISFクライアント、と呼びます。

- LISP/EVPN
- 802.1X
- Web認証
- Cisco TrustSec
- DHCP スヌーピング

これらが代表的なSISFクライアントです。

SISFクライアントを有効にすると、バインディングテーブルを作るために必要なSISFポリシーが自動で作成されます。

SISFはIPv4とIPv6の両方で動作します。

> [!NOTE]
> IPv6を使っていない環境では、SISFポリシーの設定を変更してIPv6でのトラッキングを停止した方がいいです。無用なログが表示されます。

SISFではDHCP、ARP、ND、RAなどのパケットを収集してテーブルを作成します。
これらパケットを送信しないサイレントノードがいる場合は、スタティックにエントリを設定することもできます。

SISFデバイストラッキングは、**デフォルトでは無効**になっています。

デバイストラッキングポリシーをアタッチすることでSISFを有効化します。

> [!NOTE]
> ポリシーを指定せずに有効化することもできますが、その場合は **default** という名前のポリシーが作られてアタッチされます。
> defaultポリシーは変更できません。

バインディングテーブルには、次のような情報が含まれます。

- IPv4/IPv6アドレス
- MACアドレス
- ホストが存在するインタフェース、VLAN
- 到達可能性情報

到達可能性はいくつかの状態があり、代表的なのは`Reachable`、`Stale`、`Down`などです。

- Reachableステート

端末から来た制御パケットがスイッチで正しく処理されていることが確認されている状態です。
このステートのライフタイムは **5分** です。

- Staleステート

Reachableステートのライフフタイムが切れ、パケットの送信が確認できていない状態です。
このステートのライフタイムは **24時間** です。
したがって端末の存在を確認してから24時間はテーブルにエントリが残ります。

- Down

ホストが繋がっているインタフェースがダウンしている状態です。
このステートのライフタイムは **24時間** です。

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

> [!NOTE]
> **destination-switch** という設定が？キーで表示されるコマンド一覧に表示されますが、実装されていないので設定しても何もおこりません。


**protocol**

protocolに続く設定はこれらです。デフォルトで全て有効になっています。

```text
  arp    Glean addresses in ARP packets
  dhcp4  Glean addresses in DHCPv4 packets
  dhcp6  Glean addresses in DHCPv6 packets
  ndp    Glean addresses in NDP packets
  udp    Gleaning from UDP packets
```

> [!NOTE]
> **protocol udp** という設定が？キーで表示されるコマンド一覧に表示されますが、実装されていないので設定しても何もおこりません。

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

場合によっては、ノンインテリスイッチでポートを増やしてから端末がつながることもありえますので、管理外スイッチを接続しています。

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

> [!TIPS]
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

> [!WARNING]
> show running-configで表示される `no protocol udp` は、サポートされていないコマンドですので、そのまま流し込むと警告メッセージがでます。

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

> [!NOTE]
> Catalyst 9000vは初期状態で4つのGigインタフェースを持ちますが、4個追加しています。

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


## ここまでのまとめ

device-trackingをアクセススイッチに設定することで、自装置の配下にいる端末のMACアドレス、IPアドレス、接続ポートの情報がデータベース化される。


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

> [!NOTE]
> IOSvがアドレス重複を検出する仕組みはGratuitous ARPです。

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

どちらの端末も通信できないわけではないけど、とてつもなく遅い、という状態になるでしょう。

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

## スイッチまたぎのアドレス重複を検知するには？

これはアップリンク側のポリシー設定を見直す必要がありそうです。

```text
device-tracking policy sisf_uplink_policy
 trusted-port !★ここが怪しい
 security-level guard !★表示されません
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

host-6001のアドレスをhost-6003と重複させます。

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

<br>

# 課題

バインディングテーブルは、端末の位置（接続ポート）と、アドレス（MACアドレスとIPアドレス）の情報を保持しています。
そのエントリ期間は24時間ありますので、一日１回、各スイッチから情報を吸い上げて、管理台帳との整合性を確認した方がよいのですが、効率的にエクスポートする手段がありません。

スイッチがIPアドレスの重複を検知した際に、それを通知する手段はSYSLOGでメッセージを飛ばす方法しかありません。
SNMPトラップはサポートされていません。

SYSLOGにはこのようなメッセージが飛んできますので、これをさらに次のアクションに繋げる必要があります。

`%SISF-4-IP_THEFT: IP Theft IP=192.168.10.103 VLAN=10 MAC=5254.0003.c787 IF=Gi1/0/3 New(Spoof) MAC=5254.0002.7482 New I/F=Gi1/0/1`
