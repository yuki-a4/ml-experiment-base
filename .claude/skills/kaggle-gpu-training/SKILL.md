---
name: kaggle-gpu-training
description: Kaggle Notebooks の GPU(T4×2)で学習ジョブを実行するときの手順とハマりどころ集。コードの Dataset 化 → カーネル push → 監視 → 成果物回収までの定型フロー(テンプレート同梱・コンペ非依存)。ユーザーから「Kaggle の計算リソースで学習して」と指示されたら必ずこれを読む。
---

# Kaggle GPU での学習実行(コンペ非依存の定型フロー)

どのコンペ・どのリポジトリでも使える前提で書いてある。プレースホルダは
`<USER>`(kaggle ユーザー名。`kaggle config view` で確認)、`<COMP_SLUG>`(コンペ slug)、
`<JOB>`(実験や学習ジョブの識別子。例 exp 番号)。

## 全体フロー

1. **コードを自己完結・両対応にする**
   - データ/出力/中間生成物のパスは**環境変数で上書き可能**にする(例: `DATA_ROOT` / `OUT_DIR` / `CACHE_DIR`)。既定値はローカルパス、Kaggle 上(`/kaggle` が存在)なら自動で Kaggle 側パスに切り替え。
   - device は `cuda if available else cpu` で自動判定にし、**push 前に必ずローカル CPU スモークテスト**(数枚×1〜2epoch で学習ループ・checkpoint resume・推論経路)を通す。Kaggle 上でのデバッグは 1 往復 10 分以上かかるので、ローカルで潰せるバグはローカルで潰す。
2. **コードを Kaggle Dataset にする**(カーネルにコードを渡す標準手段)
3. **カーネルを push**(code_file は小さな runner、本体はオーケストレータ)
4. **試走(SMOKE)→ 本走**: いきなり本走しない。数十枚×2epoch の SMOKE 走行で GPU 割当・データ検出・VRAM・ログ経路を確認してから本走を push する。
5. **監視**: `kaggle kernels status` の定期ポーリング(5分間隔で十分)。**実行中のログ・出力は取得できない**(完了後のみ)。
6. **回収**: `kaggle kernels output <USER>/<slug> -p <dir>`。ログは JSON 行形式(`[{"stream_name":...,"data":...}]`)なのでパースして読む。

## テンプレート

### dataset-metadata.json(コード配布用 Dataset)

```json
{
  "title": "<JOB> src",
  "id": "<USER>/<JOB>-src",
  "licenses": [{"name": "CC0-1.0"}]
}
```

push(ステージングディレクトリに `src_<JOB>/` を作り *.py 等を入れる):

```bash
STAGE=$(mktemp -d); mkdir -p "$STAGE/src_<JOB>"
cp src/*.py "$STAGE/src_<JOB>/"; cp dataset-metadata.json "$STAGE/"
if kaggle datasets status <USER>/<JOB>-src 2>/dev/null | grep -qi ready; then
  kaggle datasets version -p "$STAGE" -m "update" --dir-mode zip
else
  kaggle datasets create -p "$STAGE" --dir-mode zip
fi
```

### kernel-metadata.json

```json
{
  "id": "<USER>/<JOB>-train",
  "title": "<JOB> train",
  "code_file": "runner.py",
  "language": "python",
  "kernel_type": "script",
  "is_private": "true",
  "enable_gpu": "true",
  "enable_internet": "true",
  "machine_shape": "NvidiaTeslaT4",
  "dataset_sources": ["<USER>/<JOB>-src"],
  "competition_sources": ["<COMP_SLUG>"],
  "kernel_sources": []
}
```

### runner.py(カーネル本体。設定は env 埋め込みで渡す — metadata に env 指定は無い)

```python
import glob, os, shutil, subprocess, sys
os.environ.setdefault("SMOKE", "0")          # push 時に書き換えて渡す
os.environ.setdefault("FOLDS", "0,1,2,3,4")
cands = glob.glob("/kaggle/input/**/run_all.py", recursive=True)  # 本体名で探す
if not cands:
    for root, dirs, files in os.walk("/kaggle/input"):
        print(root, "->", dirs, files[:5])   # 失敗時に一発診断できるよう一覧を出す
    raise SystemExit("code dataset が見つからない")
shutil.copytree(os.path.dirname(cands[0]), "/kaggle/working/src", dirs_exist_ok=True)
sys.exit(subprocess.run([sys.executable, "/kaggle/working/src/run_all.py"]).returncode)
```

### オーケストレータ(run_all.py)に入れるべき機能

- 依存の pip install(import 失敗時のみ。バージョン固定)
- **前処理キャッシュは /tmp に生成**(/kaggle/working に置くと出力に含まれ回収が重くなる)
- **resume**: 毎 epoch last ckpt(model/ema/optimizer/epoch)を保存。起動時に `/kaggle/input/*/models/*.pt` を glob して前セッションの ckpt を回収(前セッション出力を Dataset 化して attach すれば自動再開になる)
- **GPU 分配**: T4×2 は DDP より「CUDA_VISIBLE_DEVICES を分けた subprocess で fold 並列」が単純で失敗しにくい。vCPU は 4 コアなので DataLoader workers はプロセスあたり 2 程度
- **時間ガード**: 完了した fold の実測所要を記録し、「残り時間 < 実測所要×1.1」なら新規 fold を開始しない(固定閾値だと走行中 fold の 9h 超過を防げない)
- fold ごとのログをファイルに書き、**最後に必ず標準出力へ tail を流す**(これをしないと完了後にログが読めない)
- SMOKE env で「枚数 limit+短 epoch」の試走モードに切り替えられるようにする

## ハマりどころ(実際に踏んだ失敗集)

- **【最重要】GPU 既定割当は P100 で、現行 PyTorch(sm_70+ ビルド)では CUDA が動かない**。`enable_gpu` だけでは P100 が来る。**`"machine_shape": "NvidiaTeslaT4"` を必ず指定**(許容値: `NvidiaTeslaT4` / `NvidiaTeslaP100` / `Tpu1VmV38`)。
- **Dataset push 直後に kernel push すると zip 展開が未完了でマウントされず即死**。`kaggle datasets status` が `ready` になるのを待ち、さらに 30 秒置いてから push する。
- **競技データのマウントパスを決め打ちしない**。`/kaggle/input/<COMP_SLUG>/` を第一候補に、無ければ `/kaggle/input` 以下を既知のファイル名で再帰 glob。見つからない場合は実際のマウント一覧を例外メッセージに含める。
- **9h でセッション強制終了。強制終了されると /kaggle/working の出力は保存されない(全損)**。上記の resume+時間ガード+セッション分割で防御する。
- **セッションまたぎで出力は自動で引き継がれない**。前セッションの出力を `kernels output` で回収 → Dataset 化 → 次 push の `dataset_sources` に追加、が確実な経路。
- スクリプトカーネルには無いライブラリがある(pip 必要)。事前学習重みのダウンロードも含め `enable_internet: "true"` にする。
- ステータス文字列は `KernelWorkerStatus.RUNNING / COMPLETE / ERROR` 形式(**大文字**)。小文字 glob でマッチさせようとすると監視ループが終わらない。

## 制約・クォータ(2026-07 時点の実測)

- GPU クォータ: 約 30h/週。T4×2 は 2 GPU 同時でも 1 セッション分の消費。
- 1 セッション上限: 9h(超過は強制終了・出力全損)。
- T4 は 15GB VRAM/枚(参考実測: 53M param の U-Net 系・768² crop・bs4・AMP fp16 で 6.4GB)。
- 出力(/kaggle/working)は 20GB まで。ただし数 GB でも回収に数十分かかるので出力は最小限に。

## 監視のテンプレ

```bash
# 完了/エラーまで5分間隔ポーリング(バックグラウンド実行向き)
for i in $(seq 1 120); do
  st=$(kaggle kernels status <USER>/<slug> 2>&1)
  case "$st" in *COMPLETE*|*ERROR*|*CANCEL*) echo "$st"; exit 0;; esac
  sleep 300
done
```
