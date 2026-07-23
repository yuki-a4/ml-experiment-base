---
name: kaggle-tpu-training
description: Kaggle Notebooks の TPU(v3-8)で PyTorch/torch_xla の学習ジョブを実行するときの手順とハマりどころ集。チップ分割 fold 並列・bf16・XLA 移植の要点(コンペ非依存)。ユーザーから「TPU で学習して」と指示されたとき、または重い学習ジョブを Kaggle に流すときに読む。共通フロー(コード Dataset 化・push・監視・回収)は kaggle-gpu-training を併読すること。
---

# Kaggle TPU (v3-8) での学習実行(コンペ非依存)

**前提**: カーネル運用の共通フロー(コードの Dataset 化 → runner → SMOKE → 本走 → 監視 → 回収、
Dataset ready 待ち、9h 強制終了 = 出力全損、resume 設計)は `kaggle-gpu-training` skill に従う。
この skill はその **TPU 差分**のみを扱う。

## いつ TPU を選ぶか(資源配分の原則)

- TPU の割当(QUEUED)待ちは **20〜80 分で時間帯によりばらつき大**(GPU T4 はほぼ待ちなし)。
  **QUEUED はクォータを消費しない**(課金は実走行分のみ。ただし RUNNING 中はセッション上限までの
  時間が「Reserved」として先取り表示され、終了時に実走行分だけ確定する)。クォータは週次リセット。
  待ちは固定コストなので、**一発で長く走る重いジョブ(本走級の学習)だけを TPU に流す**。
  デバッグ・軽い検証の往復は GPU/ローカルで済ませてから持ち込む。
- 性能の目安(v3-8 実測、53M param U-Net 系・768²): 1 コアで T4 の 2〜4 倍(bf16・大きめ bs)、
  チップ分割 4 並列でほぼ線形 = **T4×2 の約 8 倍**。GPU 週30h と別枠の週 ~20h なので、
  開通すると実効計算力が大きく増える。
- TPU 系列は bf16/XLA の数値差があるため、**学習も評価も TPU トラック内で完結**させ、
  GPU 学習モデルとの厳密比較には使わない。

## ゲート付き本走(QUEUE 待ちの償却パターン)

TPU は「SMOKE→結果確認→本走」と分けると QUEUE 待ち(20〜80分)を2回払うことになる。
新しい encoder・設定を初めて走らせるときは、**同一カーネル内で「ゲート(数十枚×2epoch の短縮走行)
→ rc=0 なら同じ設定で本走を自動開始、失敗ならログを出して中止」**という構成にすると、
待ちは1回で済み、かつ本走が壊れた設定で数時間走る丸損も防げる。ゲートの checkpoint は
本走の出力を汚さないよう最後に削除する。

## kernel-metadata(TPU 差分)

```json
{
  "enable_gpu": "false",
  "enable_tpu": "true",
  "machine_shape": "Tpu1VmV38"
}
```

## ハードの実態(v3-8)

- 4 チップ × 2 コア = 8 コア。HBM は **1 コアあたり 16GB**(合計 128GB は 1 プールではない)。
- XLA はテンソルをタイル形状(8×128)にパディングするため、**実効メモリは GPU 比 2〜3 割厳しい**。
  fp32 では GPU fp16 と同じ bs が OOM しがち → **bf16 前提**で設計する。
- ホスト CPU は非常に潤沢(数十 vCPU)。DataLoader workers は多め(プロセスあたり 8)でよい。

## 並列化: 「チップ分割の独立ジョブ並列」を第一選択にする

fold 学習・seed 違い・ハイパラ違いなど**通信不要のジョブ群**なら、協調分散(データ並列)より
チップ分割が速度(線形スケール)・単純さ・安定性のすべてで優る。

- 各ジョブを別サブプロセスとして起動し、次の env でチップに pin するだけ:
  ```python
  env = dict(os.environ, TPU_PROCESS_BOUNDS="1,1,1",
             TPU_CHIPS_PER_PROCESS_BOUNDS="1,1,1", TPU_VISIBLE_CHIPS=str(chip))  # chip=0..3
  ```
- **親(オーケストレータ)プロセスでは torch_xla を import しない**(XLA クライアントが
  チップ pin 前に初期化されると子が全滅する)。
- `xmp.spawn` / `torch_xla.launch` の協調分散は fork と XLA クライアント初期化の衝突
  (`InitializeComputationClient() can only be called once`)で壊れやすい。start_method='spawn'
  明示でも Kaggle 環境では再現。独立ジョブ並列で足りるなら深追いしない。
- この方式では 1 プロセス = 1 コアで、チップ内のもう 1 コアは遊ぶ(スレッド並列で 2 コア使う
  最適化は可能だが複雑。まず 4 並列で回してから検討)。

## torch_xla 移植の要点(PyTorch 学習ループの差分)

- device は `xm.xla_device()`。step 後に `torch_xla.sync()`(旧 `xm.mark_step()`)。
- **毎 step の `.item()` / `.cpu()` は同期を強制して激遅**。loss はデバイス上のテンソルに積算し、
  epoch 末に 1 回だけ `.item()` する。進捗 print も epoch 単位にする。
- bf16 は `torch.autocast("xla", dtype=torch.bfloat16)`(env `XLA_USE_BF16` は deprecated)。
  GradScaler は不要。
- checkpoint 保存は `xm.save(obj, path)`(XLA→CPU 転送込み)。plain `torch.save` は XLA テンソルに使わない。
  load は `torch.load(map_location="cpu")` → `load_state_dict`。
- XLA はコンパイル型: 初回 step が 50〜70 秒(コンパイル)、以降キャッシュ。**入力形状は固定**に保つ
  (可変形状・分岐はコンパイル爆発の元。crop 学習・固定タイル推論は好相性)。
- 同一カーネル内の複数フェーズで TPU を使う場合、フェーズごとにサブプロセス分離すると
  初期化状態の持ち越しによる事故を防げる。

## 環境の罠

- **TPU VM イメージは GPU イメージとパッケージ構成が違う**(例: pycocotools が無い)。
  依存は「import 試行 → 欠けたものだけ pip」方式で起動時に自動補完する。
- torch は `+cpu` ビルド(CUDA なし)。`torch.cuda.is_available()` に依存するコードは
  XLA 分岐を明示すること。
