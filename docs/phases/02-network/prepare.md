# Phase 2 事前学習ガイド - コンテナネットワーク

このドキュメントは、Phase 2でコンテナネットワークを実装するために必要な基礎知識をまとめたものです。

## 1. Linux ネットワークスタック基礎

### 1.1 ネットワーク名前空間

**Network Namespaceの概念**
- 独立したネットワークスタックの提供
- インターフェース、ルーティングテーブル、iptablesルールの分離
- プロセスごとに異なるネットワークビューを実現

**作成と操作**
```bash
# 新しいネットワーク名前空間の作成
ip netns add container1

# 名前空間内でコマンド実行
ip netns exec container1 ip addr show

# 名前空間の削除
ip netns delete container1
```

### 1.2 仮想ネットワークデバイス

**veth (Virtual Ethernet) ペア**
```
┌─────────────┐      ┌─────────────┐
│   Host NS   │      │Container NS │
│             │      │             │
│  veth0 ─────┼──────┼─── veth1    │
│             │      │             │
└─────────────┘      └─────────────┘
```

**特徴**
- 仮想的なイーサネットケーブルのように動作
- 一方に入力されたパケットは他方から出力
- 名前空間を跨いだ通信に使用

## 2. Linux Bridge

### 2.1 ブリッジの概念

**Linux Bridgeとは**
- ソフトウェアによるL2スイッチ実装
- 複数のネットワークインターフェースを接続
- MACアドレステーブルによる効率的な転送

**ブリッジの構造**
```
        ┌─────────────────┐
        │  Linux Bridge   │
        │   (ctr0)        │
        └────┬──┬──┬──────┘
             │  │  │
         veth0 veth2 veth4  ← ホスト側
             │  │  │
         veth1 veth3 veth5  ← コンテナ側
             │  │  │
        ┌────┴──┴──┴────┐
        │  Containers    │
        └────────────────┘
```

### 2.2 ブリッジ操作コマンド

```bash
# ブリッジの作成
ip link add name ctr0 type bridge

# ブリッジの起動
ip link set ctr0 up

# IPアドレスの割り当て
ip addr add 172.20.0.1/24 dev ctr0

# インターフェースをブリッジに接続
ip link set veth0 master ctr0

# ブリッジの状態確認
bridge link show
bridge fdb show
```

## 3. ネットワークアドレス変換 (NAT)

### 3.1 NAT の種類と用途

**SNAT (Source NAT)**
- 送信元アドレスの変換
- 内部ネットワークから外部への通信で使用

**DNAT (Destination NAT)**
- 宛先アドレスの変換
- 外部から内部サービスへのアクセスで使用

**MASQUERADE**
- 動的なSNATの一種
- 送信インターフェースのIPアドレスを自動的に使用

### 3.2 iptables によるNAT設定

**基本的なNATテーブル構造**
```
                  ┌─────────────┐
                  │ PREROUTING  │ ← パケット受信時
                  └──────┬──────┘
                         │
                  ┌──────┴──────┐
                  │   ROUTING   │ ← ルーティング決定
                  └──┬──────┬───┘
                     │      │
              ┌──────┴──┐ ┌─┴────────┐
              │ INPUT   │ │ FORWARD  │
              └─────────┘ └─────┬────┘
                               │
                        ┌──────┴──────┐
                        │POSTROUTING  │ ← パケット送信時
                        └─────────────┘
```

**MASQUERADEの設定例**
```bash
# コンテナからの外部通信を許可
iptables -t nat -A POSTROUTING -s 172.20.0.0/24 ! -o ctr0 -j MASQUERADE

# パケット転送の有効化
echo 1 > /proc/sys/net/ipv4/ip_forward
```

## 4. ネットワーク設計パターン

### 4.1 コンテナネットワークモデル

**Bridge モード**
```
Internet ← eth0 ← Bridge ← veth ← Container
                    │
                    └─── veth ← Container
```

**Host モード**
- コンテナがホストのネットワーク名前空間を共有
- 最高のパフォーマンス、最低の分離性

**None モード**
- ネットワークインターフェースなし
- 完全に隔離されたコンテナ

### 4.2 IPアドレス管理

**IPAM (IP Address Management)**
- サブネットの管理
- IPアドレスの動的割り当て
- アドレスプールの管理

```go
// 簡単なIPAM実装例
type IPAM struct {
    subnet   *net.IPNet
    gateway  net.IP
    allocated map[string]net.IP
    mu       sync.Mutex
}

func (ipam *IPAM) Allocate(id string) (net.IP, error) {
    ipam.mu.Lock()
    defer ipam.mu.Unlock()
    
    // 次の利用可能なIPを検索
    for ip := ipam.subnet.IP.Mask(ipam.subnet.Mask); ipam.subnet.Contains(ip); inc(ip) {
        if !ipam.isAllocated(ip) {
            ipam.allocated[id] = ip
            return ip, nil
        }
    }
    return nil, errors.New("no available IP addresses")
}
```

## 5. netlink ライブラリ

### 5.1 netlinkとは

**概要**
- カーネルとユーザー空間の通信インターフェース
- ネットワーク設定の標準的な方法
- rtnetlink: ルーティングとリンク設定

**Go言語でのnetlink使用**
```go
import "github.com/vishvananda/netlink"

// リンクの作成
la := netlink.NewLinkAttrs()
la.Name = "veth0"
veth := &netlink.Veth{
    LinkAttrs: la,
    PeerName:  "veth1",
}
err := netlink.LinkAdd(veth)

// IPアドレスの設定
addr, _ := netlink.ParseAddr("172.20.0.2/24")
netlink.AddrAdd(link, addr)
```

### 5.2 一般的なnetlink操作

**インターフェース操作**
```go
// リンクの取得
link, err := netlink.LinkByName("eth0")

// リンクの状態変更
netlink.LinkSetUp(link)
netlink.LinkSetDown(link)

// 名前空間への移動
netlink.LinkSetNsPid(link, containerPid)
```

**ルーティング操作**
```go
// デフォルトルートの追加
route := &netlink.Route{
    LinkIndex: link.Attrs().Index,
    Gw:        net.ParseIP("172.20.0.1"),
}
netlink.RouteAdd(route)
```

## 6. トラブルシューティング

### 6.1 デバッグツール

**tcpdump**
```bash
# ブリッジインターフェースのパケットキャプチャ
tcpdump -i ctr0 -n

# 特定のコンテナ間通信の確認
tcpdump -i veth0 host 172.20.0.2
```

**ip コマンド**
```bash
# ネットワーク名前空間の確認
ip netns list

# ルーティングテーブルの確認
ip route show

# ARPテーブルの確認
ip neigh show
```

**iptables デバッグ**
```bash
# パケットカウンタの確認
iptables -t nat -L -v -n

# ルールのトレース
iptables -t raw -A PREROUTING -j TRACE
```

### 6.2 一般的な問題と解決策

| 問題 | 原因 | 解決策 |
|------|------|--------|
| コンテナから外部に接続できない | ip_forward無効 | `sysctl -w net.ipv4.ip_forward=1` |
| DNSが解決できない | resolv.conf未設定 | コンテナ内にresolv.confをマウント |
| vethが見つからない | 名前空間の誤り | 正しい名前空間で操作 |
| ブリッジが動作しない | STP有効 | `brctl stp ctr0 off` |

## 7. セキュリティ考慮事項

### 7.1 ネットワーク分離

**トラフィック制御**
```bash
# 特定のポートのみ許可
iptables -A FORWARD -i ctr0 -p tcp --dport 80 -j ACCEPT
iptables -A FORWARD -i ctr0 -j DROP
```

**MACアドレススプーフィング対策**
```bash
# ebtablesでMACアドレスを固定
ebtables -A FORWARD -i veth0 -s ! 02:42:ac:14:00:02 -j DROP
```

### 7.2 リソース制限

**帯域幅制限**
```bash
# tcでトラフィックシェーピング
tc qdisc add dev veth0 root tbf rate 1mbit burst 32kbit latency 400ms
```

## 8. 実装のベストプラクティス

### 8.1 エラーハンドリング

```go
// ネットワーク設定のロールバック
func setupNetwork(containerID string) error {
    // 設定を記録
    var cleanups []func()
    
    // veth作成
    if err := createVeth(containerID); err != nil {
        return err
    }
    cleanups = append(cleanups, func() { deleteVeth(containerID) })
    
    // エラー時のクリーンアップ
    defer func() {
        if err != nil {
            for i := len(cleanups) - 1; i >= 0; i-- {
                cleanups[i]()
            }
        }
    }()
    
    // 他の設定...
    
    return nil
}
```

### 8.2 並行性の考慮

```go
// IPアドレス割り当ての排他制御
type NetworkManager struct {
    mu       sync.Mutex
    networks map[string]*Network
    ipam     *IPAM
}

func (nm *NetworkManager) AllocateIP(networkID, containerID string) (net.IP, error) {
    nm.mu.Lock()
    defer nm.mu.Unlock()
    
    network, ok := nm.networks[networkID]
    if !ok {
        return nil, fmt.Errorf("network %s not found", networkID)
    }
    
    return network.ipam.Allocate(containerID)
}
```

## 9. 参考資料

### 9.1 必読ドキュメント
- [Linux Advanced Routing & Traffic Control](https://lartc.org/)
- [netfilter/iptables documentation](https://www.netfilter.org/documentation/)
- [iproute2 documentation](https://wiki.linuxfoundation.org/networking/iproute2)

### 9.2 参考実装
- [CNI (Container Network Interface)](https://github.com/containernetworking/cni)
- [Docker libnetwork](https://github.com/moby/libnetwork)
- [netlink library](https://github.com/vishvananda/netlink)

### 9.3 理解度確認チェックリスト

- [ ] vethペアの動作原理を説明できる
- [ ] Linux Bridgeの役割を理解している
- [ ] iptablesでNAT設定ができる
- [ ] netlinkを使用したネットワーク設定が実装できる
- [ ] ネットワーク名前空間の操作ができる
- [ ] IPアドレス管理の実装方法を理解している

これらの知識を身につけることで、Phase 2のコンテナネットワーク実装に効果的に取り組むことができます。