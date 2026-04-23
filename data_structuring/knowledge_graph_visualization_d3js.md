# Knowledge Graph Visualization: D3.js アプローチ

## 位置付け

この文書は `principles_data_structuring.md` の **Graph Projection** レイヤーから生成された graph データを、  
ブラウザで対話的に可視化する実装手法を記録する。

可視化ライブラリとして **D3.js v7** を採用した。  
生成物は自己完結した単一 HTML ファイルであり、サーバー不要でブラウザで開ける。

---

## アーキテクチャ概要

```
canonical CSV
    │
    ▼
graph projection (Python)
    │  species_nodes.csv
    │  species_edges.csv
    │  graph_metadata.json
    ▼
visualizer (Python → HTML)
    │  D3.js CDN を参照する単一 HTML
    ▼
ブラウザで描画・操作
```

Python スクリプトは 2 つに分離する。

| スクリプト | 責務 |
|---|---|
| `build_species_graph.py` | canonical CSV から nodes / edges CSV と metadata JSON を生成 |
| `visualize_species_graph.py` | nodes / edges CSV を読み込み、D3.js HTML を生成 |

graph 構築と可視化の責務を分離することで、可視化ロジックを canonical data を壊さずに差し替えられる。

---

## データ形式

### nodes (Python → JSON)

```json
{
  "id": "species_code",
  "label": "species_code",
  "color": "#6c5ce7",
  "radius": 18.4,
  "occurrence_count": 312,
  "num_neighbors": 7,
  "weighted_degree": 890,
  "train_audio_occurrence_count": 280,
  "soundscape_occurrence_count": 32
}
```

- `radius` は `weighted_degree` の平方根に比例させ、差が大きい場合でも視認できる範囲に収める。
- `color` は Python 側で計算済みの hex 値を渡す。D3 側で再計算しない。

### links (Python → JSON)

```json
{
  "source": "species_a",
  "target": "species_b",
  "cooccur_count": 88,
  "lift": 3.4,
  "ppmi": 1.2,
  "width": 3.1,
  "opacity": 0.61
}
```

- `source` / `target` は D3 の `forceLink` が文字列 id からオブジェクト参照へ自動解決する。
- `width` / `opacity` は Python 側で `cooccur_count` を正規化して計算済みで渡す。

---

## D3.js 実装の要点

### Force Simulation

```js
d3.forceSimulation(nodesData)
  .force("link",    d3.forceLink(linksData).id(d => d.id).distance(120))
  .force("charge",  d3.forceManyBody().strength(-120))
  .force("center",  d3.forceCenter(W / 2, H / 2))
  .force("collide", d3.forceCollide().radius(d => d.radius + 4))
  .alphaDecay(0.028)
```

- `forceManyBody` の `strength` を負値にするとノード間に斥力が働き、密集を防ぐ。
- `forceCollide` でノード半径＋マージン分の衝突判定を入れ、重なりを抑える。
- `alphaDecay` を小さめ（0.028）にすると安定するまで時間をかけてゆっくり収束する。

### Zoom / Pan

```js
const zoom = d3.zoom().scaleExtent([0.05, 8]).on("zoom", e => g.attr("transform", e.transform));
svg.call(zoom);
```

SVG 直下に `<g>` レイヤーを置き、zoom transform をそのレイヤーに適用する。  
SVG 自体のサイズは固定したまま、内部 `<g>` だけ移動・拡縮するパターン。

### ドラッグ

```js
d3.drag()
  .on("start", (e, d) => { sim.alphaTarget(0.3).restart(); d.fx = d.x; d.fy = d.y; })
  .on("drag",  (e, d) => { d.fx = e.x; d.fy = e.y; })
  .on("end",   (e, d) => { sim.alphaTarget(0); d.fx = null; d.fy = null; })
```

ドラッグ中は `fx` / `fy` で位置を固定し、終了後に解放することで simulation に戻す。

### 隣接ノードのハイライト

クリック時に隣接マップを参照して非隣接ノード・エッジの opacity を下げる。  
隣接マップは描画前に links から事前構築し、クリックのたびに再計算しない。

```js
const neighborMap = new Map();
linksData.forEach(l => {
  neighborMap.get(s).add(t);
  neighborMap.get(t).add(s);
});
```

---

## カラーリング

### ノード色：Python 側で計算

指標値（`weighted_degree` など）を正規化し、3点カラーパレットで線形補間する。

```
低値 → 中値 → 高値
#00cec9 → #6c5ce7 → #e17055
(teal)   (indigo)  (coral)
```

Python で hex 文字列として計算済みの値を JSON に埋め込み、D3 側は受け取るだけにする。  
これにより可視化スクリプトが D3 の color scale に依存しない。

### エッジ色

- 通常: `#b2bec3`（ライトグレー）— 背景に溶け込ませ、ノードを前面に立たせる。
- ハイライト: `#e17055`（フラットコーラル）— 選択ノードの接続エッジだけを強調。

---

## フラットデザイン方針

vis.js 標準テーマから切り替えた際に採用した設計指針。

| 要素 | 方針 |
|---|---|
| ノード輪郭 | `stroke: none` — ボーダーレスで塗りのみ |
| カード・パネル | `border: none` + `box-shadow` のみ |
| ボタン | `border: none`、`border-radius: 6px`、active 時のみ背景色 |
| 背景 | `#f0f3f7`（極淡いグレー）— 純白より目に優しい |
| テキスト | `#2d3436`（メイン）/ `#636e72`（サブ）の 2 トーン |

影は `box-shadow: 0 4px 16px rgba(45,52,54,0.12)` のみ使用し、`border` との併用を避ける。

---

## 出力と実行

```bash
# Step 1: graph データを生成
python build_species_graph.py \
  --train-csv     data/train.csv \
  --soundscape-csv data/train_soundscapes_labels.csv \
  --out-dir       outputs/species_graph \
  --min-edge-count 1

# Step 2: HTML を生成
python visualize_species_graph.py \
  --graph-dir  outputs/species_graph \
  --output-html outputs/species_graph/species_graph.html \
  --max-edges  300 \
  --color-by   weighted_degree
```

主要オプション:

| オプション | 説明 |
|---|---|
| `--max-edges` | 描画するエッジ上限（`cooccur_count` 降順で選択） |
| `--min-edge-count` | この値未満の共出現エッジを除外 |
| `--color-by` | ノード色の根拠にする列名（nodes CSV の任意数値列） |

---

## CDN vs インライン埋め込み

| 方式 | メリット | デメリット |
|---|---|---|
| CDN 参照（採用） | ファイルサイズ小、常に最新 | オフライン環境で動かない |
| インライン埋め込み | 完全オフライン動作 | ファイルが数 MB 超になる |

ローカル分析用途ではネットワーク接続が前提のため CDN を選択。  
完全オフラインが必要な場合は `d3.v7.min.js` をダウンロードして `<script src="./d3.v7.min.js">` に差し替える。

---

