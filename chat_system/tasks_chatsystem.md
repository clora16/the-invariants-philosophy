# 対話システム実装タスク分解書

- 文書ID: `DS-TASK-001`
- バージョン: `1.0.0`
- 作成日: `2026-04-04`
- 参照元: [dialog_system_requirements.md](dialog_system_requirements.md)

## 1. 目的
- 要件定義（FR/NFR）を実装可能なタスクに分解し、着手順と依存を明確化する。

## 2. 運用ルール
- タスクは `REQ-ID`（FR/NFR）と必ず紐づける。
- `MUST` 要件を先行し、`SHOULD/MAY` は後続フェーズで実装する。
- 各タスクの完了条件は「コード + テスト + 監視」まで含める。

## 3. エピック一覧

| Epic ID | 名称 | 対象Phase | 主対象REQ |
|---|---|---|---|
| EP-01 | Core Conversation Loop | A | FR-001〜010, FR-025, FR-027 |
| EP-02 | Tool Runtime & Policy | B | FR-005〜018, FR-031, FR-034, FR-036 |
| EP-03 | Reliability & Persistence | C | FR-002, FR-011, FR-016, FR-026, FR-033 |
| EP-04 | RAG & Attribution | D | FR-019〜021 |
| EP-05 | Governance & Audit | D | FR-017, FR-029, FR-030, FR-044 |
| EP-06 | Production Hardening | E | FR-035, FR-038〜045, NFR群 |

## 4. タスク分解（Phase別）

## 4.1 Phase A: Core Loop（最小）

| Task ID | タスク | REQ-ID | 依存 | 完了条件 |
|---|---|---|---|---|
| A-01 | セッション作成API実装 | FR-001 | なし | `POST /sessions` が `session_id` を返す |
| A-02 | メッセージ送信API実装 | FR-003 | A-01 | 時系列で user/assistant を保存 |
| A-03 | ストリーミング出力実装（SSE） | FR-004 | A-02 | `assistant_delta` が逐次配信される |
| A-04 | 基本会話ループ実装 | FR-007, FR-008, FR-009 | A-03 | `max_iterations/max_budget` 停止が動作 |
| A-05 | エラーコード統一 | FR-027 | A-04 | APIが機械可読コードを返す |
| A-06 | ターンキャンセル実装 | FR-025 | A-04 | 実行中ターンを中断可能 |
| A-07 | 最小テレメトリ導入 | NFR-001, NFR-009 | A-03 | `request_id/trace_id` を付与 |

## 4.2 Phase B: Tool + Policy

| Task ID | タスク | REQ-ID | 依存 | 完了条件 |
|---|---|---|---|---|
| B-01 | ツールレジストリ実装 | FR-005, FR-006 | A-04 | 複数 `tool_use` を処理可能 |
| B-02 | 権限モード判定実装 | FR-012, INV-006 | B-01 | deny時に実行されない |
| B-03 | 承認フロー実装 | FR-013, FR-014 | B-02 | 承認/拒否が履歴へ反映 |
| B-04 | タイムアウト/失敗継続 | FR-016, FR-018 | B-01 | tool失敗でも会話継続 |
| B-05 | pre/post hook実装 | FR-015 | B-01 | hook結果で挙動制御可 |
| B-06 | 認証導入（JWT/OIDC） | FR-031 | A-01 | 未認証アクセス `401` |
| B-07 | レート制限導入 | FR-034, NFR-013 | B-06 | `429` と再試行情報返却 |
| B-08 | モデレーション導入 | FR-036, NFR-014 | B-06 | 入出力判定ログが残る |

## 4.3 Phase C: Reliability

| Task ID | タスク | REQ-ID | 依存 | 完了条件 |
|---|---|---|---|---|
| C-01 | セッション永続化実装 | FR-002, NFR-004 | A-02 | 再開時に履歴復元 |
| C-02 | 履歴圧縮実装 | FR-011 | C-01 | 長セッションで閾値以内 |
| C-03 | 冪等再送実装 | FR-026, INV-005 | C-01 | 同一 `request_id` 二重実行防止 |
| C-04 | 同時更新競合制御 | FR-033 | C-01 | `revision` 不一致で `409` |
| C-05 | 復旧系テスト整備 | NFR-010 | C-01 | 再起動後に継続可能 |

## 4.4 Phase D: RAG + Governance

| Task ID | タスク | REQ-ID | 依存 | 完了条件 |
|---|---|---|---|---|
| D-01 | Retrieverインターフェース実装 | FR-019 | A-04 | `retrieval_context` 注入可能 |
| D-02 | 出典フォーマット標準化 | FR-020 | D-01 | source id/url/chunk を出力 |
| D-03 | ヒットなしフォールバック | FR-021 | D-01 | 規定応答へ遷移 |
| D-04 | 監査ログ強化 | FR-017, NFR-007 | B-01 | tool I/O が検索可能 |
| D-05 | PIIマスキング強化 | FR-030 | D-04 | 秘匿情報復元不可 |
| D-06 | バージョントレース導入 | FR-044, NFR-019 | D-04 | `prompt/model/policy` 追跡可 |
| D-07 | テナント分離実装 | FR-032, NFR-011 | C-01 | クロステナント参照不能 |

## 4.5 Phase E: Production Hardening

| Task ID | タスク | REQ-ID | 依存 | 完了条件 |
|---|---|---|---|---|
| E-01 | コスト上限制御 | FR-035, NFR-015 | C-01 | 予算超過で安全停止 |
| E-02 | モデルフォールバック | FR-039, NFR-016 | A-04 | 主要モデル障害時切替 |
| E-03 | プロンプト版管理/ロールバック | FR-038 | D-06 | 指定版へ即時切戻し |
| E-04 | 有人エスカレーション | FR-037, NFR-017 | B-06 | `handoff_requested` 遷移 |
| E-05 | 構造化出力検証 | FR-040 | A-04 | schema不一致時再試行 |
| E-06 | 曖昧入力の確認質問 | FR-041 | A-04 | clarification応答率を計測 |
| E-07 | 長期記憶の同意制御 | FR-042, FR-043, NFR-018 | C-01 | 忘却APIで削除可能 |
| E-08 | 実験/フラグ運用 | FR-045, NFR-020 | D-06 | 劣化検知で自動停止 |

## 5. 横断タスク（全Phase共通）

| Task ID | タスク | REQ-ID | 完了条件 |
|---|---|---|---|
| X-01 | API契約テスト自動化 | 23章, 32章 | CIで契約差分を検知 |
| X-02 | SLI/SLOダッシュボード | 25章 | TTFT/成功率を可視化 |
| X-03 | Runbook整備 | 27章, 32章 | SEV別手順がある |
| X-04 | セキュリティレビュー | 13章, 28章 | 脅威対策の証跡がある |
| X-05 | トレーサビリティ更新 | 31章 | REQ→実装→検証が1対1 |

## 6. 優先実装順（最短で価値を出す順）
1. A-01〜A-07
2. B-01〜B-04
3. C-01, C-03, C-04
4. D-01〜D-03
5. E-01, E-02, E-03
6. 以降 SHOULD/MAY を順次導入

## 7. チケットテンプレート

```md
### Title
[REQ: FR-xxx/NFR-xxx] <task title>

### Purpose
何を達成するか

### Scope
対象ファイル/モジュール

### Acceptance Criteria
- [ ] 条件1
- [ ] 条件2

### Test
- Unit:
- Integration:
- E2E:

### Metrics
影響するSLI/SLO

### Risk & Rollback
想定リスクと切り戻し手順
```
