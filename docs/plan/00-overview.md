# コンテナランタイム学習計画：runcからPaaSまで

## Phase 0: runcでHello World
Goal: runcを使用してAlpine Linuxコンテナを起動し、基本的なコンテナ実行を体験する

実装内容:
* Docker imageからrootfsの抽出
* OCI Runtime Specification準拠のconfig.json作成
* runcコマンドでのコンテナ起動・停止

学習ポイント:
* OCI bundleの構造（rootfs + config.json）
* Linux namespaces（PID, Mount, UTS, IPC）の基本概念
* コンテナ実行の基本的な流れ

実装確認:
```bash
# rootfs準備
mkdir -p container/rootfs
docker export $(docker create alpine) | tar -C container/rootfs -xf -

# config.json作成してコンテナ実行
cd container && sudo runc run mycontainer

# コンテナ内でプロセス分離確認
/ # ps aux  # ホストのプロセスが見えないことを確認
/ # hostname  # 独立したホスト名を確認
```

## Phase 1: libcontainerで同等実装
Goal: runcの内部で使用されているlibcontainerを直接使用し、Goプログラムからコンテナを作成・実行する

実装内容:
* libcontainerのFactory作成とコンテナ設定
* configs.Configでのnamespace/cgroups設定
* プロセスの起動と標準入出力の接続
* コンテナライフサイクル管理の実装

学習ポイント:
* libcontainer APIの使用方法
* configs.Configの詳細なパラメータ
* コンテナプロセスの管理手法
* エラーハンドリングのパターン

実装確認:
```bash
# ビルドと実行
go build -o shibafu cmd/shibafu/main.go
sudo ./shibafu run alpine /bin/sh

# 複数コンテナの同時実行
sudo ./shibafu run --name container1 alpine sleep 100 &
sudo ./shibafu run --name container2 alpine sleep 100 &
sudo ./shibafu list  # 実行中のコンテナ一覧
```

## Phase 2: 基本的なコンテナネットワーク
Goal: コンテナに独立したネットワーク環境を提供し、外部ネットワークへの接続を可能にする

実装内容:
* Linux bridgeの作成と設定
* veth pairによるコンテナ・ホスト間接続
* Network namespaceへのインターフェース移動
* 基本的なNAT設定（MASQUERADE）
* IPアドレスの動的割り当て

学習ポイント:
* Linux bridgeとveth pairの仕組み
* Network namespaceの操作方法
* netlinkライブラリの使用方法
* iptablesによるNAT設定の基礎

実装確認:
```bash
# ブリッジ作成確認
ip link show ctr0
ip addr show ctr0  # 172.20.0.1/24

# ネットワーク付きコンテナ起動
./container-tool run --network bridge alpine ping -c 3 google.com

# コンテナのIP確認
./container-tool exec mycontainer ip addr show eth0
# 出力: inet 172.20.0.2/24
```

## Phase 3: 手動レイヤー管理とOverlayFS
Goal: 複数のレイヤーを手動で組み合わせ、OverlayFSを使用した効率的なrootfs管理を実現する

実装内容:
* レイヤーの手動準備機能（tar.gz形式）
* OverlayFSのlower/upper/work/mergedディレクトリ管理
* 複数レイヤーの動的組み合わせ
* コンテナごとの書き込みレイヤー分離
* レイヤー共有によるストレージ最適化

学習ポイント:
* OverlayFSの動作原理（Copy-on-Write）
* Union Filesystemの概念
* レイヤー管理のベストプラクティス
* mountシステムコールの使用方法

実装確認:
```bash
# レイヤー準備
./container-tool prepare-layer alpine-base /tmp/alpine-rootfs
./container-tool prepare-layer python-3.11 /opt/python-3.11
./container-tool prepare-layer app-code /home/user/myapp

# レイヤーを組み合わせてコンテナ実行
./container-tool run --layers alpine-base,python-3.11,app-code python app.py

# ストレージ使用量確認（レイヤー共有の効果）
du -sh /var/lib/container-tool/layers/
du -sh /var/lib/container-tool/containers/
```

## Phase 4: ポートマッピングとサービス公開
Goal: コンテナ内のサービスをホスト経由で外部に公開し、動的なポート割り当てを実現する

実装内容:
* iptables DNAT/SNATルールの自動設定
* 動的ポート割り当て（30000-32767範囲）
* ポートマッピング管理機能
* HTTP リバースプロキシの実装
* コンテナ名ベースのルーティング

学習ポイント:
* iptables NAT テーブルの詳細
* DNAT/SNATの動作原理
* Linux におけるポート管理
* L7プロキシの実装方法

実装確認:
```bash
# Webサーバーコンテナ起動（ポート自動割り当て）
./container-tool run --name web --publish 80 nginx
# 出力: Container web started, accessible at localhost:30001

# 手動ポート指定
./container-tool run --name api --publish 8080:3000 node-app

# アクセス確認
curl http://localhost:30001  # nginx welcome page
curl http://localhost:8080   # node app response
```

## Phase 5: 独自OCI Registry
Goal: OCI Distribution Specification準拠のコンテナレジストリを実装し、イメージの保存・配布を可能にする

実装内容:
* OCI Distribution API（push/pull）の実装
* Content-Addressable Storage（SHA256ベース）
* Manifest管理（OCI Image Manifest v1.0）
* Blob（レイヤー）のアップロード/ダウンロード
* レイヤー重複排除機能

学習ポイント:
* OCI Distribution Specificationの詳細
* Content-Addressable Storageの設計
* HTTP Range Requestsの処理
* Docker Registry プロトコルの仕組み

実装確認:
```bash
# レジストリ起動
./container-tool registry serve --port 5000

# レイヤーをレジストリにpush
./container-tool push-layer alpine-base localhost:5000/base/alpine:latest
./container-tool push-layer python-3.11 localhost:5000/runtime/python:3.11

# マニフェスト作成とpush
./container-tool create-manifest \
  --layers base/alpine:latest,runtime/python:3.11 \
  localhost:5000/apps/python-app:v1

# レジストリからpullして実行
./container-tool run localhost:5000/apps/python-app:v1 python --version
```

## Phase 6: コンテナ間ネットワークとDNS
Goal: コンテナ間で名前解決可能な内部ネットワークを構築し、マルチネットワーク環境を実現する

実装内容:
* 内部DNSサーバーの実装
* 複数の独立したブリッジネットワーク
* コンテナ名による自動DNS登録
* ネットワーク間の分離とルーティング
* コンテナの複数ネットワーク接続

学習ポイント:
* DNSプロトコルの基礎
* ネットワークセグメンテーション
* Linux policy routingの概念
* コンテナネットワークモデルの設計

実装確認:
```bash
# ネットワーク作成
./container-tool network create frontend 172.21.0.0/24
./container-tool network create backend 172.22.0.0/24

# バックエンドネットワークにDBコンテナ
./container-tool run --name postgres --network backend postgres:14

# 両ネットワークに接続するAPIコンテナ
./container-tool run --name api --network backend --network frontend node-api

# APIコンテナからDB接続（名前解決）
./container-tool exec api ping postgres.backend.local
./container-tool exec api psql -h postgres.backend.local -U user
```

## Phase 7: 統合Container Runtime
Goal: レジストリ、レイヤー管理、ネットワークを統合し、完全なコンテナランタイムを実現する

実装内容:
* レジストリからの自動イメージ取得
* レイヤーキャッシュ管理
* ネットワークの自動設定
* コンテナメタデータ管理
* 統一されたCLIインターフェース

学習ポイント:
* コンポーネント間の協調動作
* 非同期処理とエラーハンドリング
* 状態管理とトランザクション
* システム全体のアーキテクチャ設計

実装確認:
```bash
# 統合CLIでのコンテナ実行
./container-tool run \
  --image localhost:5000/apps/webapp:latest \
  --network production \
  --publish 80 \
  --env NODE_ENV=production \
  --name webapp

# コンテナ情報確認
./container-tool inspect webapp
# 出力: IP、ポート、レイヤー、ネットワーク情報等

# ログ確認
./container-tool logs -f webapp
```

## Phase 8: 簡易PaaS API
Goal: RESTful APIを通じてアプリケーションのデプロイを自動化し、PaaSとしての機能を提供する

実装内容:
* デプロイメントAPI（コード受付、ビルド、実行）
* アプリケーションルーティング（サブドメイン方式）
* 環境変数とシークレット管理
* ヘルスチェックと自動再起動
* デプロイメント履歴とロールバック

学習ポイント:
* PaaSアーキテクチャの設計
* マイクロサービス間の連携
* 非同期ジョブ処理
* アプリケーションライフサイクル管理

実装確認:
```bash
# アプリケーションデプロイ
curl -X POST http://localhost:8080/api/deploy \
  -F "app_name=myapp" \
  -F "runtime=node:18" \
  -F "code=@app.tar.gz" \
  -F "env[DATABASE_URL]=postgres://..." 

# レスポンス
{
  "deployment_id": "dep-123",
  "status": "building",
  "url": "http://myapp.platform.local"
}

# デプロイメント状態確認
curl http://localhost:8080/api/deployments/dep-123

# アプリケーションアクセス
curl http://myapp.platform.local
```