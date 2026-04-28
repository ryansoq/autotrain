# autoresearch

這是一個讓 LLM 自己做研究的實驗。

## 設定

要設定一個新的實驗，請與使用者一起：

1. **約定一個 run tag**：根據今天的日期提一個 tag（例如 `mar5`）。`autoresearch/<tag>` 這個 branch 必須尚未存在 — 這是一次全新的 run。
2. **建立 branch**：從目前的 master `git checkout -b autoresearch/<tag>`。
3. **讀取在範圍內的檔案**：repo 很小。讀以下檔案以取得完整背景：
   - `README.md` — repo 背景。
   - `prepare.py` — 固定常數、資料準備、tokenizer、dataloader、評估。**不要修改。**
   - `train.py` — 你會修改的檔案。模型架構、optimizer、訓練迴圈。
4. **確認資料存在**：檢查 `~/.cache/autoresearch/` 是否包含資料 shards 和 tokenizer。如果沒有，請告訴使用者執行 `uv run prepare.py`。
5. **初始化 results.tsv**：建立只有 header row 的 `results.tsv`。baseline 會在第一次 run 之後紀錄。
6. **確認後開始**：確認設定看起來沒問題。

得到確認後，就開始實驗。

## 實驗

每個實驗在單一 GPU 上執行。訓練腳本會以**固定 5 分鐘的時間預算**執行（牆上時鐘訓練時間，不含啟動／編譯）。你只要這樣啟動：`uv run train.py`。

**你可以做的事：**
- 修改 `train.py` — 這是你唯一會編輯的檔案。所有東西都是公平遊戲：模型架構、optimizer、超參數、訓練迴圈、batch size、模型大小等等。

**你不可以做的事：**
- 修改 `prepare.py`。它是唯讀的。它包含固定的評估、資料載入、tokenizer 與訓練常數（時間預算、序列長度等）。
- 安裝新套件或新增相依。你只能用 `pyproject.toml` 裡已有的。
- 修改評估 harness。`prepare.py` 裡的 `evaluate_bpb` 函式是 ground truth 指標。

**目標很簡單：取得最低的 val_bpb。** 因為時間預算是固定的，你不需要擔心訓練時間 — 永遠都是 5 分鐘。所有東西都是公平遊戲：改架構、改 optimizer、改超參數、改 batch size、改模型大小。唯一的限制是程式必須能執行不 crash，並在時間預算內結束。

**VRAM** 是軟性限制。為了有意義的 val_bpb 進步，一些增加是可以接受的，但不應該爆炸性地增加。

**簡潔性原則**：在其他條件相同時，越簡單越好。一點點進步但加上醜陋複雜的程式碼是不值得的。反之，**移除東西**而得到相等或更好的結果，是個很棒的成果 — 那是一次簡化的勝利。當在評估是否要保留某個改動時，請衡量複雜度成本與改進幅度。0.001 val_bpb 的進步但加了 20 行 hack code？大概不值得。0.001 val_bpb 的進步來自於**刪掉**程式碼？絕對保留。改進約為 0 但程式碼簡潔很多？保留。

**第一次 run**：你的第一次 run 永遠應該是建立 baseline，因此你會原封不動地執行訓練腳本。

## 輸出格式

腳本完成後會印出像這樣的摘要：

```
---
val_bpb:          0.997900
training_seconds: 300.1
total_seconds:    325.9
peak_vram_mb:     45060.2
mfu_percent:      39.80
total_tokens_M:   499.6
num_steps:        953
num_params_M:     50.3
depth:            8
```

注意腳本被設定為永遠在 5 分鐘後停止，所以根據這台電腦的計算平台不同，數字看起來可能會不一樣。你可以從 log 檔案中萃取關鍵指標：

```
grep "^val_bpb:" run.log
```

## 紀錄結果

當一個實驗完成後，把它紀錄到 `results.tsv`（**tab 分隔，不是逗號**分隔 — 描述裡的逗號會壞掉）。

TSV 有一個 header row 和 5 個欄位：

```
commit	val_bpb	memory_gb	status	description
```

1. git commit hash（短的，7 個字元）
2. 達到的 val_bpb（例如 1.234567）— crash 時用 0.000000
3. peak memory，單位 GB，四捨五入到 .1f（例如 12.3 — 由 peak_vram_mb 除以 1024）— crash 時用 0.0
4. status：`keep`、`discard`、或 `crash`
5. 這次實驗嘗試了什麼的簡短文字描述

範例：

```
commit	val_bpb	memory_gb	status	description
a1b2c3d	0.997900	44.0	keep	baseline
b2c3d4e	0.993200	44.2	keep	increase LR to 0.04
c3d4e5f	1.005000	44.0	discard	switch to GeLU activation
d4e5f6g	0.000000	0.0	crash	double model width (OOM)
```

## 實驗迴圈

實驗在一個專屬的 branch（例如 `autoresearch/mar5` 或 `autoresearch/mar5-gpu0`）上執行。

LOOP FOREVER：

1. 看一下 git 狀態：目前在哪個 branch／commit
2. 用一個實驗想法調整 `train.py`，直接 hack code。
3. git commit
4. 執行實驗：`uv run train.py > run.log 2>&1`（**全部都重導向** — 不要用 tee 或讓輸出灌進你的 context）
5. 讀取結果：`grep "^val_bpb:\|^peak_vram_mb:" run.log`
6. 如果 grep 輸出是空的，表示這次 run crash 了。執行 `tail -n 50 run.log` 讀 Python stack trace 並嘗試修復。如果嘗試了幾次後還是不能 work，就放棄。
7. 把結果紀錄到 tsv（**注意：不要 commit results.tsv 這個檔案，讓它對 git 保持 untracked**）
8. 如果 val_bpb 進步了（變低），你就「推進」branch，保留這個 git commit
9. 如果 val_bpb 持平或變差，你就 git reset 回到你開始的地方

概念是你是一個完全自主的研究者在嘗試各種東西。work 的就保留，不 work 的就丟掉。然後你會推進這個 branch 來持續迭代。如果你覺得卡住了，可以倒帶，但這應該非常非常少見（如果有的話）。

**Timeout**：每個實驗應該總共花約 5 分鐘（加上幾秒的啟動和 eval overhead）。如果一次 run 超過 10 分鐘，就終止它，視為失敗（discard 並 revert）。

**Crash**：如果一次 run crash（OOM、bug 等等），用你的判斷力：如果是某個蠢且容易修的問題（例如 typo、缺 import），就修一下重跑。如果這個想法本身就壞掉了，就跳過它，在 tsv 裡 log 為 "crash"，然後繼續。

**永遠不要停**：一旦實驗迴圈開始（在初始設定之後），**不要**停下來問人類是否該繼續。**不要**問「我該繼續嗎？」或「現在是好的停止點嗎？」。人類可能在睡覺，或離開電腦，他們期待你**無限期地**繼續工作直到被手動停止。你是自主的。如果你沒點子了，想得更深 — 讀 code 中參考的論文、重讀範圍內的檔案找新角度、嘗試組合先前差一點成功的東西、嘗試更激進的架構改動。這個迴圈會一直跑，直到人類打斷你為止，就這樣。

舉個使用情境：使用者可能在睡覺時讓你跑著。如果每個實驗大約花你 5 分鐘，你一小時可以跑大約 12 次，平均人類的睡眠時間總共可以跑大約 100 次。使用者醒來時就會看到你在他睡覺時完成的所有實驗結果！
