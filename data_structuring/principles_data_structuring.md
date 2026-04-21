# Principles: Data Structuring

## Purpose

この文書は、業務データを再利用可能な構造へ変換するための data structuring 原則を定義する。対象は raw data を canonical schema、retrieval payload、knowledge graph などの安定した表現へ変換する設計である。

## Core Principles

1. raw data を直接最終表現へ変換しない。
2. 入力依存の揺れを吸収する intermediate layer を持つ。
3. canonical schema を唯一の正本にする。
4. canonical-only mode を基本とし、非 canonical な旧キー互換を正本へ持ち込まない。
5. retrieval 用表現と graph 用表現は canonical から派生させる。
6. domain knowledge と transformation logic を分離する。
7. 同一性判定は名称ではなく stable identity で扱う。
8. 正規化と validation は後回しにせず、構造化の早い段階で行う。

## Transformation Layers

data structuring は最低限、次の層を分離する。

1. `raw`
   原本データ。Excel、CSV、PDF抽出結果、外部システム出力など入力依存の揺れを含む。
2. `intermediate`
   raw のレイアウト差異や抽出差異を吸収する中間表現。入力形式ごとの差分はこの層で閉じる。
3. `canonical`
   業務上の正本となる構造。内部キー、型、必須項目、同一性ルールを固定する。
4. `retrieval payload`
   検索・埋め込み・RAG のための派生表現。検索用 `text` や metadata を持ってよい。
5. `graph projection`
   node / edge と relation に変換した派生表現。knowledge graph や graph DB インポートのために用いる。

原則:

1. raw の揺れを canonical に持ち込まない。
2. canonical を飛ばして retrieval や graph を正本にしない。
3. 各層の責務を混ぜない。
4. layer をまたぐ convenience 変換を入れない。
5. 上流の都合で下流 schema を歪めない。

## Intermediate Layer Principles

1. raw input のレイアウト依存を吸収するために intermediate schema を定義する。
2. intermediate は抽出結果の安定化に責務を限定する。
3. 入力ファイル種別ごとの差分は intermediate 生成側で処理する。
4. canonical 固有の業務判断を intermediate に持ち込まない。
5. human review や redacted などの抽出時フラグは intermediate で保持してよい。
6. intermediate は raw の完全複製ではなく、canonical 化に必要な最小構造へ整形する。
7. intermediate schema は入力形式よりも後続変換責務に合わせて設計する。

## Review Flag Principles

1. 自動抽出で確信が持てない項目は `human_review` などの review flag を持てるようにする。
2. 非公開または読み取り不能な領域は `redacted` などの明示フラグで扱う。
3. review flag は validation failure と混同しない。
4. review flag を持つ項目でも canonical へ進めてよいが、後続処理で追跡可能にする。

## Canonical Schema Principles

1. canonical schema を唯一の正本とする。
2. 内部キーは英語 `snake_case` など一貫した命名規則で固定する。
3. 表示名と内部キーを混同しない。
4. 旧キーや入力元固有の列名を canonical の正式仕様に残さない。
5. 1つの概念は 1 つの canonical key で表す。
6. 必須項目、任意項目、型、フォーマットを明示する。
7. 新規出力は canonical schema にのみ準拠させる。
8. canonical schema は CSV、JSONL、DB payload のすべてで語彙を揃える。
9. canonical の列定義はコード・文書・テストで一致させる。

## Canonical-Only Mode

1. 入力として受け付けるキーは canonical key のみに絞る。
2. 旧カラム名や暫定別名の自動受理を標準動作にしない。
3. 互換変換が必要でも、raw から intermediate までで閉じる。
4. canonical layer では「何が正式仕様か」を曖昧にしない。
5. canonical-only mode は入力受理範囲を絞るための設計であり、利用者都合の緩和を優先しない。
6. 厳格拒否が未実装でも、正式仕様は canonical key のみとする。
7. 実装上の移行期間では warning を許容してよいが、暗黙互換を恒久仕様にしない。

## Normalization Principles

1. 文字列比較・ID生成・マッチングの前に必ず normalize する。
2. Unicode 正規化方式を固定する。
3. trim、全半角統一、ゼロ幅文字除去などを一元化する。
4. 空値は `None` / 空文字 / 欠損の扱いを固定する。
5. normalize の実装を複数箇所に分散させない。
6. normalize 後の値だけを canonical、ID、relation 判定に使う。
7. normalize 関数は single source of truth として共通化する。

## Identity Principles

1. 人間向け名称をそのまま ID にしない。
2. ID は stable かつ deterministic に生成する。
3. 同じ入力から同じ ID が再生成できることを優先する。
4. file をまたいでも同一概念を識別できるようにする。
5. ID 生成に使う要素は canonical 化後の値に限定する。
6. stable id は hash 等で短く再生成可能な形式で持ってよい。
7. stable id の構成要素は entity の境界定義そのものとして扱う。

## Validation Principles

1. required field は canonical 化時に検証する。
2. 日付、年月、数値などの format validation を早期に行う。
3. 曖昧な値を黙って補完しない。
4. 不正フォーマットは fail fast で止める。
5. canonical-only mode を採る場合、非 canonical 入力を暗黙変換しない。
6. 空行や意味のない行は黙って落としてよいが、意味があるのに不正な行は落とさず失敗させる。
7. skip と error の基準を実装内で統一する。
8. skip できる条件と fail すべき条件は、コードまたは補助文書で追跡可能にする。

## Aggregation Principles

1. long 形式の入力は canonical では long のまま保持してよい。
2. retrieval や graph 用に集約が必要な場合は、派生レイヤーで集約する。
3. 集約単位は stable id の構成要素と一致させる。
4. 月次配分やサブ明細が必要な場合は、親レコードと子レコードを分離して持つ。
5. 集約後の派生表現でも、元の canonical 行へ戻れることを優先する。

## Projection Principles

1. retrieval payload は canonical の派生物とする。
2. graph projection も canonical の派生物とする。
3. 派生表現は用途別最適化をしてよいが、正本を兼ねさせない。
4. 検索用 `text` は保持してよいが、業務正本は canonical に残す。
5. graph の node / edge は canonical の概念と relation から組み立てる。
6. retrieval payload には `text`、`type`、`metadata` など検索・推論に必要な要素だけを載せる。
7. graph projection は graph DB 都合の列構造に変換してよいが、canonical の意味を変えない。
8. canonical CSV と canonical JSONL は正本と派生の中間表現として役割を分けてよい。
9. case-level retrieval が必要な場合は、entity 群から summary を構成した上で embedding する。

## Retrieval Payload Principles

1. retrieval payload は entity 単位または case 単位のどちらかを明示して作る。
2. entity payload には `type` と `text` を必須で持たせる。
3. metadata は検索補助のために持ってよいが、canonical の代替にしない。
4. case-level summary は canonical entity 群から生成する。
5. embedding 対象は retrieval 用に整えたテキストであり、raw 文書全体をそのまま使わない。

## Relation Modeling Principles

1. 文書間・概念間の対応は独立した relation として明示する。
2. relation は少なくとも `relation_type`、`confidence`、`source` を持てるようにする。
3. 完全一致、別名一致、部分一致などは relation type で区別する。
4. 暗黙 join や名前一致だけで意味関係を確定しない。
5. relation の根拠を後から追跡できるようにする。
6. relation は node 側の属性に埋め込まず、独立した edge / mapping として持つ。

## Group Mapping Principles

1. 文書間のグループ対応は `group_mapping` のような独立成果物で管理する。
2. mapping は `exact`、`alias`、`partial` など段階を分けて扱う。
3. 名前一致だけで十分でない場合に備えて、confidence を持たせる。
4. mapping ルールの起源は `source` で残す。
5. mapping は後から差し替え可能な設計にする。
6. mapping 生成ロジックは deterministic rule を優先し、推論依存を標準にしない。

## Knowledge Separation Principles

1. domain knowledge は transformation code と分離する。
2. client-specific な mapping や vocabulary を汎用 schema に埋め込まない。
3. knowledge は更新可能なファイルまたは設定として管理する。
4. code は normalize、validate、canonicalize、project に責務を絞る。
5. 業務知識の変更でコード変更が必要になる設計を避ける。
6. code に埋めるのは一般則だけとし、業務語彙・分類表・マッピング表は knowledge 側へ置く。
7. knowledge は「変わり得る業務ルール」、code は「変わりにくい変換規則」を担う。
8. 一般的な parsing、normalization、stable id 生成規則は code に残してよい。
9. 顧客固有の抽出ラベル対応、語彙表、分類表、マッピング表は knowledge 側へ寄せる。

## Anti-Patterns

1. raw data を直接 graph 化する。
2. retrieval 用 JSON や embedding payload を正本化する。
3. UI 表示名を内部キーとして流用する。
4. 手入力の名称や連番だけに依存して entity identity を管理する。
5. 正規化前の文字列でマッチングする。
6. 顧客固有の vocabulary や mapping を canonical schema に焼き込む。
7. intermediate と canonical の責務を混ぜる。
8. relation を表で持たず、暗黙的な列対応に依存する。
9. review flag が必要な曖昧抽出を、確定データとして downstream に流す。
10. 派生レイヤーごとに別々の normalization 実装を持つ。
11. case-level summary を canonical を経由せず手作業で別管理する。

## Decision Rules

迷った場合は次を優先する。

1. まず canonical schema を整える。
2. 次に normalization と validation を固定する。
3. 次に identity と relation の表現を明示する。
4. retrieval や graph の都合で canonical を歪めない。
5. 業務知識は code ではなく knowledge 側へ寄せる。
6. 自動抽出に迷いがある場合は review flag を残し、黙って確定しない。
7. skip で済むか fail すべきか迷ったら、意味のある不正データは fail を優先する。

## Minimal Standard

最低限、次を満たすこと。

1. intermediate schema がある。
2. canonical schema がある。
3. normalize 関数が一元化されている。
4. stable id の仕組みがある。
5. 必須項目と format validation がある。
6. retrieval payload は canonical から生成される。
7. graph projection は canonical から生成される。
8. relation を明示表現する仕組みがある。
9. review flag を保持できる。
10. knowledge と code の配置責務が分かれている。
11. skip / error の基準が定義されている。
12. retrieval 用 summary または `text` の生成規則が定義されている。
