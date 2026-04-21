# The Invariants Philosophy

この repository は、案件を超えて再利用できる不変資産を管理するための場所である。ここで再利用するのはライブラリやモジュールそのものではなく、設計思想・原理原則・最低限の実装基準である。

## Repository Purpose

この repository の目的は、良い実装を安定して生み出すための判断資産を再利用することにある。LLM や人間の実装者は、ここにある不変資産と案件側の customer requirements を組み合わせて設計と実装を行う。

## What This Repository Includes

この repository に含めるもの:

- principles
- anti-patterns
- decision rules
- capability standards
- principles や capability standards の適用を支援する補助文書
- review / testing / config / security の標準方針
- 案件横断で再利用できる glossary

この repository に含めないもの:

- customer requirements
- 顧客固有要件
- 案件固有制約
- 顧客固有の task breakdown
- 顧客固有の milestones
- 進捗、TODO、暫定メモ
- 顧客専用の API / UI 仕様

## Information Model

この repository では、文書を次の三層で整理する。

1. `Principles`
   どう考えるか、どう設計するか、何を避けるか、迷ったとき何を優先するかを定義する。
2. `Capability Standards`
   ある種類のシステムが最低限満たすべき実装基準を定義する。
3. `Support Documents`
   principles や capability standards を実装や導入へ落とすための補助文書を定義する。案件横断で再利用できるものだけを含める。
4. `Customer Requirements`
   顧客ごとの差分要件を定義する。これはこの repository には置かない。

## Naming Convention

命名規則は次で固定する。

1. フォルダで関心領域を表す。
2. ファイル名は `種類_対象.md` とする。
3. `種類` には `principles` / `anti_patterns` / `decision_rules` / `capability_standard` / `glossary` などを使う。
4. 顧客名、案件名、日付、版番号をファイル名に入れない。

例:

- `principles_architecture.md`
- `principles_config.md`
- `capability_standard_chat_system.md`

## Promotion Rule

案件側の知見をこの repository に昇格してよいのは、次を満たす場合のみとする。

1. 案件を超えて再利用できる。
2. 一時的ではなく比較的安定している。
3. 実装やレビューの判断軸として使える。
4. 顧客固有情報を含まない。
5. LLM に渡したとき再利用価値が高い。

満たさない場合は、案件側 repository または案件側 document に留める。

補助文書を置く場合も、次を満たす必要がある。

1. 特定顧客の進捗管理ではない。
2. 特定案件の納期や体制に依存しない。
3. principles または capability standards の適用支援として読める。

## How To Use

1. まず `Principles` を参照する。
2. 次に対象システムの `Capability Standard` を参照する。
3. 必要に応じて `Support Documents` を参照する。
4. 最後に案件側の `Customer Requirements` と合成する。
5. 実装・レビュー・設計判断の順で適用する。

## Writing Rules

この repository の文書は、次の形で書く。

1. 原則、禁止事項、判断基準、最低限基準として書く。
2. 一時的な実装状況や TODO を残さない。
3. 顧客固有要件を invariant に偽装しない。
4. 実装例は原則を説明する最小限に留める。
5. LLM に渡しやすいよう、短く構造化して書く。
