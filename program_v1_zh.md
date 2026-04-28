# autoresearch (v1)

這是一個讓 LLM 自己做研究的實驗。

v1 在原始協定上加了三個紀律：每次 commit 必須附上 **hypothesis + prediction**、小幅進步要做**二次種子驗證**、每 N 次實驗做一次**週期性回顧**。目的是把一個快速的隨機漫步轉換成一個快速、可稽核、會自我修正的搜尋過程。

## 設定

要設定一個新的實驗，請與使用者一起：

1. **約定一個 run tag**：根據今天的日期提一個 tag（例如 `mar5`）。`autoresearch/<tag>` 這個 branch 必須尚未存在 — 這是一次全新的 run。
2. **建立 branch**：從目前的 master `git checkout -b autoresearch/<tag>`。
3. **讀取在範圍內的檔案**：repo 很小。讀以下檔案以取得完整背景：
   - `README.md` — repo 背景。
   - `prepare.py` — 固定常數、資料準備、tokenizer、dataloader、評估。**不要修改。**
   - `train.py` — 你會修改的檔案。模型架構、optimizer、訓練迴圈。
4. **確認資料存在**：檢查 `~/.cache/autoresearch/` 是否包含資料 shards 和 tokenizer。如果沒有，請告訴使用者執行 `uv run prepare.py`。
5. **初始化 `results.tsv`**：建立只有 header row 的 `results.tsv`。baseline 會在第一次 run 之後紀錄。untracked。
6. **初始化 `journal.md`**：建立一個空的 `journal.md`。它會在迴圈中累積週期性回顧。untracked。
7. **確認後開始**：確認設定看起來沒問題。

得到確認後，就開始實驗。**這是你最後一次問人類任何事。**

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

**簡潔性原則**：在其他條件相同時，越簡單越好。一點點進步但加上醜陋複雜的程式碼是不值得的。反之，**移除東西**而得到相等或更好的結果，是個很棒的成果 — 那是一次簡化的勝利。0.001 val_bpb 的進步但加了 20 行 hack code？大概不值得。0.001 val_bpb 的進步來自於**刪掉**程式碼？絕對保留。改進約為 0 但程式碼簡潔很多？保留。

**第一次 run**：你的第一次 run 永遠應該是建立 baseline，因此你會原封不動地執行訓練腳本。

## 假設紀律（v1 新增）

每次 commit 之前，commit message 在 subject 之後必須包含兩行：

```
Hypothesis: <一句話 — 這個改動會降低 val_bpb 的機制>
Prediction: <預期方向與幅度，例如「↓ 0.001–0.003」、「↓ 但 VRAM +10%」、「探索性，無強烈先驗」>
```

這幾乎是免費的，並且把每次 run 從隨機漫步升級為一次真正的實驗。失敗也變得有資訊量 — 你事後能分辨「想法錯」vs「執行錯」，回顧時也能挖掘這些。

如果你真的講不出機制，就寫 `Prediction: exploratory` — 但要把這類 commit 標記為探索性，這樣 journal 才能統計。

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
grep "^val_bpb:\|^peak_vram_mb:" run.log
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

## 小幅進步的二次種子驗證（v1 新增）

單一 run 把「真正的改進」與「seed 噪音」混在一起。歷史紀錄顯示，許多被保留的進步落在 0.001–0.002 區間 — 正好是 seed 噪音的尺度。如果不驗證，運氣好的 run 會被固化進 running-best branch，後續實驗就建在浮沙上。

**觸發條件**：一次 run 即將被標記為 `keep`，**且** `Δval_bpb < 0.002`（相對於目前最佳）。

**程序**：
1. 把 `train.py` 裡的 random seed 加 1（**其他什麼都不要改**）。
2. 重跑：`uv run train.py > run.log 2>&1`。
3. 計算兩次 run 的 val_bpb 平均。
4. 如果平均仍打敗先前最佳 → `keep`。否則 → `discard` 並 `git reset`。

兩次 run 都要寫進 `results.tsv`。在第二列的 description 後加 `(2-seed avg=X.XXXXXX)`。

`Δ ≥ 0.002` 的進步可以跳過這個 step — 進步幅度夠大，seed 噪音不太可能翻盤，多花 5 分鐘不划算。

## 週期性回顧（v1 新增）

每完成 **20 次實驗**，在迴圈的頂部執行一次 self-check。這是自主的 — **不要**停下來問人類，也**不要**跳過。

**程序**：
1. 讀 `results.tsv`。把最近 ~20 列按主題分組（架構 / optimizer / 超參數 / 正則化 / 資料 / 等）。
2. 在 `journal.md` 中追加一段（~5 行）涵蓋：
   - 目前最佳的 val_bpb。
   - 最近幾次 `keep` 的共同主題。
   - 已知的 dead end（只產生 `discard`/`crash` 的主題）。
   - 接下來要嘗試的 1–2 個未探索方向。
3. 立刻回到迴圈。下一次 tune 應該從 journal 的建議中挑題目。

這對抗局部最佳解 — 沒有它，迴圈很容易在最近一次帶來小贏的方向上一直磨，直到那條礦脈耗盡。

## 實驗迴圈

實驗在一個專屬的 branch（例如 `autoresearch/mar5` 或 `autoresearch/mar5-gpu0`）上執行。

LOOP FOREVER：

1. **（每第 20 次迭代）** 執行週期性回顧，追加到 `journal.md`。見上面「週期性回顧」。
2. 看一下 git 狀態：目前在哪個 branch／commit。
3. 用一個實驗想法調整 `train.py`，直接 hack code。如果 `journal.md` 非空，從中取題。
4. 為 commit message 寫好 `Hypothesis:` 和 `Prediction:` 兩行。見上面「假設紀律」。
5. 用 step 4 的訊息 `git commit`。
6. 執行實驗：`uv run train.py > run.log 2>&1`（**全部都重導向** — 不要用 tee 或讓輸出灌進你的 context）。
7. 讀取結果：`grep "^val_bpb:\|^peak_vram_mb:" run.log`。
8. 如果 grep 輸出是空的，表示這次 run crash 了。執行 `tail -n 50 run.log` 讀 Python stack trace 並嘗試修復。如果嘗試了幾次後還是不能 work，就 log `crash` 然後繼續。
9. 如果一次 run 總時間超過 10 分鐘，就終止它，視為失敗（`discard` 並 revert）。
10. 把結果紀錄到 `results.tsv`。
11. **（若即將 keep 且 Δval_bpb < 0.002）** 在做決定前先執行二次種子驗證。見上面「小幅進步的二次種子驗證」。
12. 如果 val_bpb 進步了（變低），就推進 branch — 保留這個 git commit。否則 `git reset` 回到你開始的地方。

概念是你是一個完全自主的研究者在嘗試各種東西。work 的就保留，不 work 的就丟掉。然後你會推進這個 branch 來持續迭代。如果你覺得卡住了，可以倒帶，但這應該非常非常少見（如果有的話）。

**Crash**：如果一次 run crash（OOM、bug 等等），用你的判斷力。簡單的修復（typo、缺 import）→ 修一下重跑。想法本身就壞掉 → log `crash`，繼續。

**永遠不要停**：一旦實驗迴圈開始（在初始設定之後），**不要**停下來問人類是否該繼續。**不要**問「我該繼續嗎？」或「現在是好的停止點嗎？」。人類可能在睡覺，或離開電腦，他們期待你**無限期地**繼續工作直到被手動停止。你是自主的。如果你沒點子了，想得更深 — 讀 code 中參考的論文、重讀範圍內的檔案找新角度、嘗試組合先前差一點成功的東西、嘗試更激進的架構改動。**journal 是你的朋友。** 這個迴圈會一直跑，直到人類打斷你為止，就這樣。

舉個使用情境：使用者可能在睡覺時讓你跑著。如果每個實驗大約花你 5 分鐘，你一小時可以跑大約 12 次，平均人類的睡眠時間總共可以跑大約 100 次。使用者醒來時就會看到你在他睡覺時完成的所有實驗結果！

## 與原始協定的差異總結

| # | 變更 | 位置 |
|---|---|---|
| 1 | 設定階段初始化 `journal.md` | 設定 §6 |
| 2 | 每次 commit 必須附 `Hypothesis:` + `Prediction:` 兩行 | 假設紀律 |
| 3 | 小幅進步（Δ < 0.002）做二次種子驗證 | 二次種子驗證 + 迴圈 §11 |
| 4 | 每 20 次實驗做一次週期性回顧 | 週期性回顧 + 迴圈 §1 |

其餘部分維持與原始協定相同。
