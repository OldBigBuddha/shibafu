# Phase 0: runcでHello World - 詳細学習計画

## 学習目標
runcを使用してAlpine Linuxコンテナを手動で起動し、コンテナ実行の最も基本的な仕組みを体験・理解する

## 事前準備

### 環境要件確認
- Linux環境（Ubuntu、CentOS、Arch Linux等）
- root権限またはsudo権限
- Docker実行環境
- runcのインストール

### 事前学習課題
以下の概念について調べ、自分なりに整理せよ：
- コンテナとは何か？仮想マシンとの違いは？
- Linux namespaceとは？どのような種類があるか？
- cgroupsとは何か？
- OCI（Open Container Initiative）とは？

## Step 1: Docker Imageの構造理解とrootfs抽出

### このステップの目的
Docker imageの内部構造を理解し、コンテナ実行に必要なファイルシステム（rootfs）を抽出する

### 調査すべき内容
- Docker imageのレイヤー構造
- rootfsとは何か？
- `docker export` vs `docker save` の違い
- Alpine Linuxの特徴

### 実装課題
1. Alpine Linux imageを取得する
2. 適切なディレクトリ構造を作成する
3. Docker imageからrootfsを抽出する
4. 抽出されたファイルシステムの内容を確認・分析する

### 注意点
- どのディレクトリに作業用フォルダを作るべきか？
- 抽出時のファイル権限はどうなるか？
- rootfsに含まれるべきディレクトリ構造は？

### 確認ポイント
```bash
# 抽出されたrootfsに以下が含まれているか確認
ls -la container/rootfs/
# /bin, /etc, /lib, /usr, /var などの標準的なUnixディレクトリ構造

# Alpine Linuxらしい特徴的なファイルは？
find container/rootfs -name "*alpine*"
```

### 考察課題
- なぜコンテナにはkernelが含まれていないのか？
- `/proc`, `/sys`, `/dev` などの特殊ディレクトリが空なのはなぜか？

## Step 2: OCI Runtime Specificationの理解

### このステップの目的
OCI Runtime Specの基本概念を理解し、config.jsonで設定すべき項目を把握する

### 調査すべき内容
- OCI Runtime Specificationの公式ドキュメント
- config.jsonの必須フィールド
- runtimeの概念とruncの位置づけ
- OCI bundleとは何か？

### 実装課題
1. OCI Runtime Specの公式ドキュメントを読む
2. config.jsonのサンプルを見つけて分析する
3. 最小限のconfig.jsonを設計する

### 考察課題
- なぜ標準化が必要だったのか？
- runcとcontainerdの関係は？
- DockerはOCIとどう関係しているか？

## Step 3: config.jsonの作成

### このステップの目的
Alpine Linuxコンテナ実行に必要な最小限のconfig.jsonを作成する

### 調査すべき内容
- `process` セクションの必須フィールド
- `root` セクションの設定方法
- `linux` セクションでのnamespace設定
- 各namespaceの役割（PID, Mount, UTS, IPC, Network, User）

### 実装課題
以下の要素を含むconfig.jsonを作成せよ：

```json
{
  "ociVersion": "?", 
  "process": {
    "terminal": ?,
    "user": {},
    "args": ["?"],
    "env": ["?"],
    "cwd": "?"
  },
  "root": {
    "path": "?"
  },
  "linux": {
    "namespaces": [
      {"type": "?"},
      {"type": "?"}
    ]
  }
}
```

### 設計課題
- どのnamespaceを有効にすべきか？それぞれの理由は？
- terminalをtrueにする場合とfalseにする場合の違いは？
- 環境変数には何を設定すべきか？
- ユーザーIDはどう設定すべきか？

### 検証方法
```bash
# config.jsonの文法チェック
runc spec --help  # ヒントを得る
# 自分で作ったconfig.jsonが正しいか検証する方法を調べよ
```

## Step 4: runcでのコンテナ実行

### このステップの目的
作成したOCI bundleを使ってruncでコンテナを実際に起動する

### 調査すべき内容
- runcコマンドの基本的な使用方法
- `runc run` vs `runc create` + `runc start` の違い
- コンテナのライフサイクル管理
- runcが要求するディレクトリ構造

### 実装課題
1. 作業ディレクトリに適切に移動する
2. runcでコンテナを起動する
3. コンテナ内でコマンドを実行する
4. コンテナを正常に停止する

### トラブルシューティング準備
以下のエラーが起きた場合の対処法を事前に調査せよ：
- "permission denied" エラー
- "container already exists" エラー
- "invalid config.json" エラー
- namespace関連のエラー

### 実行例（穴埋め問題）
```bash
# 適切なディレクトリに移動
cd ?

# コンテナ実行（コンテナ名とコマンドを指定）
sudo runc run ? ?

# 期待される結果：Alpine Linuxのシェルが起動
```

## Step 5: 実行結果の検証と理解

### このステップの目的
起動したコンテナが期待通りに分離されているかを確認し、namespaceの効果を理解する

### 検証課題
コンテナ内で以下のコマンドを実行し、結果を分析せよ：

1. **プロセス分離の確認**
   ```bash
   ps aux  # どのプロセスが見えるか？
   ```

2. **ホスト名の分離確認**
   ```bash
   hostname  # ホストと同じか？違うか？
   ```

3. **ファイルシステムの分離確認**
   ```bash
   ls /  # ホストのルートディレクトリと同じか？
   mount  # マウントポイントは？
   ```

4. **ネットワークの状態確認**
   ```bash
   ip addr show  # ネットワークインターフェースは？
   ```

### 分析課題
- なぜコンテナ内から見えるプロセス数が少ないのか？
- ホスト名が変わっている理由は？
- `/proc`, `/sys` などはどこからマウントされているか？
- なぜネットワークが制限されているのか？

### 発展検証
別のターミナルで以下を確認せよ：
```bash
# ホスト側から見たコンテナプロセス
ps aux | grep runc

# ホスト側のnamespace一覧
lsns

# コンテナ用に作成されたnamespaceを特定せよ
```

## Step 6: 複数コンテナの実行実験

### このステップの目的
複数のコンテナを同時実行し、相互の分離を確認する

### 実装課題
1. 2つ目のOCI bundleディレクトリを作成
2. 異なる名前で2つ目のコンテナを起動
3. 各コンテナの分離状況を確認

### 考察課題
- 2つのコンテナは互いを認識できるか？
- プロセスIDの割り当てはどうなっているか？
- ファイルシステムの変更は影響し合うか？

## 発展課題

### 課題1: config.jsonのカスタマイズ
- 異なる実行コマンド（`/bin/sleep 30`など）でコンテナを起動せよ
- 環境変数を追加して動作を確認せよ
- 作業ディレクトリを変更してみよ

### 課題2: namespace実験
- 各namespaceを個別に無効化して動作の違いを観察せよ
- PID namespaceなしの場合、ホストのプロセスは見えるか？
- Mount namespaceなしの場合、何が起こるか？

### 課題3: エラー分析
- 意図的に壊れたconfig.jsonを作成し、エラーメッセージを分析せよ
- 存在しないコマンドを実行しようとした場合は？
- rootfsディレクトリが存在しない場合は？

### 課題4: runcコマンド探求
```bash
runc --help  # 利用可能なサブコマンドを調査
runc list    # 実行中コンテナの一覧
runc state   # コンテナの状態確認
```

## 参考資料
- [OCI Runtime Specification](https://github.com/opencontainers/runtime-spec)
- [runc GitHub Repository](https://github.com/opencontainers/runc)
- Linux namespace man pages: `man 7 namespaces`
- Alpine Linux Documentation

## 完了チェックリスト
- [ ] Docker imageからrootfsを手動抽出できた
- [ ] 最小限のconfig.jsonを自力で作成できた
- [ ] runcでコンテナの起動・停止ができた
- [ ] namespace分離の効果を確認・理解できた
- [ ] 複数コンテナの同時実行ができた
- [ ] エラーが発生した際の基本的な対処ができた

## 学習成果の確認
このPhaseを完了した時点で、以下を説明できるようになっているべき：
1. コンテナ実行に最低限必要なファイルとは何か
2. OCI bundleの構成要素とその役割
3. Linux namespaceがコンテナ分離に果たす役割
4. runcがコンテナランタイムとして担う責任範囲

次のPhase 1では、このruncの内部で使用されているlibcontainerを直接使用して、Goプログラムからコンテナを制御します。