# Phase 1 事前学習ガイド - libcontainer

このドキュメントは、Phase 1でlibcontainerを使用したコンテナランタイム実装を行うために必要な基礎知識をまとめたものです。

## 1. libcontainerアーキテクチャ

### 1.1 libcontainerとは

**定義と目的**
- runcの内部で使用されているコアライブラリ
- Go言語でコンテナ操作を行うための低レベルAPI
- Linux名前空間、cgroups、セキュリティ機能への統一的なインターフェース

**設計思想**
- OCI Runtime Specificationの実装
- プラットフォーム依存部分の抽象化
- 拡張可能なアーキテクチャ

### 1.2 主要コンポーネント

```
libcontainer/
├── Factory         # コンテナの作成・管理
├── Container       # コンテナのライフサイクル管理
├── Process         # プロセスの実行制御
├── Config          # コンテナ設定の定義
├── State           # コンテナ状態の管理
└── Hooks           # ライフサイクルフック
```

**コンポーネント間の関係**
```
Factory ──作成──> Container
   │                  │
   │                  ├──実行──> Process
   │                  │
   └──設定──> Config  └──状態──> State
```

## 2. Go言語のシステムプログラミング

### 2.1 必須知識

**システムコール操作**
```go
// syscallパッケージの使用
import "syscall"

// 例: プロセスの作成
pid, err := syscall.ForkExec(
    "/bin/ls",
    []string{"ls", "-la"},
    &syscall.ProcAttr{},
)
```

**エラーハンドリング**
```go
// Go標準のエラーハンドリングパターン
if err != nil {
    return fmt.Errorf("failed to create container: %w", err)
}
```

### 2.2 並行処理パターン

**Goroutineとチャネル**
```go
// プロセス監視の例
done := make(chan error)
go func() {
    done <- process.Wait()
}()

select {
case err := <-done:
    // プロセス終了処理
case <-time.After(timeout):
    // タイムアウト処理
}
```

**sync パッケージの活用**
```go
// 複数のコンテナ管理
var wg sync.WaitGroup
var mu sync.Mutex
containers := make(map[string]Container)
```

## 3. libcontainer API詳解

### 3.1 Factory インターフェース

```go
type Factory interface {
    // コンテナの作成
    Create(id string, config *configs.Config) (Container, error)
    
    // 既存コンテナのロード
    Load(id string) (Container, error)
    
    // ファクトリの種類
    Type() string
    
    // 起動時の検証
    StartInitialization() error
}
```

**Factory作成の実装例**
```go
import "github.com/opencontainers/runc/libcontainer"

// Factoryの初期化
factory, err := libcontainer.New(
    "/run/containers",    // ルートディレクトリ
    libcontainer.Cgroupfs, // cgroup管理方式
    libcontainer.InitArgs(os.Args[0], "init"),
)
```

### 3.2 Container インターフェース

```go
type Container interface {
    // コンテナID取得
    ID() string
    
    // 状態取得
    State() (*State, error)
    
    // プロセス実行
    Run(process *Process) error
    Start(process *Process) error
    
    // コンテナ制御
    Pause() error
    Resume() error
    Destroy() error
    
    // シグナル送信
    Signal(s os.Signal) error
}
```

### 3.3 Config 構造体

```go
type Config struct {
    // 基本設定
    Rootfs       string
    Readonlyfs   bool
    Hostname     string
    
    // Namespace設定
    Namespaces Namespaces
    
    // Cgroups設定
    Cgroups *Cgroup
    
    // セキュリティ設定
    Capabilities *Capabilities
    Seccomp      *Seccomp
    AppArmor     string
    
    // ネットワーク設定
    Networks []*Network
    
    // マウント設定
    Mounts []*Mount
}
```

## 4. Linux システムプログラミング基礎

### 4.1 プロセス管理

**clone() システムコール**
```c
// cloneフラグの組み合わせ
int flags = CLONE_NEWPID | CLONE_NEWNS | CLONE_NEWNET;
pid_t pid = clone(child_func, stack_top, flags | SIGCHLD, arg);
```

**プロセス状態遷移**
```
作成 ──> 実行可能 ──> 実行中
 │         ↑          │
 └─────────┴──待機中──┘
           │
          終了
```

### 4.2 ファイルディスクリプタ管理

**重要な概念**
- 標準入出力のリダイレクト
- パイプとFIFO
- エポール（epoll）による効率的なI/O監視

```go
// ファイルディスクリプタの複製
if err := syscall.Dup2(int(slave.Fd()), 0); err != nil {
    return err
}
```

### 4.3 シグナル処理

**シグナルの種類と用途**
| シグナル | 番号 | 用途 |
|---------|------|------|
| SIGTERM | 15 | 正常終了要求 |
| SIGKILL | 9 | 強制終了 |
| SIGCHLD | 17 | 子プロセスの状態変化 |
| SIGSTOP | 19 | プロセス停止 |
| SIGCONT | 18 | プロセス再開 |

## 5. セキュリティ機能

### 5.1 Capabilities

**Linux Capabilitiesの理解**
```go
// 必要最小限の権限設定
caps := &configs.Capabilities{
    Bounding: []string{
        "CAP_CHOWN",
        "CAP_DAC_OVERRIDE",
        "CAP_FSETID",
        "CAP_SETGID",
        "CAP_SETUID",
    },
}
```

### 5.2 Seccomp

**システムコールフィルタリング**
```go
// Seccompプロファイル例
seccomp := &configs.Seccomp{
    DefaultAction: configs.Allow,
    Syscalls: []*configs.Syscall{
        {
            Name:   "clone",
            Action: configs.Errno,
            Args: []*configs.Arg{
                {
                    Index: 0,
                    Op:    configs.MaskedEqual,
                    Value: syscall.CLONE_NEWUSER,
                },
            },
        },
    },
}
```

## 6. デバッグとトラブルシューティング

### 6.1 デバッグ手法

**ログレベルの活用**
```go
// 環境変数でデバッグレベル制御
if os.Getenv("DEBUG") != "" {
    logrus.SetLevel(logrus.DebugLevel)
}
```

**straceでのシステムコール追跡**
```bash
# libcontainerの動作追跡
strace -f -e trace=clone,setns,unshare ./your-container-app
```

### 6.2 一般的なエラーと対処法

| エラー | 原因 | 対処法 |
|--------|------|--------|
| "operation not permitted" | 権限不足 | sudo実行、capability確認 |
| "no such file or directory" | パス誤り | 絶対パス使用 |
| "device or resource busy" | リソース使用中 | プロセス確認、強制削除 |
| "invalid argument" | 設定誤り | config検証 |

## 7. 実装パターンとベストプラクティス

### 7.1 エラーハンドリング

```go
// エラーの詳細化
func createContainer(id string) error {
    container, err := factory.Create(id, config)
    if err != nil {
        return fmt.Errorf("failed to create container %s: %w", id, err)
    }
    
    // クリーンアップの保証
    defer func() {
        if err != nil {
            container.Destroy()
        }
    }()
    
    return nil
}
```

### 7.2 リソース管理

```go
// リソースのライフサイクル管理
type ContainerManager struct {
    mu         sync.RWMutex
    containers map[string]Container
    factory    Factory
}

func (m *ContainerManager) Create(id string, config *configs.Config) error {
    m.mu.Lock()
    defer m.mu.Unlock()
    
    if _, exists := m.containers[id]; exists {
        return fmt.Errorf("container %s already exists", id)
    }
    
    // ...
}
```

## 8. 参考資料

### 8.1 必読ソースコード
- [libcontainer source](https://github.com/opencontainers/runc/tree/main/libcontainer)
- [configs package](https://github.com/opencontainers/runc/tree/main/libcontainer/configs)
- [factory implementation](https://github.com/opencontainers/runc/blob/main/libcontainer/factory_linux.go)

### 8.2 参考ドキュメント
- [Go システムプログラミング](https://golang.org/pkg/syscall/)
- [Linux Programmer's Manual](https://man7.org/linux/man-pages/)
- [OCI Runtime Specification](https://github.com/opencontainers/runtime-spec)

### 8.3 理解度確認チェックリスト

- [ ] libcontainerの主要インターフェースを説明できる
- [ ] Factoryパターンの利点を理解している
- [ ] Go言語でのエラーハンドリングパターンを実装できる
- [ ] システムコールレベルでコンテナの動作を理解している
- [ ] 並行処理とリソース管理の実装ができる

これらの知識を身につけることで、Phase 1のlibcontainerを使用した実装に効果的に取り組むことができます。