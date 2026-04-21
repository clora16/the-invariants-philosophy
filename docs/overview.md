# Chat Original — システム概要・設定・仕組みガイド

## 目次

1. [システム概要](#1-システム概要)
2. [起動方法](#2-起動方法)
3. [設定リファレンス](#3-設定リファレンス)
4. [API リファレンス](#4-api-リファレンス)
5. [SSE ストリーミング仕様](#5-sse-ストリーミング仕様)
6. [セキュリティ機能](#6-セキュリティ機能)
7. [ツールシステム](#7-ツールシステム)
8. [アーキテクチャ](#8-アーキテクチャ)
9. [データモデル](#9-データモデル)
10. [フロントエンド](#10-フロントエンド)

---

## 1. システム概要

**Chat Original** は、Claude（Anthropic）を会話エンジンとして使う汎用チャットシステムです。  
案件固有のロジックを取り除いたクリーンな骨格として設計されており、ツール実行・ファイルアップロード・認証・モデレーションを標準搭載しています。

### 主な機能

| 機能 | 概要 |
|------|------|
| チャット | Claude とのリアルタイム SSE ストリーミング対話 |
| ツール実行 | 関数呼び出し（並列実行・承認フロー対応） |
| ファイルアップロード | セッションへの添付・参照 |
| JWT 認証 | Bearer トークン検証（HS256） |
| レート制限 | 固定ウィンドウ方式（IP / トークン単位） |
| モデレーション | キーワードマッチ or Claude Haiku による AI 判定 |
| ポリシーブロック | NGワード（絶対禁止用途） |

### 技術スタック

| レイヤー | 技術 |
|----------|------|
| バックエンド | Python 3.12 / FastAPI / uvicorn |
| AI | Anthropic Python SDK（バージョンは `backend/requirements.txt` / `backend/pyproject.toml` を参照） |
| フロントエンド | React 19 / TypeScript / Vite |
| 設定管理 | pydantic-settings（環境変数ベース） |

---

## 2. 起動方法

### Docker Compose（推奨）

```bash
# プロジェクトルートで実行
docker compose up -d

# クリーン再ビルド
docker compose down -v && docker compose up -d
```

| サービス | URL |
|---------|-----|
| フロントエンド | http://localhost:13000 |
| バックエンド | http://localhost:18000 |
| ヘルスチェック | http://localhost:18000/health |

### ローカル開発

**バックエンド**

```bash
cd backend
cp .env.example .env          # .env を編集して API キーを設定
uv sync
uv run python -m uvicorn chat_original.main:app --reload --host 0.0.0.0 --port 8000
```

**フロントエンド**

```bash
cd frontend
npm install
npm run dev                   # http://localhost:5173
```

> フロントエンドの Vite プロキシが `/api/*` を `BACKEND_URL`（デフォルト: `http://localhost:8000`）へ転送します。
> Docker Compose 利用時は `http://localhost:13000` でアクセスします。

### 最小セットアップ（API キーのみ）

```bash
# backend/.env
ANTHROPIC_API_KEY=sk-ant-...
```

API キー未設定の場合はダミー応答が返ります（動作確認には使えます）。

---

## 3. 設定リファレンス

`backend/.env` ファイル、または実環境の環境変数で設定します。

### 基本設定

| 環境変数 | デフォルト | 説明 |
|---------|-----------|------|
| `ANTHROPIC_API_KEY` | `""` | Anthropic API キー（必須） |
| `ANTHROPIC_MODEL` | `claude-sonnet-4-6` | 使用モデル |
| `MAX_OUTPUT_TOKENS` | `1024` | 最大出力トークン数 |
| `UPLOAD_ROOT` | `/tmp/chat_original_uploads` | ファイル保存ディレクトリ |

### 認証設定

| 環境変数 | デフォルト | 説明 |
|---------|-----------|------|
| `AUTH_ENABLED` | `false` | JWT 認証の有効化 |
| `JWT_SECRET` | `""` | JWT 署名鍵（認証有効時は必須） |
| `JWT_ISSUER` | `""` | JWT 発行者（省略時は検証しない） |
| `JWT_AUDIENCE` | `""` | JWT 対象者（省略時は検証しない） |

### レート制限設定

| 環境変数 | デフォルト | 説明 |
|---------|-----------|------|
| `RATE_LIMIT_ENABLED` | `false` | レート制限の有効化 |
| `RATE_LIMIT_MAX_PER_MINUTE` | `120` | 1分あたりの最大リクエスト数 |

### Reliability 設定（Phase C）

| 環境変数 | デフォルト | 説明 |
|---------|-----------|------|
| `SESSION_PERSIST_ENABLED` | `false` | セッション永続化（ファイル保存） |
| `SESSION_STORE_PATH` | `/tmp/chat_original_sessions` | 保存先ディレクトリ |
| `HISTORY_COMPACTION_ENABLED` | `false` | 履歴圧縮の有効化 |
| `HISTORY_COMPACTION_MAX_MESSAGES` | `80` | 圧縮発動の閾値 |
| `HISTORY_COMPACTION_KEEP_LAST` | `40` | 残す直近メッセージ数 |
| `HISTORY_COMPACTION_USE_SUMMARY` | `false` | 要約を挿入するか |
| `HISTORY_COMPACTION_SUMMARY_MODEL` | `claude-haiku-4-5-20251001` | 要約モデル |
| `HISTORY_COMPACTION_SUMMARY_MAX_TOKENS` | `256` | 要約最大トークン |
| `HISTORY_COMPACTION_SUMMARY_TIMEOUT_SEC` | `4.0` | 要約タイムアウト |
| `PROCESSED_REQUEST_IDS_MAX` | `1000` | 冪等IDの保持上限 |

### モデレーション設定

| 環境変数 | デフォルト | 説明 |
|---------|-----------|------|
| `MODERATION_ENABLED` | `false` | モデレーションの有効化 |
| `MODERATION_USE_CLAUDE` | `false` | `true` で Claude 判定、`false` でキーワードマッチ |
| `MODERATION_CLAUDE_MODEL` | `claude-haiku-4-5-20251001` | モデレーション用モデル |
| `MODERATION_BLOCKLIST` | `""` | ブロックワード（カンマ区切り）例: `爆弾,暴力` |
| `MODERATION_RESPONSE` | `申し訳ありませんが…` | ブロック時のアシスタント返答 |

### ポリシー設定

| 環境変数 | デフォルト | 説明 |
|---------|-----------|------|
| `POLICY_ENABLED` | `false` | ポリシーチェックの有効化 |
| `POLICY_BLOCKLIST` | `""` | 絶対NGワード（カンマ区切り） |
| `POLICY_RESPONSE` | `申し訳ありませんが…` | ブロック時のアシスタント返答 |

---

## 4. API リファレンス

すべてのエンドポイントに認証・レート制限が適用されます（設定で有効化した場合）。  
認証が必要な場合は `Authorization: Bearer <token>` ヘッダを付与してください。

### セッション管理

#### セッション作成
```
POST /api/chat/sessions
```

レスポンス: `SessionState`（ウェルカムメッセージ付き）

#### セッション状態取得
```
GET /api/chat/sessions/{session_id}
```

レスポンス: `SessionState`

#### セッションキャンセル
```
POST /api/chat/sessions/{session_id}/cancel
```

ストリーミング中のターンを中断します。

### メッセージ送信

```
POST /api/chat/sessions/{session_id}/messages
Content-Type: application/json

{ "content": "メッセージ内容" }
```

レスポンス: **Server-Sent Events（SSE）ストリーム**  
→ イベント仕様は「[5. SSE ストリーミング仕様](#5-sse-ストリーミング仕様)」を参照

#### 冪等・競合制御（Phase C）

`request_id` と `expected_revision` を指定できます。

```
POST /api/chat/sessions/{session_id}/messages
{
  "content": "メッセージ内容",
  "request_id": "uuid",
  "expected_revision": 12
}
```

- 同一 `request_id` は `duplicate_request` で拒否
- `expected_revision` が不一致なら `revision_conflict`（HTTP 409）

### ファイルアップロード

```
POST /api/chat/sessions/{session_id}/upload
Content-Type: multipart/form-data

file: <バイナリ>
```

レスポンス: `SessionState`（アシスタントの受領メッセージ付き）

### ツール承認・拒否

```
POST /api/chat/sessions/{session_id}/tools/{tool_call_id}/approve
POST /api/chat/sessions/{session_id}/tools/{tool_call_id}/deny
```

`requires_approval=True` のツールが呼び出されたとき、フロントエンドから承認または拒否します。

### ヘルスチェック

```
GET /health
```

認証・レート制限なし。`{ "status": "ok" }` を返します。

---

### SessionState スキーマ

```json
{
  "session_id": "chat_abc123",
  "status": "completed",
  "revision": 1,
  "messages": [
    {
      "id": "msg_xyz",
      "role": "assistant",
      "content": "こんにちは。",
      "created_at": "2026-01-01T00:00:00Z"
    }
  ],
  "uploaded_files": [],
  "error_code": null,
  "error_message": null
}
```

**status の値**

| 値 | 意味 |
|----|------|
| `idle` | 初期状態 |
| `calling_model` | モデル呼び出し中 |
| `streaming_assistant` | アシスタントの返答をストリーミング中 |
| `completed` | 正常完了 |
| `failed` | エラー終了（`error_code` に詳細） |
| `cancelled` | キャンセル済み |

---

## 5. SSE ストリーミング仕様

メッセージ送信エンドポイントは SSE（Server-Sent Events）形式でイベントを返します。

```
data: {"type": "assistant_delta", "text": "こんに"}\n\n
data: {"type": "assistant_delta", "text": "ちは"}\n\n
data: {"type": "message_stop"}\n\n
```

### イベント一覧

| `type` | フィールド | タイミング |
|--------|-----------|-----------|
| `assistant_delta` | `text: string` | アシスタントのテキストが届くたびに随時 |
| `tool_start` | `id: string`, `name: string` | ツール呼び出し開始時 |
| `tool_end` | `id: string`, `is_error: bool`, `content: string` | ツール実行完了時 |
| `tool_approval_required` | `id: string`, `name: string`, `input: object` | 承認待ちツール発見時（UI表示が必要） |
| `tool_denied` | `id: string`, `name: string` | ツールが拒否されたとき |
| `message_stop` | — | ストリーミング終了（正常・異常問わず必ず届く） |
| `error` | `code: string`, `message: string` | エラー発生時 |

### エラーコード一覧

| `code` | 原因 |
|--------|------|
| `policy_blocked` | ポリシーブロックリストに抵触 |
| `moderation_blocked` | モデレーションでブロック |
| `turn_cancelled` | `cancel` API でキャンセルされた |
| `max_iterations_reached` | ツールループが上限（10回）に達した |
| `model_failed` | Anthropic API 呼び出しエラー |
| `session_not_found` | セッションが存在しない |

### JavaScript での受信例

```typescript
const res = await fetch(`/api/chat/sessions/${sessionId}/messages`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ content }),
});

const reader = res.body!.getReader();
const decoder = new TextDecoder();
let buf = '';

while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  buf += decoder.decode(value, { stream: true });
  const lines = buf.split('\n\n');
  buf = lines.pop()!;
  for (const line of lines) {
    if (!line.startsWith('data: ')) continue;
    const event = JSON.parse(line.slice(6));
    if (event.type === 'assistant_delta') appendText(event.text);
    if (event.type === 'message_stop') break;
  }
}
```

---

## 6. セキュリティ機能

### 6.1 JWT 認証

`AUTH_ENABLED=true` にすると、すべての `/api/chat/*` エンドポイントでトークン検証が必須になります。

**対応アルゴリズム**: HS256 のみ

**検証項目**:
- 署名（HMAC-SHA256）
- 有効期限（`exp` クレーム）
- 有効開始時刻（`nbf` クレーム）
- 発行者（`iss` クレーム、`JWT_ISSUER` 設定時のみ）
- 対象者（`aud` クレーム、`JWT_AUDIENCE` 設定時のみ）

```bash
# 設定例
AUTH_ENABLED=true
JWT_SECRET=your-256-bit-secret
JWT_ISSUER=https://your-issuer.example.com
```

### 6.2 レート制限

**方式**: 固定ウィンドウ（60秒ごとにカウントリセット）

**キーの決定順序**:
1. `Authorization: Bearer <token>` がある場合 → `sha256(token)` でユーザー単位
2. なければ → クライアント IP アドレス単位

制限超過時は `HTTP 429` と `Retry-After` ヘッダを返します。

```bash
RATE_LIMIT_ENABLED=true
RATE_LIMIT_MAX_PER_MINUTE=60
```

### 6.3 モデレーション

ユーザー入力と最終回答の両方をチェックします。ブロックされるとアシスタントが拒否メッセージを返してセッションが `failed` になります。

#### キーワードマッチモード（デフォルト）

ブロックリストに含まれる単語を部分一致で検索します。低レイテンシ・確定的です。

```bash
MODERATION_ENABLED=true
MODERATION_USE_CLAUDE=false
MODERATION_BLOCKLIST=暴力,危険物,自傷
```

#### Claude 判定モード

Claude Haiku にコンテンツ判定を委譲します。文脈を理解した精密なチェックが可能です。

```
SAFE        → 通過
UNSAFE: <理由> → ブロック
エラー・タイムアウト → フェールオープン（通過）
```

```bash
MODERATION_ENABLED=true
MODERATION_USE_CLAUDE=true
MODERATION_CLAUDE_MODEL=claude-haiku-4-5-20251001
```

**Claude 判定の判断基準**:
- 暴力・危害・テロの煽動
- 差別・ヘイトスピーチ
- 違法行為の具体的な依頼・指示
- 性的に露骨・不適切なコンテンツ
- 自傷・自殺の方法の教示

### 6.4 ポリシーブロック

モデレーションより前段で評価される絶対NGワードです。モデレーションとは独立して設定できます。

```bash
POLICY_ENABLED=true
POLICY_BLOCKLIST=社外秘,個人情報,機密
POLICY_RESPONSE=その話題には対応できません。
```

### チェック順序

```
リクエスト受信
  ↓
① JWT 認証（auth_enabled=true の場合）
  ↓
② レート制限（rate_limit_enabled=true の場合）
  ↓
③ ポリシーチェック（policy_enabled=true の場合）
  ↓
④ モデレーション（moderation_enabled=true の場合）
  ↓
⑤ Anthropic API 呼び出し
  ↓
⑥ 最終回答にポリシー・モデレーションを再適用
```

---

## Phase C: Reliability 追加事項

### セッション永続化

`SESSION_PERSIST_ENABLED=true` で JSON ファイルにセッションを保存します。  
1セッション = 1ファイル（`{session_id}.json`）。

### 履歴圧縮

長い履歴は `HISTORY_COMPACTION_MAX_MESSAGES` 超過時に圧縮されます。  
`HISTORY_COMPACTION_USE_SUMMARY=true` の場合は「要約1件 + 直近N件」を保持します。

### 冪等性

同一 `request_id` の再送は拒否されます。  
保持件数は `PROCESSED_REQUEST_IDS_MAX` で上限管理します。

### 競合制御

`expected_revision` により楽観ロックを提供します。  
不一致は `revision_conflict`（HTTP 409）です。

---

## 7. ツールシステム

Claude にカスタム関数を実行させる仕組みです。

### 7.1 概要

- ツール定義は YAML で管理できる（`backend/settings.yaml`）
- 実行ハンドラは Python 側で名前に紐付ける
- `tool_policy` によって「許可されたツールだけ」を渡す

### 7.2 使い方

1. `backend/settings.yaml` で `tool_policy` を設定  
2. 必要なら `tools:` セクションを併記（将来の拡張用）  
3. Python 側でツール名に対するハンドラを登録

### 7.3 登録の仕方

**YAML（将来拡張用、現状は併記のみ）**

```yaml
tools:
  - name: FileRead
    description: プロジェクト内のファイルを読み込む
    permission: read
    requires_approval: false
    timeout_sec: 5
    input_schema:
      type: object
      properties:
        path:
          type: string
          description: ツールルートからの相対パス
      required: [path]
```

**Python 側の登録**

```python
from chat_original.modules.chat.domain.tools import ToolDefinition, ToolParameter
from chat_original.modules.chat.application.tool_registry import ToolRegistry
from chat_original.modules.chat.infra.tool_handlers import file_read_handler

registry = ToolRegistry()
registry.register(
    ToolDefinition(
        name="FileRead",
        description="プロジェクト内のファイルを読み込む",
        parameters=[
            ToolParameter(
                name="path",
                type="string",
                description="ツールルートからの相対パス",
                required=True,
            )
        ],
        permission="read",
        timeout_sec=5.0,
        requires_approval=False,
    ),
    handler=file_read_handler,
)
```

### 7.4 ツール定義と登録

`routes.py` でツールを定義して `ToolRegistry` に登録します。

```python
from chat_original.modules.chat.domain.tools import ToolDefinition, ToolParameter
from chat_original.modules.chat.application.tool_registry import ToolRegistry

registry = ToolRegistry()

registry.register(
    ToolDefinition(
        name="get_stock_price",
        description="指定した銘柄の現在の株価を取得します",
        parameters=[
            ToolParameter(
                name="symbol",
                type="string",
                description="銘柄コード（例: 7203）",
                required=True,
            )
        ],
        permission="read",       # "read" | "write" | "full"
        timeout_sec=10.0,
        requires_approval=False, # True にするとユーザー承認が必要
    ),
    handler=get_stock_price_handler,  # async def handler(input: dict) -> str
)
```

### 7.5 ハンドラの実装

```python
async def get_stock_price_handler(input: dict) -> str:
    symbol = input["symbol"]
    price = await fetch_price_from_api(symbol)
    return f"{symbol} の現在株価: {price} 円"
```

ハンドラは `async def` で、`str` を返します。例外を送出するとツール結果に `is_error=True` が設定されますが、**会話は継続**します。

### 7.6 ツール実行フロー

```
Claude が tool_use ブロックを返す
  ↓
requires_approval=True なら SSE で tool_approval_required イベントを送信
  ↓
フロントエンドがユーザーに承認/拒否を求める
  ↓
/approve or /deny エンドポイントを呼ぶ（60秒でタイムアウト→自動拒否）
  ↓
承認されたツールを asyncio.gather で並列実行
  ↓
結果を Claude に返して続きを生成
  ↓
（ツールがなくなるまで最大10回ループ）
```

### 7.7 権限レベル

| レベル | 用途 |
|--------|------|
| `read` | 読み取り専用（外部データ取得など） |
| `write` | 書き込みあり（DB更新、ファイル作成など） |
| `full` | 制限なし（システム操作など） |

セッションの権限レベル ≥ ツール定義の `permission` でないと実行できません。セッション権限の変更は `ChatSession.permission` フィールドで管理します。

### 7.8 デフォルト登録ツール

`routes.py` には動作確認用の2ツールが登録されています。

| ツール名 | 概要 | 承認 |
|---------|------|------|
| `get_current_time` | 指定タイムゾーンの現在時刻を返す | 不要 |
| `save_memo` | メモを保存する | **必要** |

---

## 8. アーキテクチャ

### レイヤー構成

```
backend/src/chat_original/
│
├── core/                          # アプリ横断の基盤
│   ├── config/settings.py         # 設定値（pydantic-settings）
│   └── security/                  # 認証・制限・モデレーション
│
└── modules/chat/                  # チャット機能モジュール
    ├── domain/                    # ビジネスロジックの核
    │   ├── models.py              # ChatSession, ChatMessage, UploadedFile
    │   ├── tools.py               # ToolDefinition, ToolCall, ToolResult
    │   ├── errors.py              # ChatError（エラーコード定義）
    │   └── repositories.py        # リポジトリプロトコル（インターフェース）
    │
    ├── application/               # ユースケース層
    │   ├── chat_service.py        # ストリーミング処理・ツールループ
    │   ├── tool_registry.py       # ツール登録・管理
    │   ├── tool_executor.py       # ツール実行（権限・タイムアウト）
    │   ├── ports.py               # ModelGateway インターフェース
    │   └── prompt.py              # システムプロンプト生成
    │
    ├── api/                       # HTTP インターフェース
    │   ├── routes.py              # FastAPI エンドポイント
    │   └── schemas.py             # Pydantic リクエスト/レスポンス
    │
    └── infra/                     # 外部システムとの接続
        ├── anthropic_gateway.py   # Anthropic API 実装
        ├── session_repository.py  # インメモリセッション保存
        └── approval_store.py      # ツール承認状態管理
```

**依存方向**: `api` → `application` → `domain` ← `infra`  
`infra` は `domain` のインターフェースを実装し、`application` から依存性注入で使われます。

### リポジトリパターン

セッションの永続化は `InMemoryChatSessionRepository` が担います。現在はメモリ内ですが、`ChatSessionRepository` プロトコルを実装すれば DB 等に差し替えられます。

```python
# mutator パターン — スレッドセーフな更新
session, result = repo.update(session_id, lambda s: s.touch())
```

---

## 9. データモデル

### ChatSession

```
session_id      ユニーク ID（例: chat_abc123）
status          セッション状態（idle / calling_model / streaming_assistant / completed / failed / cancelled）
messages        会話履歴（ChatMessage のリスト）
uploaded_files  添付ファイル一覧（UploadedFile のリスト）
error_code      最後のエラーコード（正常時は null）
error_message   エラー詳細（正常時は null）
created_at      作成日時
updated_at      最終更新日時
```

### ChatMessage

```
id              メッセージ ID
role            user / assistant / system / tool
content         テキスト内容
structured_content  tool_use / tool_result ブロック（Anthropic API 形式）
created_at      作成日時
```

### UploadedFile

```
name            ファイル名
size            バイト数
content_type    MIME タイプ
storage_path    サーバー上のパス
uploaded_at     アップロード日時
```

---

## 10. フロントエンド

### 環境変数（`frontend/.env`）

| 変数名 | デフォルト | 説明 |
|--------|-----------|------|
| `VITE_API_BASE` | `""` | API ベースURL（空のとき Vite プロキシ経由） |
| `VITE_API_TOKEN` | `""` | 認証トークン（設定時に Bearer ヘッダを自動付与） |
| `BACKEND_URL` | `http://localhost:8000` | Vite プロキシの転送先 |

### ディレクトリ構成

```
frontend/src/
├── features/chat/
│   ├── api/chatApi.ts        # API クライアント（SSE 処理含む）
│   ├── components/
│   │   └── ChatPanel.tsx     # メイン UI
│   └── types/chat.ts         # TypeScript 型定義
└── main.tsx                  # エントリーポイント
```

### ChatPanel が扱う状態

| 状態 | 型 | 説明 |
|------|-----|------|
| `sessionId` | `string \| null` | 現在のセッション ID |
| `messages` | `MessageOut[]` | 表示中のメッセージ一覧 |
| `isStreaming` | `boolean` | ストリーミング中フラグ |

### 認証トークンの設定

```bash
# frontend/.env
VITE_API_TOKEN=eyJhbGciOiJIUzI1NiJ9...
```

設定すると全リクエストに `Authorization: Bearer <token>` が自動で付与されます。

---

## 付録: よくある設定パターン

### パターン A: 開発環境（全機能オフ）

```bash
ANTHROPIC_API_KEY=sk-ant-...
```

### パターン B: 認証あり・モデレーションあり

```bash
ANTHROPIC_API_KEY=sk-ant-...
AUTH_ENABLED=true
JWT_SECRET=your-secret-key-minimum-32-chars
RATE_LIMIT_ENABLED=true
RATE_LIMIT_MAX_PER_MINUTE=60
MODERATION_ENABLED=true
MODERATION_USE_CLAUDE=true
```

### パターン C: NGワードのみブロック（最小構成）

```bash
ANTHROPIC_API_KEY=sk-ant-...
POLICY_ENABLED=true
POLICY_BLOCKLIST=機密,社外秘,個人情報
```

### パターン D: 全機能有効（本番想定）

```bash
ANTHROPIC_API_KEY=sk-ant-...
AUTH_ENABLED=true
JWT_SECRET=your-256-bit-secret
JWT_ISSUER=https://auth.example.com
RATE_LIMIT_ENABLED=true
RATE_LIMIT_MAX_PER_MINUTE=120
MODERATION_ENABLED=true
MODERATION_USE_CLAUDE=true
POLICY_ENABLED=true
POLICY_BLOCKLIST=機密,社外秘
```
