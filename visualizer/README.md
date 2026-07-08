# visualizer/

実験結果を可視化するための**共通ビジュアライザー**を置く場所です。

- **実験ごとに HTML を作らない**。各 `exp/expNNNN/` はデータ（`figures/viz_data.json`）だけを出力し、可視化は常にここのツールで行う。
- 見せ方（ダッシュボード形式、比較特化、時系列特化など）は複数あってよいので、**1 手法 = 1 ディレクトリ**とする。

```
visualizer/
├── README.md
└── dashboard/          # 標準のビジュアライザー（実験横断の比較ダッシュボード）
    ├── index.html       # これを開く（ビルド不要・単一ファイル）
    └── viz_data.example.json  # データ契約のサンプル
```

## 使い方

1. `visualizer/dashboard/index.html` をブラウザで直接開く（`file://` で OK。サーバ起動不要）。
2. 比較したい実験の `exp/expNNNN/figures/viz_data.json` をドラッグ＆ドロップ、またはファイル選択で読み込む（複数選択可）。
3. チェックボックスで比較対象を選び、スライダーで表示件数などを調整しながら見る。

新しい見せ方が欲しくなったら `visualizer/<name>/` を新設してよい（既存の `dashboard/` は変更せず併存させる）。

## データ契約: `viz_data.json`

各実験は `exp/expNNNN/figures/viz_data.json` にこのビジュアライザーが読める要約データを出力する（`figures/` は git 追跡対象なので、クローンした直後でも比較できる）。フォーマットは `dashboard/viz_data.example.json` を参照。フィールドは基本すべて任意 — 無いものはその部分の表示がスキップされるだけなので、そのコンペで意味のある項目だけ埋めればよい。
