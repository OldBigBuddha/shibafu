# Phase 0 事前学習ガイド

このドキュメントは、Phase 0の学習をスムーズに進めるために必要な基礎知識を体系的にまとめたものです。

## 1. コンテナ技術の基礎概念

### 1.1 コンテナとは何か？

**コンテナの定義**
- プロセスレベルの仮想化技術
- アプリケーションとその依存関係をパッケージ化
- ホストOSのカーネルを共有しながら、独立した実行環境を提供

**コンテナの主な特徴**
1. **軽量性**: 仮想マシンと比べて起動が高速、リソース消費が少ない
2. **ポータビリティ**: 環境に依存せず同じ動作を保証
3. **分離性**: プロセス、ファイルシステム、ネットワークが隔離される
4. **スケーラビリティ**: 素早い起動・停止により動的なスケーリングが可能

### 1.2 仮想マシン（VM）との違い

| 特性 | コンテナ | 仮想マシン |
|------|---------|------------|
| **仮想化レベル** | OSレベル（プロセス） | ハードウェアレベル |
| **カーネル** | ホストと共有 | 独自のゲストOS |
| **起動時間** | ミリ秒〜秒 | 数十秒〜分 |
| **リソース消費** | MB単位 | GB単位 |
| **分離レベル** | プロセス分離 | 完全分離 |
| **用途** | マイクロサービス、CI/CD | レガシーアプリ、異種OS |

**アーキテクチャの違い**
```
仮想マシン:
+------------------+------------------+
|   App A          |   App B          |
|   Bins/Libs      |   Bins/Libs      |
|   Guest OS       |   Guest OS       |
+------------------+------------------+
|          Hypervisor                 |
+-------------------------------------+
|          Host OS                    |
+-------------------------------------+
|          Hardware                   |
+-------------------------------------+

コンテナ:
+------------------+------------------+
|   App A          |   App B          |
|   Bins/Libs      |   Bins/Libs      |
+------------------+------------------+
|       Container Runtime             |
+-------------------------------------+
|          Host OS                    |
+-------------------------------------+
|          Hardware                   |
+-------------------------------------+
```

## 2. Linux Namespaces

### 2.1 Namespaceとは

Linux Namespaceは、プロセスが見ることができるシステムリソースの範囲を制限する機能です。各namespaceは特定のグローバルリソースを分離します。

### 2.2 7種類のNamespace

#### 1. **PID Namespace** (Process ID)
- **役割**: プロセスIDの分離
- **効果**: コンテナ内でPID 1から始まる独立したプロセス空間
- **確認方法**: 
  ```bash
  # ホストで実行
  ps aux | wc -l  # 多数のプロセス
  
  # コンテナ内で実行
  ps aux | wc -l  # 少数のプロセスのみ
  ```

#### 2. **Mount Namespace**
- **役割**: マウントポイントの分離
- **効果**: コンテナ独自のファイルシステムビュー
- **確認方法**:
  ```bash
  # マウント情報の確認
  mount | grep -v "^cgroup"
  ```

#### 3. **UTS Namespace** (Unix Time-sharing System)
- **役割**: ホスト名とドメイン名の分離
- **効果**: コンテナごとに異なるホスト名設定が可能
- **確認方法**:
  ```bash
  hostname  # コンテナとホストで異なる値
  ```

#### 4. **IPC Namespace** (Inter-Process Communication)
- **役割**: System V IPC、POSIXメッセージキューの分離
- **効果**: コンテナ間でIPCリソースが見えない
- **確認方法**:
  ```bash
  ipcs -a  # 共有メモリ、セマフォ、メッセージキューの確認
  ```

#### 5. **Network Namespace**
- **役割**: ネットワークスタックの分離
- **効果**: 独立したネットワークインターフェース、ルーティングテーブル
- **確認方法**:
  ```bash
  ip addr show  # インターフェース一覧
  ip route show # ルーティングテーブル
  ```

#### 6. **User Namespace**
- **役割**: ユーザーID、グループIDの分離
- **効果**: コンテナ内のrootがホストの非特権ユーザーにマップ可能
- **セキュリティ**: 最も重要なセキュリティ機能の一つ

#### 7. **Cgroup Namespace**
- **役割**: cgroupルートディレクトリの分離
- **効果**: コンテナからはcgroup階層の一部のみが見える

### 2.3 Namespaceの操作

**関連するシステムコール**
- `clone()`: 新しいnamespaceでプロセスを作成
- `unshare()`: 既存プロセスを新しいnamespaceに移動
- `setns()`: 既存のnamespaceに参加

**フラグ定数**
```c
CLONE_NEWPID    // PID namespace
CLONE_NEWNS     // Mount namespace
CLONE_NEWUTS    // UTS namespace
CLONE_NEWIPC    // IPC namespace
CLONE_NEWNET    // Network namespace
CLONE_NEWUSER   // User namespace
CLONE_NEWCGROUP // Cgroup namespace
```

## 3. Control Groups (cgroups)

### 3.1 cgroupsとは

Control Groups（cgroups）は、プロセスグループのリソース使用量を制限・監視・分離するLinuxカーネル機能です。

### 3.2 主なリソースコントローラー

#### 1. **Memory Controller**
- メモリ使用量の制限
- OOM（Out of Memory）キラーの制御
- メモリ使用統計の取得

#### 2. **CPU Controller**
- CPU使用時間の制限
- CPU共有の重み付け
- CPUアフィニティの設定

#### 3. **Block I/O Controller**
- ディスクI/Oの帯域制限
- I/O優先度の設定

#### 4. **PIDs Controller**
- プロセス数の制限
- Fork爆弾の防止

### 3.3 cgroups v1 vs v2

| 特徴 | cgroups v1 | cgroups v2 |
|------|------------|------------|
| **階層構造** | コントローラーごとに独立 | 統一階層 |
| **インターフェース** | 複雑 | シンプル |
| **リソース管理** | 個別管理 | 統合管理 |

### 3.4 cgroupsファイルシステム

```bash
# cgroups v1の例
/sys/fs/cgroup/
├── cpu/
│   ├── cgroup.procs
│   ├── cpu.shares
│   └── cpu.cfs_quota_us
├── memory/
│   ├── cgroup.procs
│   ├── memory.limit_in_bytes
│   └── memory.usage_in_bytes
└── pids/
    ├── cgroup.procs
    └── pids.max
```

## 4. Open Container Initiative (OCI)

### 4.1 OCIとは

Open Container Initiative（OCI）は、コンテナ技術の業界標準を策定する団体です。Linux Foundationのプロジェクトとして、主要なコンテナ技術企業が参加しています。

### 4.2 OCI仕様の種類

#### 1. **Runtime Specification**
- コンテナの実行方法を定義
- config.jsonの形式を規定
- ライフサイクル操作を標準化

#### 2. **Image Specification**
- コンテナイメージの形式を定義
- レイヤー構造の仕様
- マニフェストとコンフィグの形式

#### 3. **Distribution Specification**
- イメージの配布方法を定義
- レジストリAPIの標準化
- プッシュ/プル操作のプロトコル

### 4.3 OCI Bundle

OCI Bundleは、コンテナを実行するために必要な最小限の構成要素です：

```
bundle/
├── config.json    # ランタイム設定
└── rootfs/        # ルートファイルシステム
    ├── bin/
    ├── etc/
    ├── lib/
    ├── usr/
    └── ...
```

### 4.4 config.jsonの構造

```json
{
  "ociVersion": "1.0.2",      // OCI仕様バージョン
  "process": {                // プロセス設定
    "terminal": true,         // TTY接続
    "user": {                 // ユーザー設定
      "uid": 0,
      "gid": 0
    },
    "args": ["/bin/sh"],      // 実行コマンド
    "env": [                  // 環境変数
      "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
    ],
    "cwd": "/"                // 作業ディレクトリ
  },
  "root": {                   // ルートファイルシステム
    "path": "rootfs",
    "readonly": false
  },
  "linux": {                  // Linux固有設定
    "namespaces": [           // 使用するnamespace
      {"type": "pid"},
      {"type": "mount"},
      {"type": "ipc"},
      {"type": "uts"},
      {"type": "network"}
    ]
  }
}
```

## 5. 実践的な確認方法

### 5.1 Namespace確認コマンド

```bash
# 現在のプロセスのnamespace確認
ls -la /proc/self/ns/

# 特定のプロセスのnamespace確認
ls -la /proc/<PID>/ns/

# namespace一覧の表示
lsns

# 特定タイプのnamespace表示
lsns -t pid
lsns -t net
```

### 5.2 cgroups確認コマンド

```bash
# cgroup階層の確認
systemctl status

# プロセスのcgroup確認
cat /proc/<PID>/cgroup

# リソース使用状況
systemd-cgtop
```

### 5.3 デバッグツール

```bash
# straceでシステムコールを追跡
strace -f runc run mycontainer

# namespace操作の確認
nsenter --target <PID> --all

# cgroup操作の確認
cgget -r memory.limit_in_bytes /
```

## 6. 参考資料とさらなる学習

### 6.1 必読ドキュメント
- [Linux Namespaces man page](https://man7.org/linux/man-pages/man7/namespaces.7.html)
- [cgroups v2 Documentation](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html)
- [OCI Runtime Specification](https://github.com/opencontainers/runtime-spec/blob/main/spec.md)
- [runc Documentation](https://github.com/opencontainers/runc#readme)

### 6.2 実験環境の準備

```bash
# 必要なツールのインストール（Ubuntu/Debian）
sudo apt-get update
sudo apt-get install -y \
    runc \
    cgroup-tools \
    util-linux \
    strace \
    docker.io

# Alpine Linuxイメージの準備
docker pull alpine:latest
```

### 6.3 理解度確認チェックリスト

- [ ] コンテナと仮想マシンの違いを3つ以上説明できる
- [ ] 7種類のLinux Namespaceの役割を説明できる
- [ ] cgroupsがリソース制限に果たす役割を理解している
- [ ] OCI仕様の3つのタイプを説明できる
- [ ] config.jsonの基本構造を理解している

これらの基礎知識を身につけることで、Phase 0の実装課題により深い理解を持って取り組むことができます。