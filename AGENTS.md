
# AGENT.md

## Purpose

この repository は、案件横断で再利用可能な原理原則と capability standard を管理するためのものである。  
この repository を参照する agent は、個別案件の実装詳細をここに持ち込まず、再利用可能な判断資産の整備と適用を優先すること。

## Role of the agent

agent の役割は次の通り。

- 原理原則を読み取り、設計判断に反映する
- capability standard を読み取り、最低限満たすべき基準を明確にする
- 案件固有要件と不変資産を分離して扱う
- 案件固有の知見を、昇格条件を満たす場合のみ invariant として一般化する
- この repository を一時的メモや案件固有情報の保管場所にしない

## Priority order

判断に迷った場合は、次の順で優先する。

1. Principles
2. Capability Standards
3. Customer Requirements
4. Local implementation convenience

ローカルな実装都合によって principles を破らないこと。  
customer requirements によって principles と standards の適用方法を調整してよいが、無根拠に逸脱しないこと。

## What to store here

保存してよいもの:

- 設計原則
- anti-pattern
- 判断基準
- capability standard
- review 基準
- testing/config/security の標準方針
- 案件横断で再利用可能な glossary

保存してはいけないもの:

- 顧客固有要件
- 個別案件の TODO
- 一時的な設計メモ
- milestone / task list
- 顧客専用の UI/API 仕様
- 特定案件でしか成立しない workaround

## Invariant writing rules

この repository に追加・修正する文書は、次を満たすこと。

- 顧客固有名詞を書かない
- 一時的な事情を書かない
- 原則・基準・判断条件として記述する
- できるだけ禁止事項と許容条件を明確にする
- 「何をすべきか」だけでなく「何を入れてはいけないか」も書く
- 実装例を書く場合も、原則を説明するための最小例にとどめる

## Promotion criteria

案件側の知見を invariant として昇格してよいのは、以下を満たす場合のみ。

- 複数案件で再利用可能
- 特定顧客事情に依存しない
- 将来も判断基準として参照したい
- 単なる記述の再利用ではなく、設計判断の再利用になる
- principles または standards として自然に位置付けられる

満たさない場合は、案件側 document に留めること。

## Expected workflow

1. 問題を「principle の問題」か「standard の問題」か「customer requirement の問題」かに分解する
2. まず既存の principles / standards を検索する
3. 既存資産で足りる場合は、それを適用する
4. 足りない場合は、案件固有の差分として扱う
5. 複数案件で再利用価値があると判断した場合のみ、この repository への追加を検討する

## Output style

agent は次の形式を優先して出力すること。

- principle
- rationale
- allowed patterns
- disallowed patterns
- minimal standard
- promotion decision

曖昧な一般論ではなく、判断可能な基準として記述すること。

## Anti-goals

この repository を次のように使ってはいけない。

- 万能な知識置き場にする
- project spec の代わりに使う
- task 管理の代わりに使う
- 顧客要求を invariant に偽装して保存する
- 思いついたベストプラクティスを十分な一般化なしに追加する

## Definition of done for updates

更新は次を満たしたときに完了とする。

- 文書の対象が案件横断である
- 原則または最低限基準として読める
- 含めるもの / 含めないものが明確である
- 将来の案件で再利用するイメージが持てる
- customer-specific な記述が除去されている