# Architecture Playbook (Reusable)

このドキュメントは、次回以降どんなアプリでも「全体構成で迷わない」ための標準指針です。

## 1. Core Principles

1. 構成は「層」より先に「機能単位」で切る。
2. 依存方向は一方向に固定する。
3. 薄い中継ラッパーを作らない。
4. 変更されやすい外部依存を端へ寄せる。
5. 1ファイル1責務を原則とする。ただし責務が近く常に一緒に読まれる処理は同一ファイルにまとめてよい。責務分離より読解性を優先する。

分割軸の優先順位:

1. 第一分割軸 = feature/module
2. 第二分割軸 = layer

## 2. Standard Top-Level Layout

```txt
<project-root>/
  frontend/
  backend/
  docs/
  tests/
  data/            # 必要時のみ
  templates/       # 帳票などがある場合のみ
```

## 3. Backend Structure (Default)

```txt
backend/src/<app_name>/
  modules/
    users/
      domain/
      application/
      api/
      infra/
    orders/
      domain/
      application/
      api/
      infra/
  core/            # 横断基盤（厳格運用）
```

依存ルール:

1. `modules/*/domain` は `application/api/infra` を import しない。
2. `modules/*/application` は同一moduleの `domain` を直接参照する。
3. `modules/*/api` は同一moduleの `application` を呼び出す入口に限定する。
4. `modules/*/infra` は DB・Storage・外部APIなどの実装詳細を持ち、業務判断を持ち込まない。
5. `core` は共通化の根拠があるものだけ置く。

module 間依存ルール:

1. `modules/*` 間の直接参照は原則禁止する。
2. 他moduleの機能が必要な場合は、依存先moduleの `application` 層が公開するサービス境界を介する。
3. 複数moduleで共有する業務概念は、安易に `core` へ入れず、本当に全体共通かを確認する。
4. 循環依存は禁止。発生した場合は module 分割の誤りを疑う。

`core` 昇格条件（第二の `utils` 化を防ぐ）:

昇格許可:

1. 複数module・複数層で再利用される。
2. 業務固有ではなく、横断的技術基盤である。
3. `core` に置くことで依存方向が改善される。
4. 特定ユースケース名を知らなくても成立する責務である。
5. 具体例: ロギング基盤、認証基盤、メトリクス、トレーシング、設定ローダー。

昇格禁止:

1. 特定業務フローにだけ必要なもの。
2. 実質1module専用の処理。
3. 名前だけ一般化した業務ロジック（例: ページング計算、見積ロジック）。

判断基準: 「3層以上・複数moduleで同一契約が必要か」。不要なら `application` か `infra` に置く。

## 4. Frontend Structure (Default)

```txt
frontend/src/
  app/             # ルーティング・DI・起動設定
  features/
    auth/
      components/
      hooks/
      api/
      types/
    orders/
      components/
      hooks/
      api/
      types/
  shared/
    ui/
    lib/
    types/
```

ルール:

1. `shared/ui` は業務知識を持たない。
2. `features/*/api` はその機能専用の通信処理のみ置く。
3. グローバル状態は最小限にし、まず `features/*` 内で閉じる。
4. 画面専用コンポーネントを `shared` に上げない。
5. `shared` をゴミ箱にしない。

## 5. Anti-Patterns (禁止)

1. 呼び出しを横流しするだけの中間層。
2. DTO変換だけを行うファイルの乱立。
3. `utils` に責務の異なる処理を混在。
4. 同じ概念を `frontend/backend` で別名管理。
5. 将来のためだけの抽象化。

DTOの扱い:

1. DTOを持ってよいのは、API境界・DB境界・外部API境界。
2. domain modelをそのまま外へ出したくない場合のみDTO変換を設ける。
3. それ以外では、DTO変換専用レイヤーを増やさない。

## 6. Add-File Checklist

新規ファイル追加前に必ず確認する:

1. このファイルが無いと責務上困るか。
2. 既存ファイルに統合できないか。
3. 依存方向を逆流させていないか。
4. テストの配置先が責務と一致しているか。
5. 命名が業務語彙と一致しているか。

## 7. Decision Rules

迷った場合の優先順位:

1. 読みやすさ
2. 変更容易性
3. 実装速度
4. 抽象度の美しさ

## 8. Practical Defaults

1. 最初は「最小構成」で開始する。
2. 重複が3回発生したら共通化を検討する。ただし偶然の類似は共通化しない。
3. 依存注入はアプリケーションの入口境界（`api`、`app`、起動点）でのみ行う。
4. DBアクセスは `infra` に閉じる。
5. API入出力スキーマは `api/schemas` に集約する。
6. 業務概念の型（Value Object/Entity型）は `domain` に置く。

分割・抽象化の判断基準:

1. 1ユースケースが長くなり読解コストが上がったら分割する。
2. 同一責務の分岐が3つ以上になったらオブジェクト化を検討する。
3. 同じ変更理由で一緒に修正されるコードは近づける。
4. 別デプロイ単位・別実行単位になるまでは不要なpackage分割をしない。
5. 構造は概念の美しさではなく「変更理由」で切る。

`shared` 昇格条件（frontend）:

1. 2機能以上で利用され、業務文脈に依存しない。
2. 呼び出し側が増えても意味がぶれない。
3. 利用側より定義側の責務が明確になる。

`shared` への昇格禁止:

1. まだ1機能でしか使っていないもの。
2. 名前を一般化しただけで実質特定機能依存のもの。

ルート設定ファイルの標準配置:

1. `.env(.example)` はプロジェクトルート。
2. `pyproject.toml` / `package.json` は各実行単位ルート（monorepoでは frontend/backend それぞれ）。
3. `docker-compose.yml` はプロジェクトルート。
4. アプリ固有設定は `backend/src/<app_name>/infra/config` へ集約。

## 9. Review Gate (PR)

PRレビュー時の必須観点:

1. 不要な中継レイヤーを増やしていないか。
2. import方向が原則を満たすか（`modules/*` 間の直接参照がないか含む）。
3. モジュール粒度が過剰に細かくないか。
4. 責務が曖昧なフォルダがないか。
5. 将来拡張より現状の明快さを優先しているか。

## 10. Testing Policy

標準配置:

```txt
tests/
  unit/
  integration/
  e2e/
```

機能別に寄せる場合:

```txt
tests/
  users/
    unit/
    integration/
  orders/
    unit/
    integration/
```

ルール:

1. `domain` は unit test を厚く。
2. `infra` は integration test で確認。
3. `api` は request/response 契約をテスト。
4. `e2e` は主要業務フローに限定。

## 11. Naming Policy

1. フォルダ名は技術用語より業務語彙を優先する。
2. `manager`, `helper`, `util`, `common` を安易に使わない。
3. usecase名は動詞始まりにする。
4. API schema名は `XxxRequest`, `XxxResponse` のように意図を明示する。

## 12. Exceptions (PoC/Small App)

1. 小規模PoCでは `domain/application/infra` を最初から分けなくてよい。
2. まずは `api + service + repository` の最小構成で開始してよい。
3. 複雑性が上がった時点で段階的に分離する。
4. Playbookは強制ではなく、複雑性に応じて適用する。