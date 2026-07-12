# expNNNN: <一言でわかる仮説/タイトル>

- **種別**: 実験 / 分析(EDA) / 調査  <!-- どれか1つ。実験以外は CV/LB を「-」にしてよい -->
- **起票者**: <記録者。LLM なら自身のモデル名（例: Claude Opus 4.8 / GPT-5）、人間なら名前>
- **Issue**: #<番号>（GitHub 連携がない場合は「-」）
- **日付**: YYYY-MM-DD
- **開始時 commit**: <git rev-parse --short HEAD の値>  <!-- 再現性のため実験開始時のハッシュを残す -->
- **ベース実験**: expNNNN / なし

## 仮説
<なぜこの実験をするのか。何が改善すると考えたか。>

## 手法
- **タスク/指標**: <分類 or 回帰 / 評価指標>
- **モデル**: <LightGBM / XGBoost / ... とバージョン>
- **CV 戦略**: <StratifiedKFold, n_splits=5, seed=42 など再現に必要な情報>
- **特徴量/前処理**: <追加/変更した特徴量、fold 内 fit の有無>
- **主なハイパーパラメータ**: <学習率、深さ、n_estimators など>

## 結果
| 指標 | 値 |
|------|----|
| CV   | <スコア（± 標準偏差）> |
| Public LB | <スコア / 未提出> |

### 診断プロット
<!-- figures/ に保存した図を埋め込む。例: 特徴量重要度・混同行列・予測/実測分布・クラス別 recall など -->
![feature importance](figures/feature_importance.png)
![confusion matrix](figures/confusion_matrix.png)

## 考察
<仮説は支持されたか。CV-LB の乖離。図から読み取れる気づき（誤分類の偏り・分布シフトなど）。>

## 次アクション
<この結果を受けて次に試すこと。新しい Issue の種。>
