# AGENT.md

## Repository Role

この repository は、案件横断で再利用するための principles と capability standards を管理する場所である。agent は個別案件の実装詳細や進捗をここへ持ち込まず、判断資産として再利用できる内容だけを扱う。

## Priority Order

判断に迷った場合の優先順位は次で固定する。

1. Principles
2. Capability Standards
3. Customer Requirements
4. Local implementation convenience

local implementation convenience を理由に principles を破ってはいけない。customer requirements は principles と standards の適用方法を調整するために使う。

## What May Be Stored

保存してよいもの:

- 設計原則
- anti-patterns
- decision rules
- capability standards
- principles や capability standards の適用を支援する補助文書
- review / testing / config / security の標準方針
- glossary

保存してはいけないもの:

- 顧客固有要件
- 案件固有制約
- 顧客固有の task list
- 顧客固有の milestone
- 進捗
- TODO
- 一時的な設計メモ
- 顧客専用の UI / API 仕様
- 特定案件でしか成立しない workaround

## Invariant Writing Rules

文書追加・更新時は次を満たすこと。

1. 顧客固有名詞を書かない。
2. 一時的な事情を書かない。
3. 原則・禁止事項・判断基準・最低限基準として記述する。
4. 許容条件と禁止条件を明示する。
5. 実装例は最小限に留める。
6. 再利用可能な判断軸になっていることを優先する。

## Promotion Criteria

案件側の知見を invariant として昇格してよいのは、次を満たす場合のみ。

1. 複数案件で再利用可能である。
2. 特定顧客事情に依存しない。
3. 将来も判断基準として参照する価値がある。
4. 実装断片ではなく、principles または standards として自然に位置付けられる。
5. 補助文書の場合でも、特定案件の進捗管理ではなく適用支援として一般化できる。

満たさない場合は案件側に残す。

## Anti-Goals

この repository を次の用途に使ってはいけない。

1. 万能な知識置き場
2. project spec の代替
3. 顧客固有の task 管理の代替
4. 顧客要求の偽装保存
5. 一般化が不十分なベストプラクティス置き場

## Expected Workflow

1. 課題を `principle` / `capability standard` / `customer requirement` に分解する。
2. 既存文書で足りるか確認する。
3. 足りるなら既存文書を適用する。
4. 適用支援が必要なら、案件横断で再利用可能な補助文書として一般化できるか確認する。
5. 足りないなら案件側差分として扱う。
6. 複数案件で再利用価値がある場合のみ、この repository への昇格を検討する。

## Output Preference

agent は次の順で判断を整理すること。

1. principle
2. rationale
3. allowed patterns
4. disallowed patterns
5. minimal standard
6. promotion decision
