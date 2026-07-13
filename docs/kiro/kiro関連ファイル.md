# .kiro ディレクトリ構成と各ファイルの説明

## 概要

`.kiro/` ディレクトリは、Kiro（AI開発環境）がスペック駆動開発で使用するファイルを格納する場所です。要件定義・設計・実装タスクを構造化して管理します。

## ディレクトリ構成

```
.kiro/
└── specs/
    └── {feature-name}/        # 機能ごとのディレクトリ（ケバブケース）
        ├── .config.kiro       # スペック設定ファイル
        ├── requirements.md    # 要件定義書
        ├── design.md          # 設計書
        └── tasks.md           # 実装タスクリスト
```

## 各ファイルの説明

### `.config.kiro`

スペックのメタ情報を保持するJSON設定ファイル。

| フィールド | 説明 | 値の例 |
|-----------|------|--------|
| `specId` | スペックの一意識別子（UUID） | `"a616f8fd-dd2f-41a6-a5f5-6cba94a82f10"` |
| `workflowType` | ワークフローの種類 | `"requirements-first"` / `"design-first"` |
| `specType` | スペックの種類 | `"feature"` / `"bugfix"` |

**例:**
```json
{"specId": "a616f8fd-...", "workflowType": "requirements-first", "specType": "feature"}
```

---

### `requirements.md`

機能の要件を定義するドキュメント。ユーザーストーリーと受け入れ基準で構成される。

**構成:**
- **Introduction** — システムの概要説明
- **Glossary** — 用語定義
- **Requirements** — 各要件（ユーザーストーリー＋受け入れ基準）

**受け入れ基準のフォーマット（EARS形式）:**
- `WHEN` — トリガー条件
- `WHERE` — 状態条件
- `WHILE` — 継続条件
- `IF ... THEN` — 条件分岐
- `THE {システム} SHALL` — システムの振る舞い

---

### `design.md`

技術設計を記述するドキュメント。アーキテクチャ、コンポーネント、データモデル、正当性プロパティを含む。

**構成:**
- **Overview** — 技術スタックと設計方針
- **Architecture** — システム構成図（Mermaid図）
- **Components and Interfaces** — APIエンドポイント一覧、インターフェース定義
- **Data Models** — データモデル、ER図、スキーマ定義
- **Correctness Properties** — プロパティベーステストで検証する正当性条件
- **Error Handling** — エラーレスポンス形式と例外設計
- **Testing Strategy** — テスト方針（プロパティベーステスト、ユニットテスト、統合テスト）

---

### `tasks.md`

実装タスクのリスト。設計書に基づき、実装の順序と依存関係を定義する。

**構成:**
- タスクの依存グラフ
- 各タスクの詳細（実装内容、受け入れ基準、依存タスク）
- タスクのステータス管理（`pending` / `in_progress` / `completed`）

---

## ワークフローの種類

| ワークフロー | 進行順序 | 適用場面 |
|-------------|---------|---------|
| Requirements-First | 要件 → 設計 → タスク | ビジネス要件が明確で技術設計が未定の場合 |
| Design-First | 設計 → 要件 → タスク | 技術アプローチが明確で要件の形式化が必要な場合 |
| Bugfix | バグ要件 → 設計 → タスク | 既存の不具合修正 |
| Quick Plan | 確認 → 要件 → 設計 → タスク（自動） | 迅速に実装計画を立てたい場合 |

## スペックの種類

| 種類 | 説明 |
|------|------|
| `feature` | 新機能の追加・既存機能の拡張 |
| `bugfix` | 既存の不具合修正 |

## その他の .kiro ディレクトリ

`.kiro/` 配下には specs 以外にも以下のディレクトリが配置可能です（本プロジェクトでは未使用）:

| ディレクトリ | 説明 |
|-------------|------|
| `.kiro/steering/` | エージェントへの追加指示・ルール（チーム標準、プロジェクト情報など） |
| `.kiro/settings/` | MCP（Model Context Protocol）サーバー設定など |
| `.kiro/hooks/` | IDEイベントに応じた自動実行フック |
