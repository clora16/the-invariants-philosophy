# Capability Standard: Chat System

## Purpose

この文書は、汎用的な chat system が最低限満たすべき capability を定義する。案件固有の UI 仕様や顧客制約は対象外とし、複数アプリケーションで再利用可能な対話システム基盤の基準を定義する。

## Scope

対象:

1. 1対1のマルチターン対話
2. ストリーミング応答
3. ツール呼び出しを含むエージェント対話
4. セッション永続化、再開、履歴圧縮
5. RAG 検索結果の注入と出典追跡
6. 運用監視、ログ、メトリクス、トレース

対象外:

1. 音声認識 / 音声合成エンジン自体の開発
2. ベクトル DB の製品依存チューニング詳細
3. 個別プロダクト UI の画面設計
4. 案件ごとの rollout 計画、マイルストーン、タスク分解

## Design Principles

1. 分岐中心ではなくイベント駆動で設計する。
2. 対話制御、ツール実行、表示、ポリシーを分離する。
3. モデル品質はプロンプトと文脈設計で担保し、業務分岐で過剰吸収しない。
4. 失敗を前提に、停止理由と再開可能性を常に保持する。

## Logical Architecture

最低限、次の責務を分離して持つ。

1. `Channel Adapter`
   Web / CLI / モバイル向けの入出力変換
2. `Conversation Orchestrator`
   ターン実行と状態遷移管理
3. `Prompt Composer`
   システム指示、履歴、RAG 結果の合成
4. `Model Gateway`
   LLM API 統合とストリーム処理
5. `Tool Runtime`
   ツール実行、タイムアウト、再試行
6. `Policy Engine`
   権限判定、承認フロー、拒否理由生成
7. `Memory & Retrieval`
   会話履歴、長期記憶、検索結果管理
8. `Session Store`
   セッション永続化と復元
9. `Observability`
   ログ、メトリクス、トレース集約

## State Machine

最低限、次の状態を扱えること。

- `idle`
- `receiving_input`
- `calling_model`
- `streaming_assistant`
- `awaiting_tool_execution`
- `awaiting_user_approval`
- `applying_tool_result`
- `completed`
- `failed`
- `cancelled`

## Functional Requirements

| ID | Requirement | Priority | Acceptance Baseline |
|---|---|---|---|
| FR-001 | セッション作成が可能であること | MUST | `session_id` が返却される |
| FR-002 | セッション再開が可能であること | MUST | 過去メッセージが復元される |
| FR-003 | ユーザー発話を時系列で保持すること | MUST | 順序が不変で保存される |
| FR-004 | アシスタント応答をストリーミング配信できること | MUST | delta 単位で受信できる |
| FR-005 | 1ターン内で複数ツール呼び出しを処理できること | MUST | 複数 `tool_use` を処理可能 |
| FR-006 | ツール結果を会話へ再注入できること | MUST | `tool_result` が次推論に反映される |
| FR-007 | ツール不要ターンは通常応答で終了できること | MUST | 無限ループしない |
| FR-008 | ターン反復回数上限を設定可能であること | MUST | 上限超過で停止理由を返す |
| FR-009 | トークン予算上限を設定可能であること | MUST | 予算超過で安全停止する |
| FR-010 | システムプロンプトを差し替え可能であること | MUST | 設定変更で反映される |
| FR-011 | 履歴圧縮を実行できること | SHOULD | 履歴サイズを削減できる |
| FR-012 | 権限モードを保持すること | MUST | ツール要件と照合される |
| FR-013 | 承認必須ツールを実行前確認できること | MUST | 承認 UI / API が呼ばれる |
| FR-014 | 拒否理由をモデルへ返却できること | MUST | 拒否理由付き `tool_result` を注入 |
| FR-015 | pre / post hook を実行できること | SHOULD | hook 結果を実行可否へ反映 |
| FR-016 | ツール実行タイムアウトを設定可能であること | MUST | タイムアウト時に失敗として扱う |
| FR-017 | ツール実行内容を監査ログへ記録できること | SHOULD | `tool/input/output/error` を保存 |
| FR-018 | ツール失敗時も会話を継続可能であること | MUST | セッション破損が発生しない |
| FR-019 | RAG 結果を構造化注入できること | MUST | `retrieval_context` を注入可能 |
| FR-020 | 出典情報を保持できること | MUST | source id / url / chunk id を保持 |
| FR-021 | 検索ヒットなし時のフォールバックがあること | MUST | 規定応答を返却 |
| FR-022 | 複数チャネルで同一セッション継続可能であること | SHOULD | Web → CLI 継続などに対応 |
| FR-023 | セッションエクスポート可能であること | SHOULD | JSON / Markdown で出力できる |
| FR-024 | モデル切替をセッション継続で行えること | SHOULD | 中断せず切替できる |
| FR-025 | 実行中ターンをキャンセルできること | MUST | キャンセル状態へ遷移 |
| FR-026 | 冪等再送制御を持つこと | SHOULD | 重複要求を二重実行しない |
| FR-027 | 機械可読なエラー分類コードを返すこと | MUST | 例: `policy_denied` |
| FR-028 | プラグインツールを動的ロードできること | SHOULD | 再起動不要で反映可能 |
| FR-029 | 管理者向け実行トレースを参照できること | SHOULD | 1ターンの全イベント追跡可 |
| FR-030 | PII マスキングポリシーを適用できること | MUST | ログに機微情報を残さない |

## Non-Functional Requirements

| ID | Requirement | Target |
|---|---|---|
| NFR-001 | 初回トークン遅延（ツールなし） | p95 < 2.5秒 |
| NFR-002 | 初回トークン遅延（ツールあり） | p95 < 8秒 |
| NFR-003 | 可用性 | 月間 99.9%以上 |
| NFR-004 | 永続化耐久性 | 書込成功率 99.99%以上 |
| NFR-005 | 同時セッション数 | 初期 1,000以上 |
| NFR-006 | セキュリティ | 通信 TLS、保存時暗号化 |
| NFR-007 | 監査性 | 全ターン追跡可能 |
| NFR-008 | 拡張性 | 新ツール追加でコア改修不要 |
| NFR-009 | 可観測性 | 全ターンに `trace_id` 付与 |
| NFR-010 | 復旧性 | セッション復元 RTO 15分以内 |

## Conversation Loop Standard

対話制御は以下の反復を基本とする。

```text
1) ユーザー入力を履歴へ追加
2) モデルへ履歴 + 文脈 + ツール定義を送信
3) ストリームイベントを受信し assistant message を構築
4) tool_use がなければターン終了
5) tool_use ごとに:
   - 権限判定
   - pre-hook
   - ツール実行
   - post-hook
   - tool_result を履歴へ追加
6) 2)へ戻る（上限まで）
7) セッション永続化
```

## Minimum Data Standard

最低限、次を追跡できること。

```json
{
  "session": {
    "id": "string",
    "created_at": "epoch_ms",
    "state": "active|closed"
  },
  "message": {
    "id": "string",
    "session_id": "string",
    "role": "user|assistant|tool",
    "blocks": []
  },
  "tool_call": {
    "id": "string",
    "name": "string",
    "input_json": {},
    "status": "pending|done|error"
  },
  "tool_result": {
    "tool_call_id": "string",
    "output": "string",
    "is_error": false
  },
  "policy_decision": {
    "tool_call_id": "string",
    "decision": "allow|deny",
    "reason": "string"
  },
  "retrieval_trace": {
    "query": "string",
    "sources": [
      { "id": "string", "score": 0.0 }
    ]
  }
}
```

## API Standard

代表的な最低限 API:

1. `POST /sessions`
2. `GET /sessions/{id}`
3. `POST /sessions/{id}/messages`
4. `GET /sessions/{id}/events`
5. `POST /sessions/{id}/cancel`
6. `POST /sessions/{id}/resume`
7. `GET /sessions/{id}/trace/{turn_id}`

## Error Handling Standard

1. モデル API 失敗時は再試行ポリシーに従う。
2. ツール失敗は `tool_result(is_error=true)` で返し、会話継続可能とする。
3. 権限拒否時は必ず理由を返却する。
4. 反復回数上限 / 予算上限到達時は停止理由を返却する。
5. セッション保存失敗時はユーザーへ再開不能リスクを通知する。

## Security Standard

1. ツールごとに required permission を定義必須とする。
2. 危険操作は明示承認必須とする。
3. API キー / トークンは KMS / Vault 等で管理する。
4. ログには PII / 秘密情報をマスクして保存する。
5. Web 取得はドメイン制御を可能にする。

## Observability Standard

1. 全リクエストに `request_id`、全ターンに `trace_id` を付与する。
2. 応答遅延、総応答時間、ツール実行時間、エラー率、反復回数、トークン使用量を記録する。
3. 監査ログは検索可能な形式で保持する。

## Testing Standard

最低限、次を検証する。

1. 単体テスト
   状態遷移、権限判定、イベント組み立て
2. 結合テスト
   モデルストリーム → ツール実行 → 再注入
3. E2E テスト
   通常対話、権限拒否、ツール失敗、タイムアウト
4. 負荷試験
   同時セッション、長時間セッション

## Acceptance Baseline

1. MUST 要件を 100% 満たす。
2. P1 / P2 の未解決障害がゼロである。
3. 監視ダッシュボードとアラートが本番運用可能である。
4. 障害時 runbook と復旧手順が整備済みである。

## Boundary Note

次は有用だが、この repository の invariant には含めない。

1. 実装フェーズ計画
2. マイルストーン週次計画
3. チケット粒度のタスク分解
4. 顧客導入順序と rollout 手順
