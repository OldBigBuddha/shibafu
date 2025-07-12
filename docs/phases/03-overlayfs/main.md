# Phase 3: 手動レイヤー管理とOverlayFS - 詳細学習計画

## 学習目標
複数のレイヤーを手動で組み合わせ、OverlayFSを使用した効率的なrootfs管理を実現し、Copy-on-Write機能による高速なコンテナ起動とストレージ最適化を理解・実装する

## 事前準備

### 環境要件確認
- Phase 2の完了（基本的なコンテナネットワーク）
- OverlayFS対応Linuxカーネル（Linux 3.18以降）
- 十分なディスク容量（実験用レイヤー保存のため）
- mount/umountコマンドの実行権限

### 事前学習課題
以下の概念について調べ、自分なりに整理せよ：
- Union Filesystemとは？従来のファイルシステムとの違い
- Copy-on-Write（CoW）の概念と利点
- OverlayFSの内部動作原理
- Dockerイメージレイヤーの仕組み
- inode、ハードリンク、シンボリックリンクの違い

## Step 1: OverlayFSの基本実験

### このステップの目的
手動でOverlayFSをマウントし、その動作原理と各ディレクトリの役割を理解する

### 調査すべき内容
- OverlayFSのlower、upper、work、mergedディレクトリの意味
- mountコマンドでのoverlay指定方法
- OverlayFS特有の制限事項
- xattr（拡張属性）とOverlayFSの関係

### 実験課題
以下の手順で基本的なOverlayFSを構築し、各段階での状態を観察せよ：

```bash
# 1. 実験用ディレクトリ構造の作成
mkdir -p overlay-test/{lower,upper,work,merged}

# 2. lowerディレクトリにベースファイルを作成
echo "base content" > overlay-test/lower/base.txt
echo "shared content" > overlay-test/lower/shared.txt

# 3. OverlayFSのマウント
sudo mount -t overlay overlay \
    -o lowerdir=?,upperdir=?,workdir=? \
    ?

# 4. マウント後の状態確認
ls -la overlay-test/merged/
cat overlay-test/merged/base.txt
```

### 実験タスク
各操作後に以下を確認し、結果を記録せよ：

1. **読み取り専用操作**
   ```bash
   # mergedディレクトリからファイルを読み取り
   cat overlay-test/merged/base.txt
   # どのディレクトリからファイルが読まれているか？
   ```

2. **書き込み操作**
   ```bash
   # mergedディレクトリに新しいファイルを作成
   echo "new content" > overlay-test/merged/new.txt
   # ファイルはどこに保存されるか？
   
   # 既存ファイルの変更
   echo "modified" > overlay-test/merged/shared.txt
   # lowerとupperの両方にあるファイルはどうなるか？
   ```

3. **削除操作**
   ```bash
   # lowerにあるファイルの削除
   rm overlay-test/merged/base.txt
   # 実際には何が起こっているか？upperディレクトリを確認せよ
   ```

### 考察課題
- なぜworkディレクトリが必要なのか？
- OverlayFSでの削除操作が「whiteout」という概念で実装される理由は？
- CoWが発生するタイミングと条件は？

## Step 2: 複数レイヤーの重ね合わせ

### このステップの目的
複数のlowerレイヤーを使用し、Dockerライクなイメージレイヤー構造を再現する

### 実装課題
以下のレイヤー構造を手動で作成せよ：

```bash
# レイヤー1: OS base layer
mkdir -p layers/base
# Alpine Linuxの基本ファイルシステムをコピー
# どのような内容を配置すべきか？

# レイヤー2: Runtime layer (Python)
mkdir -p layers/python
# Pythonランタイムファイルを配置
# /usr/bin/python3, /usr/lib/python3.x などを想定

# レイヤー3: Application layer
mkdir -p layers/app
# アプリケーションコードを配置
echo "print('Hello from layered container')" > layers/app/app.py
```

### 実装タスク
```bash
# 複数レイヤーのOverlayFSマウント
sudo mount -t overlay overlay \
    -o lowerdir=layers/app:layers/python:layers/base,upperdir=?,workdir=? \
    ?

# 動作確認
# マウントされたファイルシステムでPythonスクリプトが実行できるか？
```

### 設計課題
- レイヤーの順序はどのような意味を持つか？
- 同名ファイルが複数レイヤーにある場合の優先順位は？
- レイヤー数の上限とパフォーマンスの関係は？

### 分析タスク
各レイヤーからの読み取り優先順位を確認：
```bash
# 同名ファイルを各レイヤーに作成
echo "from base" > layers/base/test.txt
echo "from python" > layers/python/test.txt  
echo "from app" > layers/app/test.txt

# マウント後にどの内容が表示されるか？
cat merged/test.txt
```

## Step 3: Goによるレイヤー管理システム

### このステップの目的
レイヤーの作成、管理、組み合わせを自動化するGoプログラムを実装する

### 実装課題
以下の機能を含むレイヤー管理システムを設計・実装せよ：

```go
type Layer struct {
    ID          string
    Path        string
    Parent      *Layer
    Metadata    LayerMetadata
    // 他に必要なフィールドは？
}

type LayerManager struct {
    StorageRoot string
    layers      map[string]*Layer
    // どのような管理構造が適切か？
}

func (lm *LayerManager) CreateLayer(id string, parentID string) (*Layer, error) {
    // レイヤーディレクトリの作成
    // メタデータの初期化
    // 親レイヤーとの関連付け
    return nil, nil
}

func (lm *LayerManager) MountLayers(layerIDs []string, mountPoint string) error {
    // 指定されたレイヤーを順序付け
    // OverlayFSのmountコマンド構築・実行
    // どのようなエラーハンドリングが必要か？
    return nil
}
```

### 設計課題
1. **レイヤー識別**
   - レイヤーIDの生成方法（UUID？ハッシュ？）
   - レイヤー間の依存関係管理

2. **メタデータ管理**
   ```go
   type LayerMetadata struct {
       CreatedAt   time.Time
       Size        int64
       Parent      string
       // 他に必要な情報は？
   }
   ```

3. **ストレージレイアウト**
   ```
   /var/lib/container-tool/layers/
   ├── <layer-id-1>/
   │   ├── diff/        # レイヤーの実際の内容
   │   └── metadata.json
   ├── <layer-id-2>/
   └── overlays/        # マウントポイント管理
       ├── <container-id>/
       │   ├── upper/
       │   ├── work/
       │   └── merged/
   ```

### 実装タスク
```go
// 使用例
lm := NewLayerManager("/var/lib/container-tool")

// ベースレイヤーの作成
baseLayer, err := lm.CreateLayer("alpine-base", "")
err = lm.ExtractTarToLayer("alpine-base", "alpine.tar.gz")

// アプリケーションレイヤーの作成
appLayer, err := lm.CreateLayer("my-app", "alpine-base")
err = lm.CopyToLayer("my-app", "./app.py", "/usr/local/bin/app.py")

// レイヤーの組み合わせとマウント
err = lm.MountLayers([]string{"alpine-base", "my-app"}, "/tmp/container-root")
```

## Step 4: Copy-on-Write の実装と最適化

### このステップの目的
効率的なCoW機能を実装し、ストレージ使用量とパフォーマンスを最適化する

### 実装課題
1. **書き込み検出機能**
   ```go
   func (lm *LayerManager) CreateWritableLayer(baseLayerIDs []string, containerID string) error {
       // 読み取り専用レイヤーの上に書き込み可能レイヤーを作成
       // upperdirとworkdirの準備
       return nil
   }
   ```

2. **レイヤー変更追跡**
   ```go
   func (lm *LayerManager) GetLayerChanges(layerID string) ([]FileChange, error) {
       // 追加、変更、削除されたファイルの一覧を取得
       // どのような実装が効率的か？
   }
   
   type FileChange struct {
       Path   string
       Type   ChangeType  // Added, Modified, Deleted
       Size   int64
   }
   ```

3. **効率的なファイルコピー**
   ```go
   func (lm *LayerManager) OptimizedCopy(src, dst string) error {
       // reflink、hardlink、regularコピーの自動選択
       // どのような判断基準を使用するか？
   }
   ```

### パフォーマンス実験
以下の観点でパフォーマンスを測定・比較せよ：

```bash
# ベンチマークスクリプトの作成
time create_container_with_layers()
time create_container_without_layers()

# ストレージ使用量の比較
du -sh /var/lib/container-tool/layers/
du -sh /var/lib/container-tool/containers/
```

### 最適化課題
- 同一ファイルの重複排除
- レイヤーキャッシュの実装
- ガベージコレクション機能

## Step 5: libcontainerとの完全統合

### このステップの目的
Phase 1のlibcontainerベースコンテナランタイムにレイヤー機能を統合する

### 実装課題
1. **コンテナ作成時のレイヤー処理**
   ```go
   func (cm *ContainerManager) CreateContainerWithLayers(
       containerID string, 
       layerIDs []string, 
       config *configs.Config) error {
       
       // レイヤーマウントの実行
       mountPoint, err := cm.layerManager.MountLayers(layerIDs, containerID)
       
       // libcontainerのconfigにrootfsを設定
       config.Rootfs = mountPoint
       
       // コンテナの作成と実行
       return cm.factory.Create(containerID, config)
   }
   ```

2. **コンテナ削除時のクリーンアップ**
   ```go
   func (cm *ContainerManager) RemoveContainer(containerID string) error {
       // コンテナの停止
       // レイヤーのアンマウント
       // 書き込みレイヤーの削除（オプション）
       return nil
   }
   ```

3. **レイヤーキャッシュとの統合**
   ```go
   func (cm *ContainerManager) RunWithImage(imageLayers []string, cmd []string) error {
       // キャッシュされたレイヤーの確認
       // 必要に応じてレイヤーの取得
       // コンテナの作成と実行
   }
   ```

### CLI統合
```bash
# レイヤー管理コマンド
./shibafu layer create alpine-base ./alpine-rootfs.tar.gz
./shibafu layer create python-runtime ./python.tar.gz --parent alpine-base
./shibafu layer list
./shibafu layer inspect alpine-base

# レイヤーを使用したコンテナ実行
./shibafu run --layers alpine-base,python-runtime python3 /app/script.py
```

## Step 6: 高度なレイヤー機能

### このステップの目的
実用的なコンテナシステムに必要な高度なレイヤー管理機能を実装する

### 実装課題
1. **レイヤー差分管理**
   ```go
   func (lm *LayerManager) CreateDiffLayer(baseLayerID, targetLayerID string) (*Layer, error) {
       // 2つのレイヤー間の差分を新しいレイヤーとして作成
   }
   
   func (lm *LayerManager) ApplyDiffLayer(baseLayerID, diffLayerID string) (*Layer, error) {
       // 差分レイヤーをベースレイヤーに適用
   }
   ```

2. **レイヤー圧縮・展開**
   ```go
   func (lm *LayerManager) CompressLayer(layerID string) error {
       // レイヤーを圧縮してストレージ使用量を削減
   }
   
   func (lm *LayerManager) DecompressLayer(layerID string) error {
       // 圧縮されたレイヤーを展開
   }
   ```

3. **レイヤーの整合性検証**
   ```go
   func (lm *LayerManager) VerifyLayer(layerID string) error {
       // チェックサムによるレイヤー整合性確認
   }
   ```

### パフォーマンス最適化
- 並列レイヤー処理
- レイヤーの事前ロード
- メモリマップファイルの活用

### 実験課題
```bash
# 大量レイヤーでのパフォーマンステスト
for i in {1..50}; do
    ./shibafu layer create layer-$i ./test-data.tar.gz --parent layer-$((i-1))
done

# 深いレイヤー階層でのコンテナ起動時間測定
time ./shibafu run --layers layer-1,layer-10,layer-20,layer-30,layer-50 alpine
```

## Step 7: レイヤー可視化とデバッグ機能

### このステップの目的
レイヤー構造の可視化とデバッグ支援機能を実装する

### 実装課題
1. **レイヤー関係図の生成**
   ```go
   func (lm *LayerManager) GenerateLayerGraph() (Graph, error) {
       // レイヤーの依存関係をグラフ構造で表現
   }
   
   func (lm *LayerManager) ExportDOT(graph Graph) (string, error) {
       // Graphviz DOT形式での出力
   }
   ```

2. **レイヤー使用量分析**
   ```bash
   ./shibafu layer analyze
   # 出力例:
   # Layer ID: alpine-base
   # Size: 2.5MB
   # Used by: 15 containers
   # Efficiency: 87% (space saved vs individual copies)
   ```

3. **レイヤー依存関係チェック**
   ```go
   func (lm *LayerManager) CheckDependencies(layerID string) ([]Dependency, error) {
       // 循環依存や破損した依存関係の検出
   }
   ```

### デバッグ機能
```bash
# レイヤー内容の比較
./shibafu layer diff layer-1 layer-2

# レイヤーファイル構造の表示
./shibafu layer tree layer-id

# マウント状態の診断
./shibafu layer diagnose
```

## 発展課題

### 課題1: Content-Addressable Storage
- SHA256ハッシュベースのレイヤー管理
- 重複レイヤーの自動検出と統合

### 課題2: 分散レイヤー管理
- 複数ホスト間でのレイヤー共有
- レイヤーの分散キャッシュシステム

### 課題3: セキュリティ強化
- レイヤーの暗号化
- デジタル署名による検証

### 課題4: 他ファイルシステムへの対応
- Btrfs、ZFSでのCoW機能活用
- FUSE filesystemの実装

## 参考資料
- [OverlayFS documentation](https://www.kernel.org/doc/Documentation/filesystems/overlayfs.txt)
- [Docker image and layer management](https://docs.docker.com/storage/storagedriver/)
- [Union filesystems comparison](https://docs.docker.com/storage/storagedriver/select/)
- [Linux mount(8) manual](https://man7.org/linux/man-pages/man8/mount.8.html)

## 完了チェックリスト
- [ ] OverlayFSの基本動作を理解し手動操作できた
- [ ] 複数レイヤーの重ね合わせを実装できた
- [ ] Goによるレイヤー管理システムを実装できた
- [ ] Copy-on-Write機能を効率的に実装できた
- [ ] libcontainerとの完全統合ができた
- [ ] 高度なレイヤー管理機能を実装できた

## 学習成果の確認
このPhaseを完了した時点で、以下を説明・実装できるようになっているべき：
1. OverlayFSとUnion Filesystemの動作原理
2. 効率的なレイヤー管理システムの設計と実装
3. Copy-on-Writeによるストレージ最適化手法
4. 大規模コンテナ環境でのレイヤー戦略

次のPhase 4では、コンテナ内のサービスをホスト経由で外部に公開し、動的なポート割り当てとロードバランシングを実装します。