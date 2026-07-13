# Implementation Plan: Kiro CLI EC2セットアップガイド

## Overview

AWS EC2インスタンス（AlmaLinux 9）上にKiro CLI開発環境を構築するためのガイドドキュメントをマークダウン形式で作成する。ガイドはセクションごとに段階的に構築し、各セクションにバリデーションコマンドとトラブルシューティングを含める。

## Tasks

- [x] 1. ガイドドキュメントの基本構造を作成する
  - [x] 1.1 ガイドドキュメントのファイルとフロントマター・目次を作成する
    - `docs/kiro-cli-ec2-setup-guide.md` を作成
    - タイトル、概要、対象読者、前提知識を記載
    - 全セクションへのリンクを含む目次を作成
    - _Requirements: 1.1, 1.2, 1.3, 1.4_

- [x] 2. 前提条件セクションを作成する
  - [x] 2.1 EC2インスタンスの推奨スペックと前提条件を記載する
    - 推奨インスタンスタイプ（t3.medium以上）、CPU、メモリ要件を記載
    - 対応OS（AlmaLinux 9）を明記
    - 最小ディスク容量要件（20GB以上）を記載
    - ネットワーク接続要件（インターネットアクセス必須）を記載
    - VS Code Remote SSH接続が確立済みであることを前提として明記
    - _Requirements: 1.1, 1.2, 1.3, 1.4_

- [x] 3. Node.js環境構築セクションを作成する
  - [x] 3.1 Node.jsインストール手順（dnf経由・nvm経由の両方）を記載する
    - Node.jsの推奨バージョンを明記
    - dnf経由のインストール手順（NodeSourceリポジトリ追加を含む）を記載
    - nvm経由のインストール手順を記載（代替方法として）
    - バージョン確認コマンド（`node --version`, `npm --version`）による検証手順を記載
    - AlmaLinux 9のデフォルトリポジトリにNode.jsが含まれない場合のNodeSourceリポジトリ追加手順を記載
    - _Requirements: 2.1, 2.2, 2.3, 2.4_

- [x] 4. Kiro CLIインストールセクションを作成する
  - [x] 4.1 Kiro CLIのインストール手順と依存パッケージを記載する
    - npmによるKiro CLIインストールコマンドを記載
    - 動作に必要な依存パッケージを明記
    - インストール後のバージョン確認手順（`kiro --version`）を記載
    - パーミッションエラー（EACCES）発生時の対処方法を記載（npm prefix変更、nvm使用）
    - _Requirements: 3.1, 3.2, 3.3, 3.4_

- [x] 5. Checkpoint - 基本インストール確認
  - Ensure all tests pass, ask the user if questions arise.

- [x] 6. 認証・初期設定セクションを作成する
  - [x] 6.1 Kiro CLIの認証方法とデバイス認証フローを記載する
    - 認証に必要なアカウント情報の準備手順を記載
    - `kiro login` コマンドの実行手順を記載
    - ヘッドレス環境（EC2ターミナル）でのデバイス認証フロー（URL表示→ブラウザ認証）を詳述
    - 認証成功の確認方法を記載
    - 認証失敗時のトラブルシューティング（トークン期限切れ、ネットワークエラー等）を記載
    - _Requirements: 4.1, 4.2, 4.3, 4.4_

- [x] 7. IAMロール設定セクションを作成する
  - [x] 7.1 IAMロールの権限設定とアタッチ手順を記載する
    - Kiro CLIの利用に必要なIAMロール権限を明記
    - 最小権限の原則に基づいたポリシー設定例（JSON形式）を記載
    - EC2インスタンスへのIAMロールアタッチ手順を記載
    - `aws sts get-caller-identity` による確認手順を記載
    - IAMロール未設定時のアクセスキーによる代替認証方法を記載
    - _Requirements: 5.1, 5.2, 5.3, 5.4_

- [x] 8. GitLab連携設定セクションを作成する
  - [x] 8.1 Git/SSH設定とGitLabクローン手順を記載する
    - Gitクライアントのインストール確認・設定手順を記載
    - SSH鍵の生成手順（`ssh-keygen`）を記載
    - GitLab上でのSSH公開鍵登録手順を記載
    - SSH接続確認（`ssh -T git@gitlab.com`）手順を記載
    - GitLabプロジェクトのクローン手順（SSH URL使用）を記載
    - _Requirements: 6.4_

- [x] 9. 開発ワークフローセクションを作成する
  - [x] 9.1 Kiro CLIの基本操作と開発フローを記載する
    - Kiro CLIの主要コマンド一覧表を作成
    - `kiro init` によるプロジェクト初期化手順を記載
    - specの作成・管理方法（`kiro spec`）を記載
    - VS Code Remote SSH経由での基本的な開発フロー（spec作成→コード実装→Git操作→MR）を記載
    - _Requirements: 6.1, 6.2, 6.3, 6.4_

- [x] 10. Checkpoint - メインコンテンツ確認
  - Ensure all tests pass, ask the user if questions arise.

- [x] 11. トラブルシューティングセクションを作成する
  - [x] 11.1 一般的な問題と解決策を体系的に記載する
    - Node.jsバージョン不整合に関する問題と解決策を記載
    - Kiro CLI認証エラーに関する問題と解決策を記載
    - EC2インスタンスのリソース不足に関する問題と解決策を記載
    - AlmaLinux 9固有のSELinux/ファイアウォール関連の問題と解決策を記載
    - 各問題に対して「症状→原因→解決方法→確認コマンド」の統一フォーマットで記載
    - _Requirements: 7.1, 7.2, 7.3, 7.4_

- [x] 12. GitLab+Kiro統合ワークフローセクションを作成する
  - [x] 12.1 GitLabとKiro CLIを組み合わせた統合開発フローを記載する
    - Kiro specからGitLabのMR（マージリクエスト）作成までの一連のフローを記載
    - ブランチ戦略とKiro CLIの連携パターンを記載
    - 実際の開発シナリオに基づいたステップバイステップのワークフロー例を記載
    - _Requirements: 6.4_

- [x] 13. Final checkpoint - ガイドドキュメント完成確認
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- 本フィーチャーはドキュメント（ガイド）の作成であり、実行可能なコードを生成するものではない
- Property-Based Testingは適用しない（テスト対象の関数・アルゴリズムが存在しないため）
- ガイドの正しさは「内容の完全性」と「技術的正確性」で判断する
- 各タスクは依存関係に基づいた順序で実行する（前提条件→Node.js→Kiro CLI→認証→IAM→GitLab→ワークフロー→トラブルシューティング）
- Checkpointsで進捗確認を行い、段階的にドキュメントを完成させる

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1.1"] },
    { "id": 1, "tasks": ["2.1"] },
    { "id": 2, "tasks": ["3.1", "7.1"] },
    { "id": 3, "tasks": ["4.1"] },
    { "id": 4, "tasks": ["6.1", "8.1"] },
    { "id": 5, "tasks": ["9.1", "11.1"] },
    { "id": 6, "tasks": ["12.1"] }
  ]
}
```
