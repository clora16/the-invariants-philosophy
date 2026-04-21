# コンフィグ要件定義書（ベストプラクティス）

## 1. 目的
- プロジェクト開始時に毎回コンフィグ方針を議論し直さないため、再利用可能な標準要件を定義する。
- 目標は「型安全」「再現性」「CLI運用性」「秘密情報分離」の同時達成。

## 2. 適用範囲
- PythonのML/分析系プロジェクト（学習・推論・変換スクリプトを含む）。
- 単体スクリプト構成から中規模実験運用までを対象。

## 3. 採用技術（必須）
- `pydantic.BaseModel`（`extra="forbid"`）
- `YAML`（実験パラメータ）
- `python-dotenv`（ローカル開発時の環境変数読み込み）
- CLI override (`--set key=value`)

非採用（標準外）:
- `pydantic-settings` を中核にしない（環境変数優先の挙動がML実験と不一致になりやすいため）

## 4. 設定ソースと優先順位（必須）
優先順位は次で固定する。

1. `base.yaml`
2. `experiment.yaml`（任意）
3. CLI override（`--set`）
4. 環境変数（原則 secrets のみ）

## 5. 設定スキーマ要件（必須）
- すべての実行設定は `BaseModel` で定義する。
- `model_config = ConfigDict(extra="forbid")` を有効化する。
- 繰り返しを避けるため、`extra="forbid"` を持つ基底クラス（例: `StrictBaseModel`）を1つ定義し、全サブモデルはそれを継承する。
- 業務ルール（例: モードとロスの整合性）は `model_validator` で検証する。
- 必須/任意を型で明示する（`Optional` の乱用禁止）。
- アプリ本体で `getattr(cfg, "...", default)` を禁止する。

## 6. CLI override 要件（必須）
- `--set` は複数指定を許可する（例: `--set batch_size=64 lr=3e-4`）。
- ドット記法（例: `train.lr=...`）は禁止し、トップレベルキーのみ許可する。
  - 注意: ネスト内の1フィールドだけを変更したい場合は、dict ごと渡す（例: `--set scheduler='{type: cosine, gamma: 0.1}'`）。
- 既存のトップレベルキーのみ上書き可能とし、未知キーは即エラー。
- 値パースは `yaml.safe_load` 等で型付き変換する。
- `apply_set_overrides` は入力 dict を破壊的変更せず、コピーを返すこと。

## 7. secrets 要件（必須）
- APIキー等は YAML に保存しない。
- secrets は環境変数からのみ注入する。
- `.env` はローカル開発補助としてのみ使用し、リポジトリへコミットしない。

## 8. 再現性要件（必須）
- 学習/推論実行時に、最終設定（YAML + override適用後）を `exp` 配下へ YAML として保存する。
- 保存ファイル名は入力 `--config-yaml` の元ファイル名を継承する（再実行時にそのまま指定できること）。
- 保存時は `None` フィールドを除外する（`model_dump(exclude_none=True)`）。`null` が混入すると再ロード時の挙動が変わるため。
- 実行に使った override をログに残す。

## 9. スクリプト要件（必須）
- `train.py` / `infer.py` 等の主要エントリーポイントは同一の設定ローダを使う。
- モード分岐はスキーマ上の列挙値（例: `val_mode: "map" | "feature"`）で表現し、スクリプト内でのハードコードを避ける。
- Augment の命名と実装を一致させる（例: スキーマのフィールド名とクラス名を揃える）。
- 学習開始時に有効化された Augment 名をログへ出力する。

## 10. 命名・構成要件（推奨）
- `config_schema.py`: Pydanticモデル定義
- `config_loader.py`: YAML読み込み、`--set` 適用、環境変数注入
- `configs/`: 実験設定ファイル
- `<input_config_name>.yaml`（exp配下）: 実行時確定設定

## 11. 受け入れ基準（Definition of Done）
- `--help` に `--config-yaml` と `--set` が表示される。
- 未知キーoverride時に即エラーになる。
- YAML typo（未知フィールド）で起動時に失敗する。
- スキーマの `model_validator` で検証する業務ルール違反時に起動時に失敗する。
- 実行後に `exp` 配下へ再実行可能な最終YAMLが出力される。
- secretsがYAMLに含まれていない。

## 12. 実装テンプレート（最小）
```python
# schema
from pydantic import BaseModel, ConfigDict

class StrictBaseModel(BaseModel):
    model_config = ConfigDict(extra="forbid")

class DataConfig(StrictBaseModel):
    batch_size: int = 2
    train_ann: str  # 必須フィールドはデフォルト値なし

class ExperimentConfig(StrictBaseModel):
    experiment_name: str
    seed: int = 42
    data: DataConfig
```

```python
# config_loader.py（最小）
from pathlib import Path
import yaml
from project.config_schema import ExperimentConfig

def load_config(config_yaml, set_overrides=None):
    raw = yaml.safe_load(Path(config_yaml).read_text())
    if set_overrides:
        result = dict(raw)
        for item in set_overrides:
            key, _, val = item.partition("=")
            if key not in result:
                raise KeyError(f"Unknown key: {key}")
            result[key] = yaml.safe_load(val)
        raw = result
    return ExperimentConfig.model_validate(raw)
```

```bash
# run
python -m project.train \
  --config-yaml configs/base.yaml \
  --set batch_size=64 lr=0.0005
```

## 13. 変更管理ルール
- コンフィグ仕様変更は本書を先に更新し、実装は後追いで同期する。
- 互換性を壊す変更は「移行手順」と「影響範囲」をPR本文に明記する。