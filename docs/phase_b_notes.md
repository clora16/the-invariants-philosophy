# Phase B (Tool + Policy) 簡易実装メモ

このドキュメントは Phase B を「簡易実装」で完了扱いにする前提で、導入済みの機能を整理したものです。
厳密な運用・監査要件は Phase C/D 以降で強化する想定です。

## 1. ツール周り

- ツールレジストリ: `core` ではなく `modules/chat/application` に配置
- 承認フロー: `requires_approval` のツールに対して承認APIを用意
- ツール実行の失敗・タイムアウトでも会話を継続

主な実装:
- `backend/src/chat_original/modules/chat/application/tool_registry.py`
- `backend/src/chat_original/modules/chat/application/tool_executor.py`
- `backend/src/chat_original/modules/chat/infra/approval_store.py`

## 2. 認証（JWT）

- HS256 のみ対応
- `Authorization: Bearer <token>` を検証
- `auth_enabled=false` なら認証は無効化

設定:
```
AUTH_ENABLED=true
JWT_SECRET=your_secret
JWT_ISSUER=
JWT_AUDIENCE=
```

主な実装:
- `backend/src/chat_original/core/security/jwt_auth.py`

## 3. レート制限

- 固定ウィンドウ（1分単位）
- token があれば token 単位、なければ IP 単位
- `429` と `Retry-After` を返却

設定:
```
RATE_LIMIT_ENABLED=true
RATE_LIMIT_MAX_PER_MINUTE=120
```

主な実装:
- `backend/src/chat_original/core/security/rate_limit.py`

## 4. モデレーション（簡易）

`moderation_enabled=true` の場合:

- `moderation_use_claude=false` → ブロックリスト（キーワード一致）
- `moderation_use_claude=true` → Claude Haiku 判定
  - `SAFE` なら通過
  - `UNSAFE: <reason>` ならブロック
  - エラー時はフェールオープン（通過）

設定:
```
MODERATION_ENABLED=true
MODERATION_USE_CLAUDE=true
MODERATION_CLAUDE_MODEL=claude-haiku-4-5-20251001
MODERATION_BLOCKLIST=
MODERATION_RESPONSE=申し訳ありませんが、その内容には対応できません。
```

主な実装:
- `backend/src/chat_original/core/security/moderation.py`

## 5. ポリシーブロック（NGワード）

- モデレーションとは別枠
- 絶対NGワードのみブロックする用途

設定:
```
POLICY_ENABLED=true
POLICY_BLOCKLIST=違法,個人情報,ハラスメント
POLICY_RESPONSE=申し訳ありませんが、その内容には対応できません。
```

主な実装:
- `backend/src/chat_original/core/security/policy.py`
