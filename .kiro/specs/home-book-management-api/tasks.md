# Implementation Plan: 家庭用蔵書管理REST API

## Overview

FastAPI + SQLAlchemy + JWT認証によるREST APIサーバーを段階的に実装する。プロジェクトセットアップから始め、コアインフラ（DB、認証）を構築し、各機能エリア（ヘルス、認証、ユーザー、書籍）を順に実装する。テストは各機能と並行して作成する。

## Tasks

- [ ] 1. プロジェクトセットアップと基盤構成
  - [-] 1.1 プロジェクト構造とパッケージ設定を作成する
    - `src/app/` 配下にディレクトリ構成を作成（models, schemas, routers, services, repositories, auth）
    - `pyproject.toml` を作成し、依存関係を定義（fastapi, uvicorn, sqlalchemy, pyjwt, bcrypt, pydantic[email], python-dotenv）
    - 開発用依存（pytest, hypothesis, httpx, pytest-asyncio, factory-boy）
    - 各ディレクトリに `__init__.py` を配置
    - _Requirements: 全体_

  - [~] 1.2 設定管理モジュールを作成する
    - `src/app/config.py`: Pydantic BaseSettingsを使用した環境変数管理
    - JWT_SECRET_KEY, JWT_ALGORITHM (HS256), ACCESS_TOKEN_EXPIRE_MINUTES (30), DATABASE_URL (sqlite:///./books.db) を定義
    - .env ファイルのテンプレートを作成
    - _Requirements: 10.5_

  - [~] 1.3 データベース接続・セッション管理を作成する
    - `src/app/database.py`: SQLAlchemy engine, SessionLocal, Base を定義
    - get_db 依存関係（FastAPI Depends用）を実装
    - SQLite用の設定（check_same_thread=False）
    - _Requirements: 全体_

  - [~] 1.4 カスタム例外とエラーハンドラを作成する
    - `src/app/exceptions.py`: AppException, UnauthorizedException, ForbiddenException, NotFoundException, ConflictException を定義
    - FastAPI exception_handler を登録して統一エラーレスポンス形式を実現
    - _Requirements: 全体_

- [ ] 2. データモデルとスキーマ定義
  - [~] 2.1 SQLAlchemy モデルを作成する
    - `src/app/models/user.py`: User モデル（id, email[unique, max 254], password_hash, created_at）
    - `src/app/models/book.py`: Book モデル（id, title[max 200], author[max 100], owner_id[FK → users.id, CASCADE DELETE], created_at）
    - `src/app/models/__init__.py` でモデルをエクスポート
    - _Requirements: 2.1, 5.1, 4.1_

  - [~] 2.2 Pydantic スキーマを作成する
    - `src/app/schemas/user.py`: UserCreate（email: EmailStr, max_length=254; password: str, min 8, max 72）、UserResponse（id, email）
    - `src/app/schemas/book.py`: BookCreate（title: str, min 1, max 200; author: str, min 1, max 100）、BookResponse（id, title, author, owner_id）
    - `src/app/schemas/auth.py`: LoginRequest（email: EmailStr, password: str）、TokenResponse（access_token, token_type="bearer"）
    - _Requirements: 2.1, 2.3, 5.1, 5.2, 8.1_

- [ ] 3. 認証インフラストラクチャ
  - [~] 3.1 パスワードハッシュモジュールを作成する
    - `src/app/auth/password.py`: hash_password(password) → bcrypt hash、verify_password(plain, hashed) → bool
    - bcrypt ライブラリを使用
    - _Requirements: 2.4_

  - [~] 3.2 JWT ハンドラを作成する
    - `src/app/auth/jwt_handler.py`: create_access_token(user_id, email) → JWT文字列、decode_token(token) → payload dict
    - ペイロード: {"sub": user_id, "email": email, "iat": ..., "exp": iat + 30分}
    - PyJWT を使用、HS256アルゴリズム
    - _Requirements: 8.1, 8.4, 10.5_

  - [~] 3.3 トークンブラックリストを作成する
    - `src/app/auth/blacklist.py`: TokenBlacklist クラス（インメモリ set で管理）
    - add(token, expires_at), is_blacklisted(token), cleanup_expired() メソッド
    - _Requirements: 9.1, 9.2_

  - [~] 3.4 認証依存関係を作成する
    - `src/app/auth/dependencies.py`: get_current_user 依存関係
    - OAuth2PasswordBearer スキームを使用
    - トークン検証 → ブラックリストチェック → ユーザー取得の流れ
    - 失敗時は適切な401エラーメッセージを返す
    - _Requirements: 10.1, 10.2, 10.3, 10.4_

- [~] 4. Checkpoint - 基盤確認
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 5. ヘルスチェックエンドポイント
  - [~] 5.1 ヘルスルーターを実装する
    - `src/app/routers/health.py`: GET /health → {"status": "ok"}
    - 認証不要
    - _Requirements: 1.1, 1.2_

  - [ ]* 5.2 ヘルスチェックの統合テストを書く
    - `src/tests/integration/test_health.py`: 200レスポンス確認、認証不要確認
    - _Requirements: 1.1, 1.2_

- [ ] 6. 認証エンドポイント（ログイン・ログアウト）
  - [~] 6.1 ユーザーリポジトリを作成する
    - `src/app/repositories/user_repository.py`: get_by_id, get_by_email, create, delete メソッド
    - _Requirements: 2.1, 3.1, 8.1_

  - [~] 6.2 認証サービスを実装する
    - `src/app/services/auth_service.py`: login(email, password) → TokenResponse, logout(token) → None
    - ログイン: メールアドレスでユーザー検索 → パスワード検証 → トークン生成
    - ログアウト: トークンをブラックリストに追加
    - 認証失敗時は "認証に失敗しました" メッセージ（情報非開示）
    - _Requirements: 8.1, 8.2, 9.1_

  - [~] 6.3 認証ルーターを実装する
    - `src/app/routers/auth.py`: POST /auth/login（認証不要）, POST /auth/logout（認証必要）
    - _Requirements: 8.1, 8.2, 8.3, 9.1, 9.3_

  - [ ]* 6.4 認証プロパティテストを書く
    - **Property 12: ログインラウンドトリップ**
    - **Property 13: ログイン失敗時の情報非開示**
    - **Property 14: トークン有効期限**
    - **Property 15: ログアウトによるトークン無効化**
    - **Property 16: 不正トークンの拒否**
    - **Validates: Requirements 8.1, 8.2, 8.4, 9.1, 9.2, 10.1, 10.2, 10.4, 10.5**

  - [ ]* 6.5 認証統合テストを書く
    - `src/tests/integration/test_auth_endpoints.py`
    - ログイン正常系/異常系、ログアウト正常系/異常系のE2Eフロー
    - _Requirements: 8.1, 8.2, 8.3, 9.1, 9.2, 9.3_

- [ ] 7. ユーザー管理エンドポイント
  - [~] 7.1 ユーザーサービスを実装する
    - `src/app/services/user_service.py`: create_user(email, password), get_user(user_id, requesting_user_id), delete_user(user_id, requesting_user_id)
    - メールアドレス重複チェック → 409、オーナーシップチェック → 403、存在チェック → 404
    - 削除時はユーザーと紐づく書籍をアトミックに削除
    - _Requirements: 2.1, 2.2, 3.1, 3.3, 4.1, 4.2_

  - [~] 7.2 ユーザールーターを実装する
    - `src/app/routers/users.py`: POST /users（認証必要）, GET /users/{user_id}（認証必要）, DELETE /users/{user_id}（認証必要）
    - _Requirements: 2.1, 2.2, 2.3, 2.5, 3.1, 3.2, 3.3, 3.4, 4.1, 4.2, 4.3, 4.4_

  - [ ]* 7.3 ユーザープロパティテストを書く
    - **Property 1: ユーザー登録ラウンドトリップ**
    - **Property 2: パスワードハッシュ化**
    - **Property 3: 無効なユーザー入力の拒否**
    - **Property 4: メールアドレスの一意性**
    - **Property 5: ユーザーリソースのオーナーシップ**
    - **Property 6: ユーザー削除のカスケード**
    - **Validates: Requirements 2.1, 2.2, 2.3, 2.4, 3.1, 3.3, 4.1, 4.2**

  - [ ]* 7.4 ユーザー統合テストを書く
    - `src/tests/integration/test_user_endpoints.py`
    - 登録・取得・削除の正常系/異常系のE2Eフロー
    - _Requirements: 2.1, 2.2, 2.3, 2.5, 3.1, 3.2, 3.3, 3.4, 4.1, 4.2, 4.3, 4.4_

- [~] 8. Checkpoint - 認証とユーザー管理確認
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 9. 書籍管理エンドポイント
  - [~] 9.1 書籍リポジトリを作成する
    - `src/app/repositories/book_repository.py`: create, get_by_id, get_by_owner_id, delete メソッド
    - _Requirements: 5.1, 6.1, 6.3, 7.1_

  - [~] 9.2 書籍サービスを実装する
    - `src/app/services/book_service.py`: create_book(title, author, owner_id), get_books(owner_id), get_book(book_id, owner_id), delete_book(book_id, owner_id)
    - 他ユーザーの書籍アクセスは404を返す（存在隠蔽）
    - 重複タイトル・著者の組み合わせを許容
    - _Requirements: 5.1, 5.4, 6.1, 6.3, 6.4, 7.1, 7.2_

  - [~] 9.3 書籍ルーターを実装する
    - `src/app/routers/books.py`: POST /books, GET /books, GET /books/{book_id}, DELETE /books/{book_id}（すべて認証必要）
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 6.1, 6.2, 6.3, 6.4, 6.5, 7.1, 7.2, 7.3_

  - [ ]* 9.4 書籍プロパティテストを書く
    - **Property 7: 書籍登録ラウンドトリップ**
    - **Property 8: 無効な書籍入力の拒否**
    - **Property 9: 書籍の重複許容**
    - **Property 10: 書籍リソースの隔離**
    - **Property 11: 書籍一覧の完全性**
    - **Validates: Requirements 5.1, 5.2, 5.4, 6.1, 6.2, 6.3, 6.4, 7.2**

  - [ ]* 9.5 書籍統合テストを書く
    - `src/tests/integration/test_book_endpoints.py`
    - 書籍登録・一覧・詳細・削除の正常系/異常系のE2Eフロー
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 6.1, 6.2, 6.3, 6.4, 6.5, 7.1, 7.2, 7.3_

- [ ] 10. アプリケーション統合と起動設定
  - [~] 10.1 FastAPIアプリケーションのメインモジュールを完成させる
    - `src/app/main.py`: FastAPIインスタンス作成、全ルーター登録、例外ハンドラ登録、起動時のDB初期化
    - _Requirements: 全体_

  - [~] 10.2 テスト共通フィクスチャを作成する
    - `src/tests/conftest.py`: TestClient作成、インメモリSQLiteセッション、テスト用JWT生成ヘルパー、DB自動リセット
    - `src/tests/factories.py`: UserFactory, BookFactory（factory_boy使用）
    - _Requirements: 全体_

- [~] 11. Final checkpoint - 全テスト実行
  - Ensure all tests pass, ask the user if questions arise.

## Task Dependency Graph

```json
{
  "waves": [
    {
      "wave": 1,
      "tasks": ["1"],
      "description": "プロジェクトセットアップと基盤構成"
    },
    {
      "wave": 2,
      "tasks": ["2", "3"],
      "description": "データモデル・スキーマ定義と認証インフラストラクチャ"
    },
    {
      "wave": 3,
      "tasks": ["4"],
      "description": "基盤確認チェックポイント"
    },
    {
      "wave": 4,
      "tasks": ["5", "6"],
      "description": "ヘルスチェックと認証エンドポイント"
    },
    {
      "wave": 5,
      "tasks": ["7"],
      "description": "ユーザー管理エンドポイント"
    },
    {
      "wave": 6,
      "tasks": ["8"],
      "description": "認証とユーザー管理確認チェックポイント"
    },
    {
      "wave": 7,
      "tasks": ["9"],
      "description": "書籍管理エンドポイント"
    },
    {
      "wave": 8,
      "tasks": ["10"],
      "description": "アプリケーション統合と起動設定"
    },
    {
      "wave": 9,
      "tasks": ["11"],
      "description": "最終チェックポイント - 全テスト実行"
    }
  ]
}
```

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP
- Each task references specific requirements for traceability
- Checkpoints ensure incremental validation
- Property tests validate universal correctness properties (Hypothesis, min 100 iterations)
- Unit tests validate specific examples and edge cases
- Source code is placed under `src/app/`, tests under `src/tests/`
- Python 3.11+ required
