# Shibafu: コンテナ技術学習プロジェクト

ShibafuはPlatform-as-a-Service（PaaS）システムを通じて、コンテナ技術の深い理解を得るための教育プロジェクトです。

## 学習アプローチ

Linux カーネル機能からPaaSまで、9つのフェーズで段階的に実装することで、コンテナ技術のあらゆる側面を網羅的に学習します。

### 学習の特徴
- **実装主体**: すべての機能を自分で実装
- **段階的構築**: 前のフェーズの上に次の機能を積み重ね
- **包括的理解**: Linuxカーネルから分散システムまで

## 9つの学習フェーズ

| フェーズ | 技術領域 | 主な学習内容 |
|---------|---------|-------------|
| **Phase 0** | コンテナ基礎 | OCI仕様、namespaces、基本的なコンテナ実行 |
| **Phase 1** | システムプログラミング | libcontainer API、プロセス管理 |
| **Phase 2** | ネットワーク | Linux bridge、veth pair、NAT |
| **Phase 3** | ストレージ | OverlayFS、レイヤー管理、Copy-on-Write |
| **Phase 4** | サービス公開 | iptables、ポートマッピング、L7プロキシ |
| **Phase 5** | イメージ配布 | OCI Distribution、Content-Addressable Storage |
| **Phase 6** | サービス発見 | DNS、複数ネットワーク、セグメンテーション |
| **Phase 7** | 統合ランタイム | 全コンポーネント統合、アーキテクチャ設計 |
| **Phase 8** | PaaS実装 | デプロイメント自動化、アプリケーション管理 |

## 期待される学習成果

### 技術的理解の獲得
- **Linux システムプログラミング**: namespaces、cgroups、mount操作
- **ネットワークプログラミング**: netlink、iptables、ブリッジネットワーク
- **分散システム**: サービス発見、負荷分散、API設計

### 実装によって構築するシステム
- コンテナランタイム（Docker相当）
- イメージレジストリ（Docker Hub相当）
- 統合プラットフォーム（基本的なKubernetes相当）
- PaaS システム（Heroku相当）

## 詳細なドキュメント

各フェーズの詳細な実装ガイドは `docs/phase/` ディレクトリに格納されています:

- [Phase 0: Hello World](phase/00-helloworld.md) - runcを使った基本的なコンテナ実行
- [Phase 1: libcontainer](phase/01-libcontainer.md) - Goによるコンテナランタイム
- [Phase 2: ネットワーク](phase/02-network.md) - コンテナネットワーク基盤
- [Phase 3: OverlayFS](phase/03-overlayfs.md) - レイヤー型ストレージ
- [Phase 4: ポートマッピング](phase/04-port-mapping.md) - サービス公開機能
- [Phase 5: OCI Registry](phase/05-oci-registry.md) - イメージ配布システム
- [Phase 6: DNS](phase/06-container-dns.md) - コンテナ間通信
- [Phase 7: 統合ランタイム](phase/07-integrated-runtime.md) - 完全なランタイム
- [Phase 8: PaaS API](phase/08-paas-api.md) - アプリケーションプラットフォーム

## 学習の進め方

1. **各フェーズのドキュメントを熟読**
2. **概念の理解**: 関連するLinux技術の調査
3. **自力実装**: 全機能を自分で実装
4. **動作確認**: 各フェーズの検証手順を実行
5. **理解の深化**: なぜそう動くのかを重視

このプロジェクトを通じて、コンテナ技術の全容を体系的に理解し、現代的なアプリケーション基盤の構築スキルを獲得できます。