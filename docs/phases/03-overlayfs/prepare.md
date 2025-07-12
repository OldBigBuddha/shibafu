# Phase 3 事前学習ガイド - OverlayFS とレイヤー管理

このドキュメントは、Phase 3でOverlayFSを使用したレイヤー管理システムを実装するために必要な基礎知識をまとめたものです。

## 1. Union Filesystem の概念

### 1.1 Union Filesystem とは

**基本概念**
- 複数のファイルシステムを統合して単一のビューを提供
- レイヤー構造による効率的なストレージ管理
- Copy-on-Write (CoW) による差分管理

**主なUnion Filesystem**
| 名称 | 特徴 | 使用例 |
|------|------|--------|
| OverlayFS | カーネル統合、高性能 | Docker (現在) |
| AUFS | 歴史が長い、安定 | Docker (以前) |
| UnionFS | オリジナル実装 | 研究用途 |
| DeviceMapper | ブロックレベル | LVM統合環境 |

### 1.2 レイヤー構造の利点

**ストレージ効率**
```
アプリA: [Base OS] + [Runtime] + [App A]
アプリB: [Base OS] + [Runtime] + [App B]
         └─共有──┘   └─共有──┘
```

**高速なデプロイメント**
- 共通レイヤーのダウンロード不要
- 差分のみの転送で済む
- キャッシュの効果的な活用

## 2. OverlayFS 詳解

### 2.1 OverlayFS アーキテクチャ

**ディレクトリ構造**
```
overlay/
├── lower/      # 読み取り専用レイヤー（複数可）
├── upper/      # 読み書き可能レイヤー
├── work/       # OverlayFS内部作業用
└── merged/     # 統合されたビュー（マウントポイント）
```

**レイヤーの統合**
```
        merged (統合ビュー)
           ↑
    ┌──────┴──────┐
    │   upper     │ ← 書き込み層
    └──────┬──────┘
    ┌──────┴──────┐
    │  lower2     │ ← 読み取り専用
    └──────┬──────┘
    ┌──────┴──────┐
    │  lower1     │ ← 読み取り専用
    └─────────────┘
```

### 2.2 OverlayFS の動作原理

**ファイル操作の挙動**

1. **読み取り操作**
   - upperから順に検索
   - 最初に見つかったファイルを返す

2. **書き込み操作**
   - upperレイヤーに書き込み
   - lowerのファイルはcopy-upされる

3. **削除操作**
   - whiteoutファイルで削除をマーク
   - 実際のファイルは削除されない

**Copy-on-Write の仕組み**
```bash
# lowerにあるファイルを変更する場合
1. lowerからupperへファイルをコピー
2. upperのファイルを変更
3. 以降はupperのファイルが使用される
```

## 3. OverlayFS の実装

### 3.1 マウントオプション

**基本的なマウント**
```bash
mount -t overlay overlay \
  -o lowerdir=/lower,upperdir=/upper,workdir=/work \
  /merged
```

**複数のlowerレイヤー**
```bash
mount -t overlay overlay \
  -o lowerdir=/lower1:/lower2:/lower3,upperdir=/upper,workdir=/work \
  /merged
```

**Go言語での実装**
```go
import "syscall"

func mountOverlay(lower, upper, work, merged string) error {
    opts := fmt.Sprintf("lowerdir=%s,upperdir=%s,workdir=%s", lower, upper, work)
    return syscall.Mount("overlay", merged, "overlay", 0, opts)
}
```

### 3.2 マウントオプションの詳細

| オプション | 説明 | 必須 |
|-----------|------|------|
| lowerdir | 読み取り専用ディレクトリ | ○ |
| upperdir | 読み書き可能ディレクトリ | △ |
| workdir | 作業用ディレクトリ | △ |
| redirect_dir | ディレクトリ名変更の追跡 | × |
| index | ハードリンクサポート | × |
| xino | inode番号の管理 | × |

## 4. レイヤー管理システムの設計

### 4.1 レイヤーストレージ設計

**ディレクトリ構造**
```
/var/lib/container-tool/
├── layers/           # レイヤーストレージ
│   ├── sha256/
│   │   ├── abc123.../  # レイヤーコンテンツ
│   │   └── def456.../
│   └── metadata.json   # レイヤーメタデータ
├── containers/       # コンテナ別upper/work
│   ├── container1/
│   │   ├── upper/
│   │   └── work/
│   └── container2/
└── mounts/          # マウントポイント
```

### 4.2 レイヤー識別とメタデータ

**レイヤーメタデータ構造**
```go
type Layer struct {
    ID          string    // SHA256ハッシュ
    Parent      string    // 親レイヤーID
    Size        int64     // レイヤーサイズ
    Created     time.Time // 作成日時
    Author      string    // 作成者
    Comment     string    // コメント
    DiffIDs     []string  // 差分ID
}

type LayerStore struct {
    layers map[string]*Layer
    mu     sync.RWMutex
}
```

**コンテンツアドレッシング**
```go
func calculateLayerID(tarPath string) (string, error) {
    file, err := os.Open(tarPath)
    if err != nil {
        return "", err
    }
    defer file.Close()
    
    hash := sha256.New()
    if _, err := io.Copy(hash, file); err != nil {
        return "", err
    }
    
    return fmt.Sprintf("sha256:%x", hash.Sum(nil)), nil
}
```

## 5. tar アーカイブの処理

### 5.1 レイヤーのインポート/エクスポート

**tarアーカイブの展開**
```go
import "archive/tar"

func extractTar(tarPath, destDir string) error {
    file, err := os.Open(tarPath)
    if err != nil {
        return err
    }
    defer file.Close()
    
    tr := tar.NewReader(file)
    for {
        header, err := tr.Next()
        if err == io.EOF {
            break
        }
        if err != nil {
            return err
        }
        
        target := filepath.Join(destDir, header.Name)
        
        switch header.Typeflag {
        case tar.TypeDir:
            os.MkdirAll(target, os.FileMode(header.Mode))
        case tar.TypeReg:
            createFile(target, tr, header.Mode)
        case tar.TypeSymlink:
            os.Symlink(header.Linkname, target)
        }
    }
    
    return nil
}
```

### 5.2 差分レイヤーの作成

**変更の検出と差分作成**
```go
func createDiffTar(oldDir, newDir string) (*bytes.Buffer, error) {
    buf := new(bytes.Buffer)
    tw := tar.NewWriter(buf)
    defer tw.Close()
    
    // 新規・変更ファイルの検出
    err := filepath.Walk(newDir, func(path string, info os.FileInfo, err error) error {
        if err != nil {
            return err
        }
        
        relPath, _ := filepath.Rel(newDir, path)
        oldPath := filepath.Join(oldDir, relPath)
        
        // ファイルが新規または変更されている場合
        if isNewOrModified(oldPath, path) {
            addToTar(tw, path, relPath, info)
        }
        
        return nil
    })
    
    // 削除ファイルのwhiteout作成
    addWhiteouts(tw, oldDir, newDir)
    
    return buf, err
}
```

## 6. ストレージ最適化

### 6.1 重複排除

**ハッシュベースの重複検出**
```go
type DeduplicationStore struct {
    hashes map[string]string // hash -> layerID
    mu     sync.RWMutex
}

func (ds *DeduplicationStore) CheckDuplicate(content []byte) (string, bool) {
    hash := sha256.Sum256(content)
    hashStr := fmt.Sprintf("%x", hash)
    
    ds.mu.RLock()
    defer ds.mu.RUnlock()
    
    if layerID, exists := ds.hashes[hashStr]; exists {
        return layerID, true
    }
    return "", false
}
```

### 6.2 圧縮とストレージ効率

**レイヤー圧縮**
```go
import "compress/gzip"

func compressLayer(src, dst string) error {
    srcFile, err := os.Open(src)
    if err != nil {
        return err
    }
    defer srcFile.Close()
    
    dstFile, err := os.Create(dst)
    if err != nil {
        return err
    }
    defer dstFile.Close()
    
    gw := gzip.NewWriter(dstFile)
    defer gw.Close()
    
    _, err = io.Copy(gw, srcFile)
    return err
}
```

## 7. セキュリティとパーミッション

### 7.1 ファイルパーミッションの保持

**拡張属性とケーパビリティ**
```go
import "golang.org/x/sys/unix"

func preserveAttributes(src, dst string) error {
    // 基本的なパーミッション
    info, err := os.Stat(src)
    if err != nil {
        return err
    }
    os.Chmod(dst, info.Mode())
    
    // 拡張属性
    attrs, err := unix.Listxattr(src)
    if err == nil {
        for _, attr := range attrs {
            value, _ := unix.Getxattr(src, attr)
            unix.Setxattr(dst, attr, value, 0)
        }
    }
    
    return nil
}
```

### 7.2 レイヤー検証

**チェックサム検証**
```go
func verifyLayer(layerPath string, expectedHash string) error {
    actualHash, err := calculateLayerID(layerPath)
    if err != nil {
        return err
    }
    
    if actualHash != expectedHash {
        return fmt.Errorf("layer verification failed: expected %s, got %s", 
            expectedHash, actualHash)
    }
    
    return nil
}
```

## 8. トラブルシューティング

### 8.1 一般的な問題と解決策

| 問題 | 原因 | 解決策 |
|------|------|--------|
| "invalid argument" | カーネルサポート不足 | OverlayFSモジュール確認 |
| "device or resource busy" | マウント済み | アンマウント後に再試行 |
| "no space left" | work領域不足 | ディスク容量確認 |
| パーミッションエラー | UID/GIDマッピング | 正しい権限で実行 |

### 8.2 デバッグ手法

**OverlayFSの状態確認**
```bash
# マウント状態の確認
mount | grep overlay

# レイヤー構成の確認
ls -la /proc/*/mountinfo | xargs grep overlay

# inode使用状況
df -i
```

**パフォーマンス分析**
```bash
# I/Oパフォーマンス
iostat -x 1

# ファイルシステムキャッシュ
free -h
```

## 9. ベストプラクティス

### 9.1 レイヤー設計指針

**レイヤーの粒度**
- ベースOS: 変更頻度が低い
- ランタイム: 言語/フレームワーク
- アプリケーション: 頻繁に更新
- 設定: 環境別の設定

**レイヤーサイズの最適化**
```dockerfile
# 良い例: 単一レイヤーで処理
RUN apt-get update && \
    apt-get install -y package && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# 悪い例: 複数レイヤーで無駄
RUN apt-get update
RUN apt-get install -y package
RUN apt-get clean
```

### 9.2 実装パターン

**レイヤーマネージャーの設計**
```go
type LayerManager struct {
    store      *LayerStore
    mounter    *Mounter
    downloader *Downloader
    cache      *Cache
}

func (lm *LayerManager) PrepareLayer(layerID string) (string, error) {
    // キャッシュ確認
    if path, exists := lm.cache.Get(layerID); exists {
        return path, nil
    }
    
    // ダウンロード
    if !lm.store.Exists(layerID) {
        if err := lm.downloader.Download(layerID); err != nil {
            return "", err
        }
    }
    
    // マウント準備
    return lm.mounter.Prepare(layerID)
}
```

## 10. 参考資料

### 10.1 必読ドキュメント
- [OverlayFS Documentation](https://www.kernel.org/doc/html/latest/filesystems/overlayfs.html)
- [Docker Storage Drivers](https://docs.docker.com/storage/storagedriver/)
- [OCI Image Specification](https://github.com/opencontainers/image-spec)

### 10.2 参考実装
- [containerd snapshotter](https://github.com/containerd/containerd/tree/main/snapshots)
- [Docker graph driver](https://github.com/moby/moby/tree/master/daemon/graphdriver)

### 10.3 理解度確認チェックリスト

- [ ] Union Filesystemの概念を説明できる
- [ ] OverlayFSの動作原理を理解している
- [ ] Copy-on-Writeの仕組みを説明できる
- [ ] レイヤー管理システムの設計ができる
- [ ] tarアーカイブの処理を実装できる
- [ ] ストレージ最適化の手法を理解している

これらの知識を身につけることで、Phase 3のOverlayFSを使用したレイヤー管理実装に効果的に取り組むことができます。