# Principles: Config

## Purpose

この文書は、設定管理を再利用可能な原則として定義する。目標は、型安全・再現性・運用性・秘密情報分離を同時に満たすことである。

## Core Principles

1. 設定は型で定義する。
2. 設定の優先順位は固定する。
3. secrets は設定本体から分離する。
4. 実行時に使った最終設定を再現可能な形で残す。
5. 設定スキーマ違反は起動時に失敗させる。

## Preferred Implementation Standard

この repository では、Python 系の設定管理に対して次を標準とする。

1. スキーマ定義には `pydantic.BaseModel` を使う。
2. 未知フィールド拒否には `ConfigDict(extra="forbid")` を使う。
3. 設定ファイル形式には `YAML` を使う。
4. ローカル開発時の環境変数読み込みには `python-dotenv` を使ってよい。
5. 実行時 override には CLI の `--set key=value` 形式を使う。

非標準:

1. `pydantic-settings` を中核にしない。
2. 環境変数優先の設定設計を標準にしない。

## Schema Rules

1. すべての実行設定は `BaseModel` などの型付きスキーマで定義する。
2. 未知フィールドは拒否する。
3. 必須 / 任意は型で明示する。
4. 業務ルールは validator で検証する。
5. アプリ本体で `getattr(..., default)` のような曖昧な参照をしない。
6. `extra="forbid"` を持つ基底クラスを 1 つ定義し、全サブモデルはそれを継承してよい。

## Source Priority Rules

優先順位は次で固定する。

1. `base.yaml`
2. `experiment.yaml` などの追加 YAML
3. CLI override
4. 環境変数

環境変数は原則 secrets の注入に限定する。

## Override Rules

1. override は複数指定を許可してよい。
2. 未知キーの override は即エラーにする。
3. 値は `yaml.safe_load` などで型付き解釈する。
4. override 処理は元の設定値を破壊的変更しない。
5. CLI 形式は `--set key=value` を標準とする。
6. ドット記法による部分 override は標準にしない。
7. override 対象は既存のトップレベルキーに限定する。

## Secret Handling Rules

1. API キーやトークンは設定ファイルに保存しない。
2. secrets は環境変数または同等の secret store から注入する。
3. `.env` はローカル開発補助に限定し、機密を commit しない。

## Reproducibility Rules

1. 実行に使った最終設定を保存する。
2. 保存された設定だけで再実行可能であることを優先する。
3. `None` や未設定値の扱いは再ロード時の挙動が変わらない形で固定する。
4. 実行に使った override を追跡可能にする。
5. 最終設定の保存時は `exclude_none=True` 相当を使い、`null` 混入による再ロード差異を避ける。

## Structural Defaults

推奨構成:

- `config_schema.py`: スキーマ定義
- `config_loader.py`: 読み込み・merge・secrets 注入
- `configs/`: 入力設定
- 実行時出力先: 再現可能な最終設定の保存先

最小テンプレート:

```python
from pydantic import BaseModel, ConfigDict

class StrictBaseModel(BaseModel):
    model_config = ConfigDict(extra="forbid")
```

```python
from pathlib import Path
import yaml

def load_config(config_yaml: str, set_overrides: list[str] | None = None):
    raw = yaml.safe_load(Path(config_yaml).read_text())
    if set_overrides:
        result = dict(raw)
        for item in set_overrides:
            key, _, val = item.partition("=")
            if key not in result:
                raise KeyError(f"Unknown key: {key}")
            result[key] = yaml.safe_load(val)
        raw = result
    return raw
```

## Acceptance Baseline

1. 未知フィールドで起動時に失敗する。
2. 業務ルール違反で起動時に失敗する。
3. 実行後に最終設定が再実行可能な形で残る。
4. secrets が設定ファイルへ混入しない。
5. `--help` に `--config-yaml` と `--set` が表示される。
6. 未知キー override 時に即エラーになる。
