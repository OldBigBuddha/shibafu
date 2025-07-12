# Phase 2: 基本的なコンテナネットワーク - 詳細学習計画

## 学習目標
コンテナに独立したネットワーク環境を提供し、Linux bridgeとveth pairを使用してホストおよび外部ネットワークとの通信を実現する

## 事前準備

### 環境要件確認
- Phase 1の完了（libcontainerによるコンテナ管理）
- ネットワーク管理ツール（ip、bridge、iptables）
- root権限（ネットワーク設定のため）
- Goの netlink ライブラリへのアクセス

### 事前学習課題
以下の概念について調べ、自分なりに整理せよ：
- Linux bridgeとは？スイッチとの違いは？
- veth pairの概念と仕組み
- Network namespaceでの通信経路
- NATとMASQUERADEの違い
- iptablesの基本的なテーブル構造

## Step 1: Linuxネットワーク基礎の実験

### このステップの目的
手動でLinuxネットワークコンポーネントを操作し、コンテナネットワークの基本原理を理解する

### 調査すべき内容
- ip linkコマンドの使用方法
- bridge-utilsの使用方法
- Network namespaceの作成と操作
- veth pairの作成と接続方法

### 実験課題
以下の手動操作を順番に実行し、各ステップでの結果を分析せよ：

```bash
# 1. Network namespaceの作成
sudo ip netns add ?

# 2. bridgeの作成
sudo ip link add ? type bridge

# 3. veth pairの作成
sudo ip link add ? type veth peer name ?

# 4. 片方をbridgeに接続
sudo ip link set ? master ?

# 5. もう片方をnamespaceに移動
sudo ip netns exec ? ip link set ? netns ?

# 6. IPアドレスの設定
sudo ip addr add ? dev ?
sudo ip netns exec ? ip addr add ? dev ?
```

### 検証課題
各ステップで以下を確認し、なぜそうなるかを考察せよ：
```bash
# 作成されたインターフェースの確認
ip link show
ip netns exec testns ip link show

# ルーティングテーブルの確認
ip route show
ip netns exec testns ip route show

# 接続性テスト
ping ?  # どのIPアドレスを指定すべきか？
```

### 考察課題
- veth pairの両端がどのような関係にあるか？
- bridgeにインターフェースを接続する意味は？
- namespace間での通信はどのような経路を辿るか？

## Step 2: Goによるnetlink操作の理解

### このステップの目的
Go言語のnetlinkライブラリを使用してネットワーク設定を自動化する

### 調査すべき内容
- github.com/vishvananda/netlinkライブラリ
- Go言語でのLinuxネットワーク操作
- エラーハンドリングのベストプラクティス

### 実装課題
以下の機能をGoコードで実装せよ：

```go
import "github.com/vishvananda/netlink"

// Bridgeの作成
func createBridge(name string) error {
    bridge := &netlink.Bridge{
        LinkAttrs: netlink.LinkAttrs{
            Name: name,
        },
    }
    
    // どのAPIを呼び出すべきか？
    return ?
}

// veth pairの作成
func createVethPair(hostVeth, containerVeth string) error {
    veth := &netlink.Veth{
        LinkAttrs: netlink.LinkAttrs{Name: hostVeth},
        PeerName:  containerVeth,
    }
    
    // どのAPIを呼び出すべきか？
    return ?
}

// インターフェースをnamespaceに移動
func moveToNamespace(interfaceName string, pid int) error {
    // どのような実装が必要か？
    return ?
}
```

### 設計課題
- エラーハンドリングはどうするか？
- 既存のbridge/vethがあった場合の処理は？
- cleanup処理はどうするか？

### テスト課題
```bash
# 実装したコードをテスト
go run network_setup.go create-bridge ctr0
go run network_setup.go create-veth host-veth container-veth
```

## Step 3: libcontainerとの統合

### このステップの目的
Phase 1で作成したlibcontainerベースのコンテナランタイムにネットワーク機能を統合する

### 実装課題
1. **Network namespace設定の追加**
   ```go
   config.Namespaces = append(config.Namespaces, configs.Namespace{
       Type: configs.NEWNET,
   })
   ```

2. **Hooksの活用**
   ```go
   config.Hooks = &configs.Hooks{
       Prestart: []configs.Hook{
           // ネットワーク設定のhookをどう実装するか？
       },
   }
   ```

3. **ネットワーク設定関数**
   ```go
   func setupContainerNetwork(containerPID int, containerID string) error {
       // bridgeの作成
       // veth pairの作成
       // containerVethをnamespaceに移動
       // IPアドレスの設定
       // routeの設定
       return nil
   }
   ```

### 設計課題
- コンテナ作成とネットワーク設定のタイミングは？
- IPアドレスの割り当てルールは？
- エラー時のcleanup戦略は？

### 考察課題
- なぜHookを使用するのか？
- 他のアプローチはあるか？
- パフォーマンスへの影響は？

## Step 4: IPアドレス管理とDHCP風機能

### このステップの目的
動的なIPアドレス割り当てとアドレス重複回避の仕組みを実装する

### 実装課題
1. **IPアドレスプール管理**
   ```go
   type IPPool struct {
       Network    *net.IPNet
       Gateway    net.IP
       allocated  map[string]net.IP  // containerID -> IP
       // どのような管理構造が適切か？
   }
   
   func (p *IPPool) AllocateIP(containerID string) (net.IP, error) {
       // どのような割り当てアルゴリズムを使用するか？
   }
   ```

2. **永続化機能**
   ```go
   // IPアドレス割り当て情報の永続化
   // ファイル？データベース？どこに保存するか？
   ```

3. **解放処理**
   ```go
   func (p *IPPool) ReleaseIP(containerID string) error {
       // IPアドレスの解放処理
   }
   ```

### 設計課題
- IPアドレスの枯渇対策は？
- コンテナ再起動時の同一IP保証は？
- 複数のネットワークセグメントへの対応は？

### 検証課題
```bash
# 複数コンテナでのIPアドレス割り当てテスト
./shibafu run --name web1 alpine
./shibafu run --name web2 alpine
./shibafu run --name web3 alpine

# 各コンテナのIPアドレス確認
./shibafu exec web1 ip addr show eth0
./shibafu exec web2 ip addr show eth0
./shibafu exec web3 ip addr show eth0
```

## Step 5: NAT設定とインターネット接続

### このステップの目的
iptablesを使用してNAT設定を行い、コンテナからインターネットへの接続を可能にする

### 調査すべき内容
- iptablesのNATテーブル
- MASQUERADEルールの動作原理
- FORWARDチェーンの役割
- Linuxカーネルのip_forward設定

### 実装課題
1. **基本NAT設定**
   ```go
   func setupNAT(bridgeName string) error {
       // MASQUERADEルールの追加
       // iptables -t nat -A POSTROUTING -s 172.20.0.0/24 -j MASQUERADE
       
       // FORWARDルールの追加
       // iptables -A FORWARD -i bridge -j ACCEPT
       
       // どのライブラリを使用するか？
       // github.com/coreos/go-iptables?
       return nil
   }
   ```

2. **ip_forward有効化**
   ```go
   func enableIPForward() error {
       // /proc/sys/net/ipv4/ip_forwardを1に設定
       return nil
   }
   ```

3. **cleanup処理**
   ```go
   func cleanupNAT(bridgeName string) error {
       // 追加したiptablesルールの削除
       return nil
   }
   ```

### 実験課題
```bash
# NAT設定前後でのパケットフロー確認
sudo tcpdump -i any icmp  # 別ターミナルで実行

# コンテナからの外部接続テスト
./shibafu exec webtest ping 8.8.8.8
./shibafu exec webtest wget http://example.com
```

### トラブルシューティング
- iptablesルールが正しく適用されているか？
- ip_forwardが有効になっているか？
- bridge-nf-call-iptablesの設定は？

## Step 6: ブリッジネットワークの完成

### このステップの目的
すべての機能を統合し、完全なブリッジネットワーク実装を完成させる

### 実装課題
1. **統合ネットワーク管理**
   ```go
   type NetworkManager struct {
       bridge   string
       ipPool   *IPPool
       // 他に必要なフィールドは？
   }
   
   func (nm *NetworkManager) CreateContainerNetwork(containerID string, pid int) error {
       // 完全なネットワーク設定フロー
   }
   ```

2. **コマンドライン統合**
   ```bash
   ./shibafu run --network bridge alpine ping google.com
   ./shibafu run --ip 172.20.0.100 alpine  # 手動IP指定
   ```

3. **情報表示機能**
   ```bash
   ./shibafu network ls
   ./shibafu network inspect bridge
   ```

### 設計課題
- 設定ファイルによるカスタマイズ対応
- 複数bridgeネットワークへの対応
- エラー回復機能の実装

### パフォーマンス検証
```bash
# ネットワーク性能テスト
./shibafu exec perftest iperf3 -c external-server
```

## Step 7: 高度なネットワーク機能

### このステップの目的
実用的なコンテナネットワークに必要な追加機能を実装する

### 実装課題
1. **ネットワーク分離**
   ```go
   // 異なるネットワークセグメントの作成
   func createIsolatedNetwork(name string, subnet string) error {
       // どのような実装が必要か？
   }
   ```

2. **帯域制限**
   ```go
   // tc (traffic control) を使用した帯域制限
   func setBandwidthLimit(interfaceName string, limitMbps int) error {
       // github.com/vishvananda/netlink のRate機能？
   }
   ```

3. **ネットワークポリシー**
   ```go
   // iptablesを使用したトラフィック制御
   func applyNetworkPolicy(containerID string, policy NetworkPolicy) error {
       // どのような実装が適切か？
   }
   ```

### 実験課題
- 異なるネットワークに接続されたコンテナ間での通信テスト
- 帯域制限の効果測定
- セキュリティポリシーの動作確認

## 発展課題

### 課題1: マルチホストネットワーク
- VXLAN overlay networkの実装
- 複数ホスト間でのコンテナ通信

### 課題2: ネットワークプラグイン
- CNIプラグインインターフェースの実装
- サードパーティネットワークソリューションとの統合

### 課題3: IPv6対応
- デュアルスタック（IPv4/IPv6）対応
- IPv6 only ネットワークの実装

### 課題4: 高可用性
- bridge冗長化
- 障害時の自動復旧

## 参考資料
- [Linux Advanced Routing & Traffic Control](https://lartc.org/)
- [netlink library documentation](https://pkg.go.dev/github.com/vishvananda/netlink)
- [iptables tutorial](https://www.netfilter.org/documentation/HOWTO/packet-filtering-HOWTO.html)
- [Container Networking: From Docker to Kubernetes](https://www.nginx.com/blog/container-networking-docker-kubernetes/)

## 完了チェックリスト
- [ ] 手動でのLinuxネットワーク設定ができた
- [ ] Go言語でのnetlink操作を実装できた
- [ ] libcontainerとの統合ができた
- [ ] 動的IPアドレス管理を実装できた
- [ ] NAT設定による外部接続を実現できた
- [ ] 完全なブリッジネットワークを実装できた

## 学習成果の確認
このPhaseを完了した時点で、以下を説明・実装できるようになっているべき：
1. Linuxネットワークスタックの基本的な仕組み
2. コンテナネットワークの設計と実装方法
3. iptablesを使用したトラフィック制御
4. 動的ネットワーク設定の自動化

次のPhase 3では、OverlayFSを使用した効率的なコンテナファイルシステム管理と、複数レイヤーの組み合わせによるイメージ管理を実装します。