# 対話システムマイルストーン計画

- 文書ID: `DS-MILE-001`
- バージョン: `1.0.0`
- 作成日: `2026-04-04`
- 参照:
  - [dialog_system_requirements.md](dialog_system_requirements.md)
  - [dialog_system_task_breakdown.md](dialog_system_task_breakdown.md)

## 1. 目的
- Phase A〜E の到達条件を固定し、Go/No-Go 判定を標準化する。

## 2. 全体ロードマップ（推奨12週間）

| 期間 | マイルストーン | 目標 |
|---|---|---|
| Week 1-2 | M1: Core MVP | 単純対話が安定動作 |
| Week 3-4 | M2: Tool & Policy | ツール実行と権限制御が成立 |
| Week 5-7 | M3: Reliability | 永続化/冪等/競合制御で運用可能化 |
| Week 8-9 | M4: RAG & Audit | 出典付き応答と監査追跡 |
| Week 10-12 | M5: Production Ready | SLO運用・DR・セキュリティ完了 |

## 3. マイルストーン詳細

## 3.1 M1: Core MVP（Phase A）

### 完了条件
- FR-001, FR-003, FR-004, FR-007, FR-008, FR-009, FR-025, FR-027 実装済み
- NFR-001 の暫定目標を満たす（TTFT p95 < 3.0秒）
- 単体 + 結合 + 基本E2EがCIで安定

### 判定ゲート
- Go: 主要シナリオ成功率 >= 99%
- No-Go: 停止理由未返却、またはキャンセル不全がある

## 3.2 M2: Tool & Policy（Phase B）

### 完了条件
- FR-005, FR-006, FR-012, FR-013, FR-014, FR-016, FR-018 実装済み
- FR-031, FR-034, FR-036 実装済み
- INV-006/007 を満たす

### 判定ゲート
- Go: deny時に tool が実行されない
- No-Go: 未認証アクセス通過、またはモデレーション未記録

## 3.3 M3: Reliability（Phase C）

### 完了条件
- FR-002, FR-011, FR-026, FR-033 実装済み
- NFR-004, NFR-010 を満たす
- 再起動/再送/競合系のE2Eを通過

### 判定ゲート
- Go: 同一 `request_id` 二重実行ゼロ
- No-Go: セッション再開で履歴欠損

## 3.4 M4: RAG & Audit（Phase D）

### 完了条件
- FR-019, FR-020, FR-021, FR-017, FR-030, FR-044 実装済み
- FR-032（テナント分離）実装済み
- NFR-007, NFR-011, NFR-019 を満たす

### 判定ゲート
- Go: クロステナント漏えい試験 0件
- No-Go: 出典欠落率が閾値を超過

## 3.5 M5: Production Ready（Phase E）

### 完了条件
- FR-035, FR-038, FR-039 は MUST として完了
- FR-040〜FR-045 は導入計画とガード条件を確定
- NFR-015〜NFR-020 の監視導線を実装
- 32章/39章のGo条件を満たす

### 判定ゲート
- Go: 1週間ステージングでSLOがエラーバジェット内
- No-Go: フォールバック不能、またはロールバック不能

## 4. クリティカルパス
1. 認証（FR-031）未完了だとB以降が全停止
2. 永続化（FR-002）未完了だとC以降の信頼性検証不可
3. テナント分離（FR-032）未完了だと本番判定不可
4. フォールバック（FR-039）未完了だと運用リスク高

## 5. メトリクス運用

| 指標 | 開始時点 | 責任 |
|---|---|---|
| TTFT p95 | M1 | Backend |
| Turn success rate | M2 | Backend/SRE |
| Tool success rate | M2 | Backend |
| Resume success rate | M3 | Backend |
| Cross-tenant leakage | M4 | Security |
| Cost overrun rate | M5 | Backend/PM |

## 6. リスクと対策

| リスク | 影響 | 対策 |
|---|---|---|
| プロンプト変更で品質劣化 | 高 | prompt_version固定 + 即時ロールバック |
| ツール障害連鎖 | 高 | timeout/retry/circuit breaker |
| 認証基盤遅延 | 中 | キャッシュ + 縮退モード |
| コスト超過 | 高 | 予算閾値 + 自動停止 |

## 7. マイルストーン判定テンプレート

```md
## Milestone Review: Mx

### Scope
- 完了タスク:
- 未完了タスク:

### Quality
- P1/P2件数:
- E2E成功率:

### SLO
- TTFT p95:
- Turn success rate:

### Security
- 漏えい試験:
- 監査証跡:

### Decision
- Go / Conditional Go / No-Go
- 条件付きの場合の解除条件:
```
