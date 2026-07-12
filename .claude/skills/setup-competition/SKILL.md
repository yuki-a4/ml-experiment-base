---
name: setup-competition
description: 新しいコンペを始めるときの初期セットアップ — データ取得、CLAUDE.md へのコンペ概要記入、視覚的な解説 HTML の作成、GitHub Issue ラベル作成、exp0001 着手までの案内を行う。コンペ名/URL/slug を引数に取る（未指定ならユーザーに確認）。
---

# コンペ開始時のセットアップ

引数: `<コンペ名 / URL / kaggle の `-c` slug>`

**引数が無ければ、まずユーザーに聞く**（推測で進めない）。

このリポジトリでの実験は CLAUDE.md の運用方針に従う。セットアップ未完了（コンペ概要が未記入・データが無い・ラベルが無い）のまま実験を始めない。

## 1. data/ にデータを取得する

**Kaggle のコンペで、API が使え認証済み（`kaggle.json` あり）なら**:

```bash
kaggle competitions download -c <competition-slug> -p data/
cd data && unzip -o '*.zip' && cd ..
```

**それ以外（Kaggle 以外のプラットフォーム — SIGNATE / atmaCup / Zindi / DrivenData 等、または Kaggle API が未インストール・未認証）は CLI で自動取得できないことが多い。無理に自動化せずユーザーと連携する**: CLI が使えない・対応していない旨を伝えた上で、(a) ブラウザから手動でダウンロードして `data/` に置いてもらう、(b) `kaggle.json` のセットアップを依頼する、(c) データの URL/入手方法を教えてもらう、のいずれかを相談する。取得した生データはコミットしない（`.gitignore` 済み）。

## 2. コンペ概要を CLAUDE.md に記入する

CLAUDE.md の「## コンペ概要」節を、**事実のみ**で埋める。考察・仮説（「〜が効きそう」等）は書かない — それは Issue や KNOWLEDGE.md の役割。

- **コンペ**: 名称 / URL / `-c` の slug
- **タスク種別**: 二値分類 / 多クラス分類 / 回帰 / ...
- **評価指標**: AUC / RMSE / LogLoss / Accuracy / ...（CV はこの指標で測る）
- **target 列**: 列名 / 予測対象
- **データ**: train/test の行数・主なカラム・特徴的な点
- **期限 / 提出制限**: 締切・1 日あたりの提出上限

情報源は Kaggle のコンペページ（Overview / Data / Evaluation タブ、WebFetch で取得可）や `data/` に取得したファイルの中身。不明な項目（特に評価指標・target 列は CV 設計に直結するため曖昧にしない）はユーザーに確認する。

## 3. コンペ解説 HTML を作る

コンペの中身が一目でわかる解説ページを 1 枚の HTML として作り、`docs/competition_overview.html` に保存する（git 追跡・以後の再セットアップで上書き更新してよい）。作る前に **dataviz skill を読み込んで**方針に従う（配色・グラフ選定・レスポンシブ対応など）。

含める内容の例（コンペの性質に応じて取捨選択する）:

- コンペ概要（タスク・評価指標・target・期限を短くまとめたテキスト）
- target の分布（分類ならクラス別件数、回帰ならヒストグラム）
- train/test の基本統計（行数・列数・型・欠損率）
- 主要な特徴量の分布や target との関係が分かる図（多すぎる場合は代表的なものに絞る）
- 気づいた特徴的な点（外れ値・不均衡・欠損パターンなど）があれば一言添える

`data/` を実際に読み込んで（pandas 等）作図し、画像は外部ファイルに依存させず HTML に埋め込む（base64 データ URI など）。`file://` で直接開いて完結するようにする。

## 4. GitHub Issue ラベルを用意する

テンプレートから作った新規リポジトリにはラベルがコピーされないため、毎回作り直す（`--force` なので重複実行しても安全）。GitHub 連携が無効（リモート未設定 / `gh` 未認証）ならこのステップは飛ばしてよい。

```bash
gh label create experiment --color 1D76DB --description "実験（submissionを生む）" --force
gh label create eda        --color 0E8A16 --description "分析（EDA）" --force
gh label create survey     --color 5319E7 --description "調査（外部/内部の調べもの）" --force
gh label create "priority:critical" --color B60205 --description "優先度：最重要・緊急" --force
gh label create "priority:high"     --color D93F0B --description "優先度：高" --force
gh label create "priority:medium"   --color FBCA04 --description "優先度：中" --force
gh label create "priority:low"      --color 0E8A16 --description "優先度：低" --force
gh label create "priority:backlog"  --color C5DEF5 --description "優先度：バックログ（着手未定のアイデア）" --force
```

## 5. exp0001（ベースライン）から実験を始める

セットアップ完了後は通常の実験ライフサイクル（CLAUDE.md「実験のライフサイクル」節）に従う: まず Issue を立ててから `cp -r exp/_template exp/exp0001` で着手する。ここから先はこの skill の範囲外。
