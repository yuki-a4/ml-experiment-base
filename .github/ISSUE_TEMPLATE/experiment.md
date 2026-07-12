---
name: 実験
about: 実験の着手前に仮説と計画を宣言する
title: 'expNNNN: <一言でわかる仮説>'
labels: experiment
assignees: ''
---

<!-- 以降のコメントも冒頭または末尾に発言者を記名すること（例: — Claude Opus 4.8） -->

## 起票者
<記録者。LLM なら自身のモデル名（例: Claude Opus 4.8 / GPT-5）、人間なら名前>

## 仮説
<なぜこの実験をするのか。何が改善すると考えるか。>

## アプローチ
- **タスク/指標**: <分類 or 回帰 / 評価指標>
- **モデル**: <LightGBM / XGBoost / ...>
- **CV 戦略**: <StratifiedKFold, n_splits=5, seed=42 など>
- **特徴量/前処理**: <追加・変更する予定のもの>
- **ベース実験**: <expNNNN / なし>

## 検証したいこと
<この実験で確かめたい問い。成功と判断する基準（例: CV が exp0003 を上回る）。>

## 完了条件
- [ ] `exp/expNNNN/output/` に submission と OOF を出力
- [ ] `exp/expNNNN/REPORT.md` を記入
- [ ] `EXPERIMENT.md` に 1 行追記
- [ ] 得られた知見を `KNOWLEDGE.md` に反映（あれば）
