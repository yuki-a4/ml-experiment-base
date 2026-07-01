# CLAUDE.md

機械学習コンペのための実験リポジトリです。
このファイルは実験を進める上での運用方針を定めます。**実験に関わる作業では必ずこの方針に従ってください。**

> **コンペ開始時にやること**
> 1. 下の「コンペ概要」を埋める（コンペ名・タスク種別・評価指標・target 列など）。
> 2. `data/` にデータを取得する（例: `kaggle competitions download -c <competition>`）。
> 3. Issue ラベルを用意する（テンプレートからの新リポジトリにはラベルがコピーされないため）:
>    ```
>    gh label create experiment --color 1D76DB --description "実験（submissionを生む）" --force
>    gh label create eda        --color 0E8A16 --description "分析（EDA）" --force
>    gh label create survey     --color 5319E7 --description "調査（外部/内部の調べもの）" --force
>    gh label create "priority:high"   --color B60205 --description "優先度：高" --force
>    gh label create "priority:medium" --color FBCA04 --description "優先度：中" --force
>    gh label create "priority:low"    --color 0E8A16 --description "優先度：低" --force
>    ```
> 4. exp0001（ベースライン）から実験を始める。

## コンペ概要

> **この節が未記入の場合は、実験を始める前に必ず記入すること**（指標や target が曖昧なまま実験すると CV が比較不能になる）。
>
> **事実のみを書く**。考察・気づき・仮説（「〜が効きそう」等）は書かない —— やってみたいアイデアは Issue へ、検証で確定した知見は KNOWLEDGE.md へ、分析の結論は EDA の REPORT.md へ回す（概要は不変の事実だけに保つ）。

- **コンペ**: <名称 / URL / `-c` の slug>
- **タスク種別**: <二値分類 / 多クラス分類 / 回帰 / ...>
- **評価指標**: <AUC / RMSE / LogLoss / Accuracy / ...>（CV はこの指標で測る）
- **target 列**: <列名 / 予測対象>
- **データ**: <train/test の行数・主なカラム・特徴的な点>
- **期限 / 提出制限**: <締切 / 1日あたりの提出上限>

## リポジトリ構成

```
.
├── CLAUDE.md          # このファイル（運用方針）
├── EXPERIMENT.md      # 全実験の結果を表形式で記録する台帳
├── KNOWLEDGE.md       # 実験から得られた普遍的・重要な知見の蓄積
├── .github/
│   └── ISSUE_TEMPLATE/
│       └── experiment.md   # 実験 Issue のテンプレート（GitHub 機能）
├── data/              # コンペのデータ（train/test/sample_submission など）
└── exp/
    ├── _template/     # 新実験の雛形。これをコピーして expNNNN を作る
    │   ├── src/
    │   ├── output/
    │   ├── figures/
    │   └── REPORT.md  # REPORT のテンプレート
    ├── exp0001/
    │   ├── src/       # ソースコード
    │   ├── output/    # OOF・submission・中間生成物など（git 管理外）
    │   ├── figures/   # REPORT/Issue に載せる確定プロット（git 追跡）
    │   └── REPORT.md  # この実験の仮説・手法・結果のまとめ
    ├── exp0002/
    └── ...
```

- 新しい実験は `cp -r exp/_template exp/expNNNN` で作成し、コードを自己完結させる（共通コードのバージョン問題を避ける）。
- **採番前に最新の実験番号を必ず確認し、重複させない**。`ls exp/` で既存フォルダを見て最大番号 +1 を採番する（例: `ls -d exp/exp* | sort | tail -1` で最新を確認）。

- `data/` … データ置き場。生データはリポジトリにコミットしない（`.gitignore` 参照）。
- `exp/expNNNN/` … **1 フォルダ 1 テーマ**。番号は `exp0001` から連番で採番する。
  - `src/` … そのソースコード。**notebook(`.ipynb`)と `.py` は用途に応じて使い分けてよい**（探索的な EDA・可視化は notebook、再現性・再実行が要る学習/推論パイプラインは `.py` が向く）。どちらも `src/` に置く。
  - `output/` … OOF 予測、submission ファイル、学習過程で出力する中間生成物、探索中の雑多な画像など（**git 管理外**）。
  - `figures/` … REPORT/Issue に載せる**確定プロット**を保存する（**git 追跡**）。GitHub 上でレンダリングさせたい図はここに置く。
  - **exp は「実験」だけでなく「分析(EDA)」「調査」も対象**。submission を生まない作業（データの分布調査、CV-LB 乖離の原因調査、上位解法の比較調査など）も exp として残す。
    REPORT.md 冒頭の **`種別`** に `実験 / 分析(EDA) / 調査` を明記する。分析・調査では CV/LB 欄は「-」でよい。得られた結論は必ず KNOWLEDGE.md に反映する。

## 実験のライフサイクル（必ずこの順で行う）

1. **Issue を作成する（実験開始前に必須）**
   - **Issue 本文・コメントは Markdown で書く**。見出し・箇条書き・表・コードブロック（```）・チェックリスト・画像添付を活用し、読みやすく構造化する。スコアやパラメータは表、コードやログはコードブロックにする。
   - **作成前に既存の Issue を確認し、重複を作らない**。`gh issue list --state all --search "<キーワード>"` 等で open/closed を検索し、同じ内容の Issue が既にあればそれを使う（再開する場合は該当 Issue を reopen）。似た Issue があれば新規作成せずコメントで追記する。
   - GitHub の Issue に、実験番号・仮説・アプローチ・検証したいことを書いてから着手する。テンプレート: `.github/ISSUE_TEMPLATE/experiment.md`。
   - **種別ラベルを付ける**（REPORT の種別と一致）: 実験=`experiment` / 分析=`eda` / 調査=`survey`。
   - **優先度ラベルを付ける**: `priority:high` / `priority:medium` / `priority:low`。自律実験モードでは、まず高優先度の Issue から着手する。
     - さらに必要なら**他の軸のラベルを足してよい**（例: 状態 `blocked`、モデル `model:lgbm`、バックログ `idea` など）。ただし軸を増やしすぎず**最小限に保つ**こと。
   - `gh issue create --title "exp0001: <一言でわかる仮説>" --body "<仮説・手法・検証項目>" --label experiment`
   - **例外**: リポジトリが GitHub 上で管理されていない（リモート未設定 / `gh` 未セットアップ・未認証）場合は Issue の作成・クローズは不要。
     その場合は REPORT.md と EXPERIMENT.md への記録で代替する。GitHub 連携が有効なときは必ず Issue を運用すること。
2. **実験する**
   - `exp/expNNNN/src/` にコードを置き、`exp/expNNNN/output/` に成果物を出力する。
3. **REPORT.md を書く**
   - 実験終了後、`exp/expNNNN/REPORT.md`（`_template` の雛形ベース）に仮説・手法・結果・考察・次アクションをまとめる。Issue と同じ枠を埋める形で転記する。
4. **EXPERIMENT.md に追記する**
   - 下記フォーマットの表に 1 行追記する。
5. **KNOWLEDGE.md を更新する**
   - 後の実験に普遍的に役立つ知見・重要な発見があれば追記する。
6. **Issue に結果を書いて Close する**（GitHub 連携が有効な場合のみ）
   - `gh issue close <番号> --comment "<CV/LB スコア・結論・次アクション>"`

## 実験内容の決め方（ユーザー指示がない場合）

ユーザーから具体的な実験内容の指示がない場合は、**自律的に次の実験を決めて開始する**こと。

1. まず **GitHub の Issue** を確認し、着手可能で有効そうな実験（未着手の仮説・TODO）を探す。**優先度ラベル（`priority:high` → `medium` → `low`）の高いものから選ぶ**。
2. Issue に適当なものがなければ、**これまでの実験結果（EXPERIMENT.md / 各 REPORT.md / KNOWLEDGE.md）** を振り返り、
   スコア改善に効きそうな次の一手を検討する（例: 効いた特徴量の深掘り、CV-LB 乖離の調査、有望モデルのチューニング、アンサンブル）。
3. 決めた実験について**必ず先に Issue を作成**してから（上記ライフサイクル 1）着手する。

**ユーザーからの指定がない限り、1 セッション 1 実験とする。** 1 つの実験をライフサイクル（Issue → 実験 → REPORT → EXPERIMENT → KNOWLEDGE → Issue クローズ）まで完了させることに集中し、勝手に次の実験へ進まない。次に何をやるかは「次アクション」として REPORT.md / Issue に残す。複数実験をまとめて進めたい場合はユーザーが明示的に指示する。

**実験の途中で気づいたことや、次にやるべき実験のアイデアが見つかったら、その場で新しい Issue として書き残してよい**（実験バックログになる）。この場合も**既存 Issue と重複しないか確認してから**起票する。ただし現在の実験は中断せず完了させること（1 セッション 1 実験）。書き残した Issue が、次セッションで自律的に実験を選ぶときの候補になる。

## KNOWLEDGE.md の運用

KNOWLEDGE.md は毎セッション読む「索引」として薄く保つ（詳しくは KNOWLEDGE.md 冒頭のルール）。

- 1 知見 = 結論 1 行 + 出典リンク。詳細は各 `REPORT.md` にあるので**重複させない**。
- 載せる基準は「**次の実験の判断を変えるか**」。データを見れば分かる事実は載せない。
- 覆った知見は更新 or 削除して**刈り込む**。外部由来は出典 URL/名称を必ず併記。
- 肥大化したら本ファイルを目次にして `knowledge/*.md` に分割する（常時ロードは目次のみ）。

## submission ファイルの命名規則

- 必ず実験番号を含める。例: `exp/exp0001/output/submission_exp0001.csv`
- 複数バリエーションがある場合は接尾辞を付ける。例: `submission_exp0001_seed42.csv`

## Kaggle への提出（submit）

Kaggle API が使える場合は API 経由で提出してよい。ただし**自動で提出しない**。

- **必ず確認を挟む**: 提出前に実験結果（CV スコア・提出予定ファイル・想定される LB への影響など）をユーザーに伝え、**提出してよいか問う**。承認を得てから提出する。
- コマンド例: `kaggle competitions submit -c <competition> -f <submission.csv> -m "exp0001: <一言>"`（メッセージに exp 番号を入れる）。
- **1 日の提出上限**（多くのコンペで 5 回/日）に注意し、上限が近い場合はその旨も伝える。
- 提出したら、その結果（Public LB スコア）を EXPERIMENT.md の LB 欄と Issue に記録する。

## EXPERIMENT.md のフォーマット

`EXPERIMENT.md` は下記の表に 1 実験 1 行で追記していく。

| exp | Issue | 日付 | 仮説/概要 | モデル | 特徴量/前処理 | CV | Public LB | 結論 |
|-----|-------|------|-----------|--------|----------------|----|-----------|------|

- **CV と LB は必ず両方記録する**（CV-LB の乖離は重要な知見）。
- CV の分割方法（KFold/StratifiedKFold・fold 数・seed）を再現できるよう REPORT.md 側に明記する。

## REPORT.md に書く項目

- 仮説（なぜこの実験をするのか）
- 手法（モデル・特徴量・CV 戦略・ハイパーパラメータ）
- 結果（CV / Public LB スコア、**診断プロット**）
- 考察（仮説は支持されたか、想定外の挙動）
- 次アクション（この結果を受けて次に何を試すか）

### 診断プロットを残す（スコアだけで終わらせない）

スコアは要約に過ぎず、次の一手のヒントは図に出る。実験に応じて有用なプロットを生成し、`figures/` に保存して REPORT に埋め込み、Issue にも添付する。例（これに限らない）:

- **特徴量重要度**（gain / permutation / SHAP）
- **混同行列**（生 / 行正規化。どのクラスをどこへ誤るか）
- **予測 vs 実測の分布**、回帰なら**残差プロット**
- **クラス別 recall/precision**、**PR 曲線 / ROC 曲線**
- **予測確率のヒストグラム**、**キャリブレーションプロット**
- **CV fold ごとのスコアばらつき**、**学習曲線**（iteration vs metric）
- **train vs test の特徴量分布**（分布シフト・リーク検出）、**誤分類サンプルの傾向**

図から得た気づきは考察に言語化し、次アクション（新しい Issue）に繋げる。

## 運用ルール

- **再現性**: 乱数 seed を固定し、CV 分割・パラメータを REPORT.md に残す。
- **CV を信じる**: LB へのフィッティングを避け、まず信頼できる CV を設計する。
- **リーク注意**: 前処理・特徴量生成は fold 内で fit し、OOF で評価する。
- **成果物を残す**: OOF は必ず保存する（後段のアンサンブル/スタッキングで再利用する）。
- 既存の実験フォルダは上書きしない。改変する場合は新しい `expNNNN` を採番する。
- **git 管理下では適宜コミット・push する**。実験の区切り（コード追加時、REPORT/EXPERIMENT/KNOWLEDGE 更新時など）でこまめに push し、成果と記録をリモートに残す。
- **時間のかかる処理は進捗を出力する**: 学習・CV・探索など長時間かかる処理では、進捗（現在の fold / iteration、経過時間、可能なら ETA）を標準出力やログに出す（`tqdm`、fold ごとの print、開始時刻の記録など）。**残り時間の見込みを聞かれたら答えられる状態**にしておく。バックグラウンド実行する場合はログをファイルに残し、進捗と経過時間を確認できるようにする。

## 環境メモ

- Issue 運用には GitHub CLI (`gh`) が必要。未インストールの場合はセットアップと `gh auth login` を行うこと。
- Kaggle データの取得には Kaggle API (`kaggle`) を使う。認証情報 `kaggle.json` はコミットしないこと（`.gitignore` 済み）。
