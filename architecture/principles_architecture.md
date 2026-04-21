# Principles: Architecture

## Purpose

この文書は、案件を超えて再利用するためのアーキテクチャ原則を定義する。対象は特定技術ではなく、構造の切り方、依存方向、共通化の基準である。

## Core Principles

1. 構成は層より先に機能単位で切る。
2. 依存方向は一方向に固定する。
3. 変更されやすい外部依存は境界へ寄せる。
4. 責務分離より読解性を優先する。ただし責務の混在は許容しない。
5. 構造は概念の美しさではなく、変更理由で切る。

## Preferred Split Axis

優先順位は次で固定する。

1. 第一分割軸は `feature` / `module`
2. 第二分割軸は `layer`

## Standard Top-Level Layout

```txt
<project-root>/
  frontend/
  backend/
  docs/
  tests/
  data/        # 必要時のみ
  templates/   # 必要時のみ
```

## Backend Principles

標準構成:

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
  core/
```

依存原則:

1. `modules/*/domain` は `application` / `api` / `infra` を参照しない。
2. `modules/*/application` は同一 module の `domain` を直接参照する。
3. `modules/*/api` は同一 module の `application` を呼び出す入口に限定する。
4. `modules/*/infra` は DB・Storage・外部 API などの実装詳細を持ち、業務判断を持ち込まない。
5. `core` は横断基盤のみを置く。

module 間依存原則:

1. `modules/*` 間の直接参照は原則禁止する。
2. 他 module の機能が必要な場合は、その module の公開境界を介する。
3. 共通概念を見つけても、すぐに `core` へ昇格しない。
4. 循環依存が発生した場合は module 分割の誤りを疑う。

## Core Promotion Principles

`core` に昇格してよいのは、次をすべて満たす場合のみとする。

1. 複数 module と複数層で再利用される。
2. 業務固有ではなく横断的技術基盤である。
3. `core` に置くことで依存方向が改善される。
4. 特定ユースケース名を知らなくても成立する責務である。

典型例:

- ロギング基盤
- 認証基盤
- メトリクス
- トレーシング
- 設定ローダー

## Frontend Principles

標準構成:

```txt
frontend/src/
  app/
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

原則:

1. `shared/ui` は業務知識を持たない。
2. `features/*/api` はその機能専用の通信処理のみ置く。
3. 状態はまず `features/*` 内で閉じる。
4. 画面専用コンポーネントは `shared` に上げない。
5. `shared` は再利用根拠があるものだけ置く。

## Shared Promotion Principles

`shared` に昇格してよいのは、次をすべて満たす場合のみとする。

1. 2機能以上で利用される。
2. 業務文脈に依存しない。
3. 利用側より定義側の責務が明確になる。

## File and Naming Principles

1. 1ファイル1責務を原則とする。
2. 責務が近く常に一緒に読まれる処理は同一ファイルにまとめてよい。
3. フォルダ名は技術用語より業務語彙を優先する。
4. use case 名は動詞始まりにする。
5. API schema 名は `XxxRequest` / `XxxResponse` のように意図を明示する。

## Testing Placement Principles

1. `domain` は unit test を厚くする。
2. `infra` は integration test で検証する。
3. `api` は request / response 契約をテストする。
4. `e2e` は主要業務フローに限定する。

## Anti-Patterns

1. 呼び出しを横流しするだけの中間層を増やさない。
2. DTO 変換だけを行うファイルを乱立させない。
3. `utils` に責務の異なる処理を混在させない。
4. 同じ概念を `frontend` / `backend` で別名管理しない。
5. 将来のためだけの抽象化を入れない。

`core` に次を入れてはいけない。

1. 特定業務フローにだけ必要なもの。
2. 実質 1 module 専用の処理。
3. 名前だけ一般化した業務ロジック。

`shared` に次を入れてはいけない。

1. まだ 1 機能でしか使っていないもの。
2. 名前だけ一般化した特定機能依存のもの。
3. 画面専用コンポーネント。

DTO の扱い:

1. API 境界・DB 境界・外部 API 境界以外で DTO を持ち込まない。
2. domain model を隠す必要がない箇所に DTO 変換レイヤーを増やさない。
3. DTO 変換を責務にしただけの層を設けない。

命名上の禁止事項:

1. `manager` を安易に使わない。
2. `helper` を安易に使わない。
3. `util` を安易に使わない。
4. `common` を安易に使わない。

## Decision Rules

迷った場合の優先順位は次で固定する。

1. 読みやすさ
2. 変更容易性
3. 実装速度
4. 抽象度の美しさ

新規ファイル追加前に次を確認する。

1. そのファイルが無いと責務上困るか。
2. 既存ファイルに統合できないか。
3. 依存方向を逆流させていないか。
4. テスト配置が責務と一致しているか。
5. 命名が業務語彙と一致しているか。

抽象化と分割の判断基準:

1. 最初は最小構成で開始する。
2. 重複が 3 回発生したら共通化を検討する。
3. 偶然の類似は共通化しない。
4. 同じ変更理由で一緒に修正されるコードは近づける。
5. 1ユースケースが長くなり読解コストが上がったら分割する。
6. 同一責務の分岐が 3 つ以上になったらオブジェクト化を検討する。
7. 別デプロイ単位・別実行単位になるまでは不要な package 分割をしない。

配置判断:

1. 依存注入はアプリケーションの入口境界でのみ行う。
2. DB アクセスは `infra` に閉じる。
3. API 入出力スキーマは `api/schemas` に集約する。
4. 業務概念の型は `domain` に置く。
5. `.env` と `.env.example` はプロジェクトルートに置く。
6. `pyproject.toml` / `package.json` は各実行単位ルートに置く。
7. `docker-compose.yml` はプロジェクトルートに置く。
8. アプリ固有設定は `infra/config` に集約する。

レビュー時の必須確認:

1. 不要な中継レイヤーを増やしていないか。
2. import 方向が原則を満たしているか。
3. module 粒度が過剰に細かくないか。
4. 責務が曖昧なフォルダがないか。
5. 将来拡張より現状の明快さを優先しているか。

## Small App Exception

小規模 PoC または小規模アプリでは、最初から厳密な多層構成を強制しない。

1. 初期は `api + service + repository` の最小構成で開始してよい。
2. 複雑性が上がった時点で段階的に分離する。
3. 原則は強制ではなく、複雑性に応じて適用する。
