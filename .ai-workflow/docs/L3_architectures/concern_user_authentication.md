---
doc:
  id: 'arch:user-authentication'
  title: 'ユーザー認証 基本設計'
  type: 'architecture'
  status: 'active'
  derives_from:
    - ref: 'spec:user-authentication'
      relation: 'implements'
  updated: '2026-04-08'
---

# ユーザー認証 基本設計

## 概要

メールアドレスとパスワードを用いたユーザー登録・ログイン機能の設計。
Web アプリとして `/register` および `/login` エンドポイントを提供し、認証成功後はセッションまたは JWT トークンによりログイン状態を維持する。

## 設計内容

### コンポーネント構成

```
[ブラウザ]
   │
   ▼
[Webサーバー / ルーティング]
   ├── GET  /register  → RegisterPage (登録フォーム)
   ├── POST /register  → AuthController#register
   ├── GET  /login     → LoginPage (ログインフォーム)
   └── POST /login     → AuthController#login
         │
         ▼
   [AuthService]
   ├── register(email, password): ユーザー作成・パスワードハッシュ化
   └── login(email, password):    照合・セッション/トークン発行
         │
         ▼
   [UserRepository]
   └── DB (usersテーブル)
```

### 処理フロー

#### ユーザー登録フロー

```mermaid
sequenceDiagram
    actor User
    participant Browser
    participant AuthController
    participant AuthService
    participant UserRepository
    participant DB

    User->>Browser: /register にアクセス
    Browser->>AuthController: GET /register
    AuthController-->>Browser: 登録フォーム表示

    User->>Browser: メールアドレス・パスワード入力 → 登録ボタン押下
    Browser->>AuthController: POST /register {email, password}
    AuthController->>AuthService: register(email, password)
    AuthService->>AuthService: メールアドレス形式バリデーション
    AuthService->>UserRepository: findByEmail(email)
    UserRepository->>DB: SELECT
    DB-->>UserRepository: 結果

    alt メールアドレス重複
        UserRepository-->>AuthService: 既存ユーザーあり
        AuthService-->>AuthController: エラー（重複）
        AuthController-->>Browser: エラーメッセージ表示
    else 新規登録可能
        AuthService->>AuthService: パスワードをハッシュ化 (bcrypt等)
        AuthService->>UserRepository: create(email, hashedPassword)
        UserRepository->>DB: INSERT
        DB-->>UserRepository: 完了
        AuthService-->>AuthController: 登録成功
        AuthController-->>Browser: 登録完了（ログイン画面へリダイレクト）
    end
```

#### ユーザーログインフロー

```mermaid
sequenceDiagram
    actor User
    participant Browser
    participant AuthController
    participant AuthService
    participant UserRepository
    participant DB
    participant Session

    User->>Browser: /login にアクセス
    Browser->>AuthController: GET /login
    AuthController-->>Browser: ログインフォーム表示

    User->>Browser: メールアドレス・パスワード入力 → ログインボタン押下
    Browser->>AuthController: POST /login {email, password}
    AuthController->>AuthService: login(email, password)
    AuthService->>UserRepository: findByEmail(email)
    UserRepository->>DB: SELECT
    DB-->>UserRepository: ユーザーレコード

    alt 認証失敗（ユーザー不存在 or パスワード不一致）
        AuthService-->>AuthController: 認証エラー
        AuthController-->>Browser: エラーメッセージ表示
    else 認証成功
        AuthService->>AuthService: パスワードハッシュ照合
        AuthService->>Session: セッション or JWT トークン発行
        AuthService-->>AuthController: 認証成功
        AuthController-->>Browser: ホーム画面へリダイレクト
    end
```

### データモデル

**users テーブル**

| カラム名       | 型           | 制約                  |
|----------------|--------------|---------------------- |
| id             | UUID / INT   | PK, AUTO INCREMENT    |
| email          | VARCHAR(255) | UNIQUE, NOT NULL      |
| password_hash  | VARCHAR(255) | NOT NULL              |
| created_at     | TIMESTAMP    | NOT NULL, DEFAULT NOW |
| updated_at     | TIMESTAMP    | NOT NULL, DEFAULT NOW |

## 設計上の決定事項

| 決定事項 | 内容 | 根拠 |
|----------|------|------|
| パスワードハッシュアルゴリズム | bcrypt を使用 | ソルト付きハッシュで総当たり攻撃に耐性があり、広く採用されている標準的手法 |
| 認証状態の維持方式 | セッション Cookie または JWT トークンを採用 | Web アプリとして HTTP ステートレス環境でログイン状態を維持するために必要。実装スタックに応じて選択 |
| メールアドレスバリデーション | サーバーサイドで正規表現によるフォーマットチェック | クライアントサイドのみでは回避可能なため、サーバー側での検証を必須とする |
| 重複メールアドレスのエラー処理 | 登録時に DB の UNIQUE 制約 + アプリ層の事前チェックで検出 | 競合状態を考慮し DB 制約を最終防衛ラインとする |
| 認証失敗メッセージ | 「メールアドレスまたはパスワードが正しくありません」と汎用化 | ユーザー存在有無を露出するユーザー列挙攻撃を防止するため |
