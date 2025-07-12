# Phase 1: libcontainerで同等実装 - 詳細学習計画

## 学習目標
runcの内部で使用されているlibcontainerライブラリを直接使用し、Goプログラムからコンテナの作成・実行・管理を行う

## 事前準備

### 環境要件確認
- Go開発環境（Go 1.19以上推奨）
- Phase 0の完了（runcによるコンテナ実行の理解）
- Linux開発ツール（gcc、make等）
- git

### 事前学習課題
以下の概念について調べ、自分なりに整理せよ：
- libcontainerとは何か？runcとの関係は？
- Go言語でのシステムプログラミングの基礎
- Linuxシステムコールの基本概念
- プロセス管理とシグナル処理

## Step 1: libcontainerアーキテクチャの理解

### このステップの目的
libcontainerの内部構造とAPIを理解し、コンテナ作成に必要なコンポーネントを把握する

### 調査すべき内容
- libcontainerのGitHubリポジトリ構造
- Factory、Container、Process、Configの関係
- libcontainer.NewFactory()の役割
- configs.Configで設定可能なパラメータ

### 実装課題
1. libcontainerのソースコードを調査する
2. サンプルコードを読み、基本的な流れを理解する
3. 最小限のGoプログラムでlibcontainerを使用する準備をする

### 調査課題
```go
// 以下のインターフェースの役割を調べよ
type Factory interface {
    Create(id string, config *configs.Config) (Container, error)
    Load(id string) (Container, error)
    // その他のメソッドは？
}

type Container interface {
    Run(process *Process) error
    Start(process *Process) error
    // その他のメソッドは？
}
```

### 考察課題
- なぜruncはlibcontainerを直接使用しているのか？
- libcontainerの抽象化レベルは適切か？
- どのような設計パターンが使用されているか？

## Step 2: 開発環境のセットアップ

### このステップの目的
libcontainerを使用したGoプロジェクトの開発環境を構築する

### 実装課題
1. 新しいGoモジュールプロジェクトを作成する
2. 必要な依存関係を追加する
3. 基本的なプロジェクト構造を設計する

### 設計課題
以下のディレクトリ構造を作成せよ：
```
shibafu/
├── go.mod
├── go.sum
├── cmd/
│   └── shibafu/
│       └── main.go
├── internal/
│   ├── container/
│   ├── config/
│   └── factory/
└── README.md
```

### 依存関係課題
```go
// go.modに以下の依存関係を追加すべきか調査せよ
module shibafu

go ?

require (
    github.com/opencontainers/runc/libcontainer ?
    // 他に必要な依存関係は？
)
```

### トラブルシューティング準備
- Go modules、vendor、replace directiveの違いは？
- libcontainerの正しいimport pathは？
- 権限エラーが発生する可能性のある操作は？

## Step 3: 最小限のコンテナ作成

### このステップの目的
libcontainerを使用して最も基本的なコンテナを作成・実行する

### 実装課題
以下の機能を実装せよ：

```go
func main() {
    // 1. Factoryの作成
    factory := ?
    
    // 2. Configの設定
    config := &configs.Config{
        Rootfs: ?,
        // namespaceの設定は？
        // プロセスの設定は？
    }
    
    // 3. コンテナの作成
    container := ?
    
    // 4. プロセスの実行
    process := ?
    // 実行方法は？
}
```

### 設計課題
- Factoryの初期化に必要なパラメータは？
- configs.Configの最小限の設定項目は？
- プロセスの標準入出力をどう接続するか？
- エラーハンドリングはどうするか？

### 検証方法
```bash
# ビルドと実行
go build -o shibafu cmd/shibafu/main.go
sudo ./shibafu

# 期待される結果
# Phase 0で作成したrootfsを使用してコンテナが起動すること
```

## Step 4: コンテナライフサイクル管理

### このステップの目的
コンテナの作成、開始、停止、削除の完全なライフサイクルを管理する

### 実装課題
1. **コンテナ状態管理**
   ```go
   // コンテナの状態を取得する方法は？
   state := container.State()
   // State.Statusで何が取得できるか？
   ```

2. **シグナル処理**
   ```go
   // コンテナプロセスにシグナルを送信する方法は？
   err := container.Signal(syscall.SIGTERM)
   ```

3. **ライフサイクル管理**
   ```go
   // Create -> Start -> Wait -> Destroy の流れを実装せよ
   ```

### 考察課題
- runc createとrunc startの違いをlibcontainerで再現するには？
- プロセスの終了をどうやって検知するか？
- ゾンビプロセスを防ぐにはどうするか？

### 検証課題
```bash
# 複数コンテナの管理
./shibafu create container1 alpine /bin/sleep 30
./shibafu list
./shibafu start container1
./shibafu stop container1
./shibafu delete container1
```

## Step 5: 高度なnamespace設定

### このステップの目的
各種namespaceの詳細な設定と組み合わせを理解する

### 実装課題
configs.Configで以下のnamespace設定を実装せよ：

```go
config.Namespaces = []configs.Namespace{
    // PID namespace: どう設定する？
    // Mount namespace: どう設定する？
    // UTS namespace: どう設定する？
    // IPC namespace: どう設定する？
    // Network namespace: どう設定する？
    // User namespace: どう設定する？
}
```

### 実験課題
1. **PID namespace実験**
   - PID namespaceを無効化した場合の動作確認
   - initプロセスの役割を理解する

2. **Mount namespace実験**
   - 個別のマウントポイント設定
   - bind mountの使用方法

3. **User namespace実験**
   - UID/GIDマッピングの設定
   - 権限の制限と昇格

### 設計課題
```go
// User namespaceでのUID/GIDマッピング
config.UidMappings = []configs.IDMap{
    // どのような設定が必要？
}
config.GidMappings = []configs.IDMap{
    // どのような設定が必要？
}
```

## Step 6: cgroups統合

### このステップの目的
cgroupsを使用してコンテナのリソース制限を実装する

### 調査すべき内容
- cgroups v1とv2の違い
- libcontainerでのcgroups操作方法
- 主要なリソース制限項目

### 実装課題
```go
config.Cgroups = &configs.Cgroup{
    Name: ?,
    Resources: &configs.Resources{
        // メモリ制限: どう設定する？
        // CPU制限: どう設定する？
        // I/O制限: どう設定する？
    },
}
```

### 実験課題
1. **メモリ制限テスト**
   ```bash
   # 100MBのメモリ制限でコンテナを起動
   ./shibafu run --memory 100m alpine
   ```

2. **CPU制限テスト**
   ```bash
   # CPU使用率50%制限でコンテナを起動
   ./shibafu run --cpu-quota 50000 alpine
   ```

### 検証方法
```bash
# cgroupsの設定確認
cat /sys/fs/cgroup/memory/shibafu_*/memory.limit_in_bytes
cat /sys/fs/cgroup/cpu/shibafu_*/cpu.cfs_quota_us
```

## Step 7: 高度なプロセス管理

### このステップの目的
複雑なプロセス実行パターンと入出力処理を実装する

### 実装課題
1. **標準入出力の詳細制御**
   ```go
   process := &libcontainer.Process{
       Args:   []string{"/bin/sh"},
       Env:    []string{"PATH=/bin:/usr/bin"},
       // Stdin, Stdout, Stderrの接続方法は？
   }
   ```

2. **TTY/PTYサポート**
   ```go
   // 疑似端末を使用したインタラクティブシェル
   // どのライブラリを使用するか？
   ```

3. **プロセス実行モード**
   ```go
   // exec vs run の違いをどう実装するか？
   ```

### 考察課題
- docker execの動作をlibcontainerでどう再現するか？
- プロセスの親子関係はどうなっているか？
- セッション管理はどうするか？

## Step 8: エラーハンドリングとデバッグ

### このステップの目的
堅牢なエラーハンドリングとデバッグ機能を実装する

### 実装課題
1. **包括的なエラーハンドリング**
   ```go
   // どのようなエラーが発生しうるか？
   // 各エラーに対する適切な対処法は？
   ```

2. **ログ機能**
   ```go
   // libcontainerの内部動作をログ出力する方法は？
   ```

3. **デバッグ機能**
   ```bash
   # デバッグモードでの実行
   ./shibafu --debug run alpine
   ```

### トラブルシューティング
- "operation not permitted" エラーの原因と対策
- namespace作成失敗の原因と対策
- cgroups設定エラーの原因と対策

## 発展課題

### 課題1: CLIインターフェース改善
- CobraやCLIライブラリを使用したコマンドライン引数処理
- docker run風のオプション実装

### 課題2: コンテナイメージ対応
- tar.gz形式のrootfsの動的展開
- 複数のベースイメージからの選択

### 課題3: 並行処理
- 複数コンテナの同時実行
- Goroutineを使用した非同期処理

### 課題4: 監視機能
- コンテナメトリクスの収集
- ヘルスチェック機能の実装

## 参考資料
- [libcontainer GoDoc](https://pkg.go.dev/github.com/opencontainers/runc/libcontainer)
- [runc source code](https://github.com/opencontainers/runc)
- [configs package documentation](https://pkg.go.dev/github.com/opencontainers/runc/libcontainer/configs)
- Linux namespaces man pages: `man 7 namespaces`

## 完了チェックリスト
- [ ] libcontainerの基本的なAPIを理解できた
- [ ] Goプログラムからコンテナを作成・実行できた
- [ ] 完全なライフサイクル管理を実装できた
- [ ] 各種namespaceを適切に設定できた
- [ ] cgroupsによるリソース制限を実装できた
- [ ] 堅牢なエラーハンドリングを実装できた

## 学習成果の確認
このPhaseを完了した時点で、以下を説明・実装できるようになっているべき：
1. libcontainerアーキテクチャとその設計思想
2. プログラマティックなコンテナ管理の実装方法
3. namespaceとcgroupsの詳細な制御方法
4. runcが提供する機能の大部分をGoコードで再現

次のPhase 2では、コンテナに独立したネットワーク環境を提供し、外部との通信を可能にするネットワーク機能を実装します。