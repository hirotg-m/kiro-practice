# 家庭用蔵書管理 REST API 仕様書

## 基本情報

| 項目 | 値 |
|------|-----|
| ベースURL | `http://localhost:8000` |
| プロトコル | HTTP/HTTPS |
| データ形式 | JSON |
| 認証方式 | Bearer Token（JWT） |
| フレームワーク | FastAPI (Python) |

---

## 認証

保護エンドポイントへのアクセスには、`Authorization` ヘッダーに Bearer トークンを含める必要があります。

```
Authorization: Bearer <access_token>
```

- トークンはログイン成功時に発行される
- 有効期限: 発行から30分
- トークン形式: JWT（JSON Web Token）

---

## エンドポイント一覧

| メソッド | パス | 認証 | 説明 |
|---------|------|:----:|------|
| GET | /health | 不要 | ヘルスチェック |
| POST | /users | 必要 | ユーザー登録 |
| GET | /users/{user_id} | 必要 | ユーザー情報取得 |
| DELETE | /users/{user_id} | 必要 | ユーザー削除 |
| POST | /auth/login | 不要 | ログイン |
| POST | /auth/logout | 必要 | ログアウト |
| POST | /books | 必要 | 書籍登録 |
| GET | /books | 必要 | 書籍一覧取得 |
| GET | /books/{book_id} | 必要 | 書籍詳細取得 |
| DELETE | /books/{book_id} | 必要 | 書籍削除 |

---

## エンドポイント詳細

---

### GET /health

ヘルスチェック。APIサーバーの稼働状態を確認する。

#### リクエスト

認証不要。リクエストボディなし。

#### curl 例

```bash
curl http://localhost:8000/health
```

#### レスポンス

**200 OK**

```json
{
  "status": "ok"
}
```

#### エラー

このエンドポイントはエラーを返さない（サーバーが応答可能な限り200を返す）。

---

### POST /users

新規ユーザーを登録する。

#### リクエスト

認証必要。

#### curl 例

```bash
curl -X POST http://localhost:8000/users \
  -H "Authorization: Bearer <access_token>" \
  -H "Content-Type: application/json" \
  -d '{"email": "tanaka@example.com", "password": "securepass123"}'
```

**リクエストボディ:**

```json
{
  "email": "string",
  "password": "string"
}
```

| フィールド | 型 | 必須 | バリデーション |
|-----------|-----|:----:|-------------|
| email | string | ✓ | 有効なメールアドレス形式（最大254文字） |
| password | string | ✓ | 8〜72文字 |

#### レスポンス

**201 Created**

```json
{
  "id": 1,
  "email": "tanaka@example.com"
}
```

#### エラー

| ステータス | 条件 | レスポンス例 |
|-----------|------|-------------|
| 401 Unauthorized | Authorization ヘッダーが未指定 | `{"detail": "認証が必要です"}` |
| 401 Unauthorized | トークンが無効、改ざん済み、または期限切れ | `{"detail": "トークンが無効です"}` |
| 409 Conflict | 指定したメールアドレスが既に登録されている | `{"detail": "このメールアドレスは既に使用されています"}` |
| 422 Unprocessable Entity | email が有効なメールアドレス形式でない | `{"detail": [{"loc": ["body", "email"], "msg": "...", "type": "..."}]}` |
| 422 Unprocessable Entity | email が254文字を超過している | 同上 |
| 422 Unprocessable Entity | password が8文字未満または72文字超過 | 同上 |
| 422 Unprocessable Entity | email または password が欠落している | 同上 |

---

### GET /users/{user_id}

指定したユーザーの情報を取得する。

#### リクエスト

認証必要。リクエストボディなし。

#### curl 例

```bash
curl http://localhost:8000/users/1 \
  -H "Authorization: Bearer <access_token>"
```

**パスパラメータ:**

| パラメータ | 型 | 説明 |
|-----------|-----|------|
| user_id | int | 取得するユーザーのID |

#### レスポンス

**200 OK**

```json
{
  "id": 1,
  "email": "tanaka@example.com"
}
```

#### エラー

| ステータス | 条件 | レスポンス例 |
|-----------|------|-------------|
| 401 Unauthorized | Authorization ヘッダーが未指定 | `{"detail": "認証が必要です"}` |
| 401 Unauthorized | トークンが無効、改ざん済み、または期限切れ | `{"detail": "トークンが無効です"}` |
| 403 Forbidden | 認証済みユーザーが他のユーザーの情報を取得しようとした | `{"detail": "このリソースへのアクセス権限がありません"}` |
| 404 Not Found | 指定した user_id のユーザーが存在しない | `{"detail": "ユーザーが見つかりません"}` |

---

### DELETE /users/{user_id}

指定したユーザーと、そのユーザーに紐づく全書籍を削除する。

#### リクエスト

認証必要。リクエストボディなし。

#### curl 例

```bash
curl -X DELETE http://localhost:8000/users/1 \
  -H "Authorization: Bearer <access_token>"
```

**パスパラメータ:**

| パラメータ | 型 | 説明 |
|-----------|-----|------|
| user_id | int | 削除するユーザーのID |

#### レスポンス

**200 OK**

```json
{
  "detail": "ユーザーを削除しました"
}
```

#### エラー

| ステータス | 条件 | レスポンス例 |
|-----------|------|-------------|
| 401 Unauthorized | Authorization ヘッダーが未指定 | `{"detail": "認証が必要です"}` |
| 401 Unauthorized | トークンが無効、改ざん済み、または期限切れ | `{"detail": "トークンが無効です"}` |
| 403 Forbidden | 認証済みユーザーが他のユーザーを削除しようとした | `{"detail": "このリソースへのアクセス権限がありません"}` |
| 404 Not Found | 指定した user_id のユーザーが存在しない | `{"detail": "ユーザーが見つかりません"}` |

#### 備考

- ユーザー削除時、そのユーザーに紐づくすべての書籍レコードがアトミックに削除される

---

### POST /auth/login

ユーザー認証を行い、アクセストークンを発行する。

#### リクエスト

認証不要。

#### curl 例

```bash
curl -X POST http://localhost:8000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "tanaka@example.com", "password": "securepass123"}'
```

**リクエストボディ:**

```json
{
  "email": "string",
  "password": "string"
}
```

| フィールド | 型 | 必須 | 説明 |
|-----------|-----|:----:|------|
| email | string | ✓ | 登録済みメールアドレス |
| password | string | ✓ | パスワード |

#### レスポンス

**200 OK**

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIs...",
  "token_type": "bearer"
}
```

| フィールド | 型 | 説明 |
|-----------|-----|------|
| access_token | string | JWT アクセストークン（有効期限30分） |
| token_type | string | 常に `"bearer"` |

#### エラー

| ステータス | 条件 | レスポンス例 |
|-----------|------|-------------|
| 401 Unauthorized | メールアドレスが存在しない | `{"detail": "認証に失敗しました"}` |
| 401 Unauthorized | パスワードが不正 | `{"detail": "認証に失敗しました"}` |
| 422 Unprocessable Entity | email または password が欠落している | `{"detail": [{"loc": ["body", "email"], "msg": "...", "type": "..."}]}` |

#### 備考

- セキュリティ上、メールアドレスとパスワードのどちらが不正かは応答から判別できない
- 同一のエラーメッセージを返すことでタイミング攻撃を緩和する

---

### POST /auth/logout

現在のアクセストークンを無効化し、セッションを終了する。

#### リクエスト

認証必要。リクエストボディなし。

#### curl 例

```bash
curl -X POST http://localhost:8000/auth/logout \
  -H "Authorization: Bearer <access_token>"
```

#### レスポンス

**200 OK**

```json
{
  "detail": "ログアウトしました"
}
```

#### エラー

| ステータス | 条件 | レスポンス例 |
|-----------|------|-------------|
| 401 Unauthorized | Authorization ヘッダーが未指定 | `{"detail": "認証が必要です"}` |
| 401 Unauthorized | トークンが無効、改ざん済み、期限切れ、または既にログアウト済み | `{"detail": "トークンが無効です"}` |

#### 備考

- ログアウト後、同じトークンで保護エンドポイントにアクセスすると401が返る
- トークンはブラックリスト方式で無効化される

---

### POST /books

認証済みユーザーの蔵書に書籍を追加する。

#### リクエスト

認証必要。

#### curl 例

```bash
curl -X POST http://localhost:8000/books \
  -H "Authorization: Bearer <access_token>" \
  -H "Content-Type: application/json" \
  -d '{"title": "吾輩は猫である", "author": "夏目漱石"}'
```

**リクエストボディ:**

```json
{
  "title": "string",
  "author": "string"
}
```

| フィールド | 型 | 必須 | バリデーション |
|-----------|-----|:----:|-------------|
| title | string | ✓ | 1〜200文字 |
| author | string | ✓ | 1〜100文字 |

#### レスポンス

**201 Created**

```json
{
  "id": 1,
  "title": "吾輩は猫である",
  "author": "夏目漱石",
  "owner_id": 1
}
```

#### エラー

| ステータス | 条件 | レスポンス例 |
|-----------|------|-------------|
| 401 Unauthorized | Authorization ヘッダーが未指定 | `{"detail": "認証が必要です"}` |
| 401 Unauthorized | トークンが無効、改ざん済み、または期限切れ | `{"detail": "トークンが無効です"}` |
| 422 Unprocessable Entity | title が空文字または200文字超過 | `{"detail": [{"loc": ["body", "title"], "msg": "...", "type": "..."}]}` |
| 422 Unprocessable Entity | author が空文字または100文字超過 | `{"detail": [{"loc": ["body", "author"], "msg": "...", "type": "..."}]}` |
| 422 Unprocessable Entity | title または author が欠落している | 同上 |

#### 備考

- 同一タイトル・同一著者の書籍を重複登録することは許容される（別レコードとして作成）
- 書籍は認証済みユーザーに自動的に紐づけられる

---

### GET /books

認証済みユーザーの全蔵書を一覧取得する。

#### リクエスト

認証必要。リクエストボディなし。

#### curl 例

```bash
curl http://localhost:8000/books \
  -H "Authorization: Bearer <access_token>"
```

#### レスポンス

**200 OK**

```json
[
  {
    "id": 1,
    "title": "吾輩は猫である",
    "author": "夏目漱石",
    "owner_id": 1
  },
  {
    "id": 2,
    "title": "坊っちゃん",
    "author": "夏目漱石",
    "owner_id": 1
  }
]
```

書籍が0件の場合:

```json
[]
```

#### エラー

| ステータス | 条件 | レスポンス例 |
|-----------|------|-------------|
| 401 Unauthorized | Authorization ヘッダーが未指定 | `{"detail": "認証が必要です"}` |
| 401 Unauthorized | トークンが無効、改ざん済み、または期限切れ | `{"detail": "トークンが無効です"}` |

#### 備考

- 他のユーザーの書籍は一切返されない（自分の書籍のみ）

---

### GET /books/{book_id}

指定した書籍の詳細情報を取得する。

#### リクエスト

認証必要。リクエストボディなし。

#### curl 例

```bash
curl http://localhost:8000/books/1 \
  -H "Authorization: Bearer <access_token>"
```

**パスパラメータ:**

| パラメータ | 型 | 説明 |
|-----------|-----|------|
| book_id | int | 取得する書籍のID |

#### レスポンス

**200 OK**

```json
{
  "id": 1,
  "title": "吾輩は猫である",
  "author": "夏目漱石",
  "owner_id": 1
}
```

#### エラー

| ステータス | 条件 | レスポンス例 |
|-----------|------|-------------|
| 401 Unauthorized | Authorization ヘッダーが未指定 | `{"detail": "認証が必要です"}` |
| 401 Unauthorized | トークンが無効、改ざん済み、または期限切れ | `{"detail": "トークンが無効です"}` |
| 404 Not Found | 指定した book_id の書籍が存在しない | `{"detail": "書籍が見つかりません"}` |
| 404 Not Found | 指定した book_id の書籍が他のユーザーに属する | `{"detail": "書籍が見つかりません"}` |

#### 備考

- 他のユーザーの書籍にアクセスした場合、存在自体を隠蔽するため403ではなく404を返す

---

### DELETE /books/{book_id}

指定した書籍を蔵書から削除する。

#### リクエスト

認証必要。リクエストボディなし。

#### curl 例

```bash
curl -X DELETE http://localhost:8000/books/1 \
  -H "Authorization: Bearer <access_token>"
```

**パスパラメータ:**

| パラメータ | 型 | 説明 |
|-----------|-----|------|
| book_id | int | 削除する書籍のID |

#### レスポンス

**200 OK**

```json
{
  "detail": "書籍を削除しました"
}
```

#### エラー

| ステータス | 条件 | レスポンス例 |
|-----------|------|-------------|
| 401 Unauthorized | Authorization ヘッダーが未指定 | `{"detail": "認証が必要です"}` |
| 401 Unauthorized | トークンが無効、改ざん済み、または期限切れ | `{"detail": "トークンが無効です"}` |
| 404 Not Found | 指定した book_id の書籍が存在しない | `{"detail": "書籍が見つかりません"}` |
| 404 Not Found | 指定した book_id の書籍が他のユーザーに属する | `{"detail": "書籍が見つかりません"}` |

#### 備考

- 他のユーザーの書籍を削除しようとした場合も404を返す（存在を隠蔽）

---

## エラーコード一覧

### 全エラーコードと発生条件

| ステータスコード | 意味 | 発生条件 |
|----------------|------|---------|
| **401 Unauthorized** | 認証エラー | 下記参照 |
| **403 Forbidden** | アクセス権限なし | 下記参照 |
| **404 Not Found** | リソース未検出 | 下記参照 |
| **409 Conflict** | リソース競合 | 下記参照 |
| **422 Unprocessable Entity** | バリデーションエラー | 下記参照 |

---

### 401 Unauthorized — 発生条件一覧

保護エンドポイント = GET /health, POST /auth/login **以外** の全エンドポイント

| 条件 | 対象エンドポイント | メッセージ |
|------|-------------------|-----------|
| Authorization ヘッダーが未指定 | 全保護エンドポイント | 認証が必要です |
| Bearer トークンが空 | 全保護エンドポイント | 認証が必要です |
| トークンの署名が改ざんされている | 全保護エンドポイント | トークンが無効です |
| トークンの形式が不正（JWT として解析不能） | 全保護エンドポイント | トークンが無効です |
| トークンの有効期限（exp）が切れている | 全保護エンドポイント | トークンの有効期限が切れています |
| トークンがログアウトにより無効化されている（ブラックリスト） | 全保護エンドポイント | トークンが無効です |
| ログイン時にメールアドレスが存在しない | POST /auth/login | 認証に失敗しました |
| ログイン時にパスワードが不正 | POST /auth/login | 認証に失敗しました |

---

### 403 Forbidden — 発生条件一覧

| 条件 | 対象エンドポイント | メッセージ |
|------|-------------------|-----------|
| 認証済みユーザーが他のユーザーの情報を取得しようとした | GET /users/{user_id} | このリソースへのアクセス権限がありません |
| 認証済みユーザーが他のユーザーを削除しようとした | DELETE /users/{user_id} | このリソースへのアクセス権限がありません |

---

### 404 Not Found — 発生条件一覧

| 条件 | 対象エンドポイント | メッセージ |
|------|-------------------|-----------|
| 指定した user_id のユーザーが存在しない | GET /users/{user_id} | ユーザーが見つかりません |
| 指定した user_id のユーザーが存在しない | DELETE /users/{user_id} | ユーザーが見つかりません |
| 指定した book_id の書籍が存在しない | GET /books/{book_id} | 書籍が見つかりません |
| 指定した book_id の書籍が他のユーザーに属する | GET /books/{book_id} | 書籍が見つかりません |
| 指定した book_id の書籍が存在しない | DELETE /books/{book_id} | 書籍が見つかりません |
| 指定した book_id の書籍が他のユーザーに属する | DELETE /books/{book_id} | 書籍が見つかりません |

---

### 409 Conflict — 発生条件一覧

| 条件 | 対象エンドポイント | メッセージ |
|------|-------------------|-----------|
| 指定したメールアドレスが既に登録されている | POST /users | このメールアドレスは既に使用されています |

---

### 422 Unprocessable Entity — 発生条件一覧

| 条件 | 対象エンドポイント |
|------|-------------------|
| email が欠落 | POST /users |
| email が有効なメールアドレス形式でない | POST /users |
| email が254文字を超過している | POST /users |
| password が欠落 | POST /users |
| password が8文字未満 | POST /users |
| password が72文字超過 | POST /users |
| email が欠落 | POST /auth/login |
| password が欠落 | POST /auth/login |
| title が欠落 | POST /books |
| title が空文字 | POST /books |
| title が200文字超過 | POST /books |
| author が欠落 | POST /books |
| author が空文字 | POST /books |
| author が100文字超過 | POST /books |

---

## エラーレスポンス形式

### 一般エラー

```json
{
  "detail": "エラーメッセージ"
}
```

### バリデーションエラー（422）

FastAPI 標準形式:

```json
{
  "detail": [
    {
      "loc": ["body", "フィールド名"],
      "msg": "エラー内容の説明",
      "type": "エラータイプ"
    }
  ]
}
```

**例: email が不正な形式の場合**

```json
{
  "detail": [
    {
      "loc": ["body", "email"],
      "msg": "value is not a valid email address",
      "type": "value_error"
    }
  ]
}
```

---

## データモデル

### User

| フィールド | 型 | 説明 |
|-----------|-----|------|
| id | int | ユーザーID（自動採番） |
| email | string | メールアドレス（一意、最大254文字） |
| password_hash | string | bcrypt ハッシュ化パスワード（APIレスポンスには含まれない） |
| created_at | datetime | 作成日時 |

### Book

| フィールド | 型 | 説明 |
|-----------|-----|------|
| id | int | 書籍ID（自動採番） |
| title | string | タイトル |
| author | string | 著者 |
| owner_id | int | 所有ユーザーID（外部キー: users.id） |
| created_at | datetime | 作成日時 |

---

## セキュリティ設計

| 項目 | 方針 |
|------|------|
| パスワード保存 | bcrypt ハッシュ化（平文は一切保存・返却しない） |
| 認証失敗メッセージ | メールアドレス/パスワードのどちらが不正かを明かさない |
| 他ユーザーの書籍アクセス | 404 を返し、リソースの存在自体を隠蔽する |
| トークン無効化 | ブラックリスト方式でログアウト後のトークン使用を防止 |
| トークン有効期限 | 発行から30分 |
| トークン送受信 | Authorization ヘッダー（Bearer スキーム） |

---

## curl による操作フロー例

一連の操作を curl で実行する例です。

```bash
# 1. ヘルスチェック
curl http://localhost:8000/health

# 2. ログイン（トークン取得）※初回ユーザーは管理者が事前作成済みとする
TOKEN=$(curl -s -X POST http://localhost:8000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "admin@example.com", "password": "adminpass123"}' | jq -r '.access_token')

# 3. ユーザー登録（認証必要）
curl -X POST http://localhost:8000/users \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"email": "tanaka@example.com", "password": "mypassword123"}'

# 4. ユーザー情報取得
curl http://localhost:8000/users/1 \
  -H "Authorization: Bearer $TOKEN"

# 5. 書籍登録
curl -X POST http://localhost:8000/books \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"title": "吾輩は猫である", "author": "夏目漱石"}'

# 6. 書籍一覧取得
curl http://localhost:8000/books \
  -H "Authorization: Bearer $TOKEN"

# 7. 書籍詳細取得
curl http://localhost:8000/books/1 \
  -H "Authorization: Bearer $TOKEN"

# 8. 書籍削除
curl -X DELETE http://localhost:8000/books/1 \
  -H "Authorization: Bearer $TOKEN"

# 9. ログアウト
curl -X POST http://localhost:8000/auth/logout \
  -H "Authorization: Bearer $TOKEN"

# 10. ログアウト後のアクセス確認（401エラーになる）
curl http://localhost:8000/books \
  -H "Authorization: Bearer $TOKEN"
```
