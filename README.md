# The Invariants Philosophy

旧来の再利用単位はライブラリ・モジュールだった。しかし実装コストが下がった今、再利用すべき資産をもう一段抽象化する必要がある。それは、案件を超えて適用できる設計思想・原理原則である。

顧客の要求はそれぞれ微妙に異なる。従来はその差異を既存ライブラリに無理やり合わせることで吸収しようとしていた。結果として、顧客要求と実装の双方に妥協が生まれていた。

原理原則を持つことで、これを起点に顧客の要求と組み合わせ、よりフィットした実装を生み出せる。LLM に **「原理原則」と「顧客の要求」** の二つを渡すことで、その合成を高い品質で実現できる。

## What this repository is

この repository は、案件を超えて再利用可能な **設計思想・原理原則・最低限の実装基準** を蓄積するための場所である。

ここで管理するのは、特定案件の要件や実装詳細ではない。  
管理対象は、複数案件で再利用される以下のような資産である。

- 設計原則
- 分割原則
- 命名原則
- anti-pattern
- 判断基準
- capability standard
- review 観点
- testing / config / security の標準方針

## What this repository is not

この repository は、案件固有の情報を置く場所ではない。  
以下は原則として含めない。

- 顧客固有要件
- 個別案件の制約
- milestone / task breakdown
- 現在の実装状況
- 一時的な意思決定メモ
- 顧客専用の API / UI 仕様

それらは各案件側の repository / document で管理する。

## Core idea

実装は、次の三層の合成で考える。

1. **Principles**  
   どう考え、どう設計し、何を避けるかを定義する。

2. **Capability Standards**  
   ある種類のシステムが最低限満たすべき実装基準を定義する。

3. **Customer Requirements**  
   顧客ごとの差分要件を定義する。

LLM には主に次を渡す。

- The Invariants Philosophy にある Principles / Standards
- 案件側の Customer Requirements

この二つを合成することで、案件ごとに適合した実装を生成する。

## Naming Convention

- フォルダは関心領域ごとに分ける  
  例: `architecture/`, `config/`, `testing/`, `standards/`

- ファイル名は `種類_対象.md` を基本とする  
  例:
  - `principles_architecture.md`
  - `anti_patterns_architecture.md`
  - `principles_config.md`
  - `capability_standard_chat_system.md`

- 種類には次を使う
  - `principles`
  - `anti_patterns`
  - `decision_rules`
  - `capability_standard`
  - `checklist`
  - `glossary`

- 顧客名・案件名は入れない
- 一時的な版番号や日付をファイル名に入れない

## Inclusion criteria

この repository に入れてよいものは、次を満たすものに限る。

案件を超えて再利用できる
一時的ではなく、比較的安定している
実装やレビューの判断軸として使える
顧客固有情報を含まない
LLM に渡したときに再利用価値が高い
Promotion rule

案件で生まれた知見をこの repository に昇格してよいのは、次の場合のみとする。

複数案件で繰り返し現れる
特定案件の事情に閉じない
単なる実装テクニックではなく、判断基準として一般化できる
今後も継続して参照する価値がある

## How to use

まず Principles を参照する
次に Capability Standard を参照する
最後に案件側の Customer Requirements と組み合わせる
実装・レビュー・設計判断を行う

# Goal

この repository の目的は、コードを再利用することではない。
良い実装を安定して生み出すための判断資産を再利用すること である。