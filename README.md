# Valkey ドキュメント

Amazon ElastiCache for Valkey を使って、設計・実装・運用を行うチームのためのドキュメントです。
対象は Valkey 9.1、情報は 2026 年 11 月の時点のものです。

## 読み方

| 立場 | 読む範囲 | 時間の目安 |
|---|---|---|
| Valkey の全体像をつかみたい人 | [概要編](docs/overview.md)の ★ の節 | 30 分程度 |
| 設計の考え方まで知りたい人 | [概要編](docs/overview.md)の全体 | 1 時間程度 |
| 設計・実装・レビュー・運用を担当する人 | 詳細編の全章（[00 章](docs/00-introduction.md)から） | — |

## 詳細編

### 第1部 基礎

- [00. はじめに](docs/00-introduction.md)
- [01. Valkey とは](docs/01-what-is-valkey.md)
- [02. 基本操作とキー](docs/02-basics-and-keys.md)
- [03. データ型 (1) String と Hash](docs/03-string-and-hash.md)
- [04. データ型 (2) Set・Sorted Set・List・Stream](docs/04-set-zset-list-stream.md)
- [05. データモデリングとキー設計](docs/05-data-modeling.md)
- [06. 複数コマンドの扱い](docs/06-multi-command.md)
- [07. JSON と検索](docs/07-json-and-search.md)

### 第2部 アプリ開発

- [08. FastAPI からの利用](docs/08-fastapi.md)
- [09. データの投入・入れ替え・削除](docs/09-loading-swapping-deleting.md)
- [10. ローカル環境とテスト](docs/10-local-and-testing.md)

### 第3部 インフラ・運用

- [11. メモリ管理とエビクション](docs/11-memory.md)
- [12. 永続化・バックアップ・復旧](docs/12-persistence-and-backup.md)
- [13. レプリケーションとクラスター](docs/13-replication-and-cluster.md)
- [14. ElastiCache の構成選定](docs/14-elasticache-design.md)
- [15. セキュリティ](docs/15-security.md)
- [16. 監視](docs/16-monitoring.md)
- [17. トラブルシューティング](docs/17-troubleshooting.md)
- [18. バージョンアップとメンテナンス](docs/18-upgrade-and-maintenance.md)

### 第4部 横断

- [19. アンチパターン・障害パターン集](docs/19-antipatterns.md)

### 付録

- [A. 扱わない機能の一覧](docs/appendix-a-out-of-scope.md)
- [B. コマンド早見表](docs/appendix-b-commands.md)
- [C. チェックリスト](docs/appendix-c-checklists.md)
- [D. 用語集](docs/appendix-d-glossary.md)

## リポジトリの構成

| パス | 内容 |
|---|---|
| `docs/` | ドキュメント本体 |
| `planning/` | このドキュメントの企画資料（要件、調査結果、構成）。読者向けではありません |
