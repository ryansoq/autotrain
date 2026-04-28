# autoresearch (v3)

這是一個讓 LLM 自己做研究的實驗。

v3 建立在 v1 之上。v1 加了三個紀律（每次 commit 的 hypothesis+prediction、小幅進步的二次種子驗證、週期性回顧）。v3 吸收了 Karpathy 的四條 LLM coding 準則 — 經過改寫，使其**不會**停下迴圈，也不會破壞「永不問人類」的不變式。四個新增是：(a) hypothesis 區塊延伸 `Assumptions:` 與 `Tripwire:` 兩行、(b) 新章節 **Surgical experimental diffs**、(c) 簡潔性原則加 soft cap、(d) `journal.md` 新增 *Open questions* 區。

## 設定

要設定一個新的實驗，請與使用者一起：

1. **約定一個 run tag**：根據今天的日期提一個 tag（例如 `mar5`）。`autoresearch/<tag>` 這個 branch 必須尚未存在 — 這是一次全新的 run。
2. **建立 branch**：從目前的 master `git checkout -b autoresearch/<tag>`。
3. **讀取在範圍內的檔案**：repo 很小。讀以下檔案以取得完整背景：
   - `README.md` — repo 背景。
   - `prepare.py` — 固定常數、資料準備、tokenizer、dataloader、評估。**不要修改。**
   - `train.py` — 你會修改的檔案。模型架構、optimizer、訓練迴圈。
4. **確認資料存在**：檢查 `~/.cache/autoresearch/` 是否包含資料 shards 和 tokenizer。如果沒有，請告訴使用者執行 `uv run prepare.py`。
5. **初始化 `results.tsv`**：建立只有 header row 的檔案。untracked。
6. **初始化 `journal.md`**（untracked），使用以下範本：

   ```markdown
   # Journal

   ## Open questions
   - (empty)

   ## Reflections
   - (empty)
   ```

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

**目標很簡單：取得最低的 val_bpb。** 因為時間預算是固定的，你不需要擔心訓練時間 — 永遠都是 5 分鐘。

**VRAM** 是軟性限制。為了有意義的 val_bpb 進步，一些增加是可以接受的，但不應該爆炸性地增加。

**簡潔性原則**：在其他條件相同時，越簡單越好。一點點進步但加上醜陋複雜的程式碼是不值得的。反之，**移除東西**而得到相等或更好的結果，是個很棒的成果 — 那是一次簡化的勝利。**Soft cap（v3 新增）：如果這次實驗的 diff 超過 50 行新增，必須在 commit body 為這個大小辯護** — 大 diff 是詮釋上的危險，通常也是過度工程的訊號。問自己：「資深工程師會說這太複雜嗎？」如果是，commit 前先簡化。

**第一次 run**：你的第一次 run 永遠應該是建立 baseline，因此你會原封不動地執行訓練腳本。

## 假設紀律（v3 延伸）

每次 commit 之前，commit message 在 subject 之後必須包含**四行**：

```
Hypothesis: <一句話 — 這個改動會降低 val_bpb 的機制>
Assumptions: <一行 — 我所依賴的當前程式碼狀態；如果這個假設錯，這次實驗就無法詮釋>
Prediction: <預期方向與幅度，例如「↓ 0.001–0.003」、「↓ 但 VRAM +10%」、「探索性」>
Tripwire: <一個結果，代表實驗本身壞了（不是想法錯）；如果觸發要先排查再 discard>
```

範例：

```
Lower AdamW beta2 from 0.95 to 0.9

Hypothesis: 降低 momentum staleness 應該能在 5 分鐘預算的少步數情境下幫助收斂。
Assumptions: 目前 beta1=0.9 與 LR schedule 是合理的；瓶頸真的在 momentum
             staleness，而不是容量或 LR。
Prediction: ↓ 0.001–0.003。
Tripwire: 如果 val_bpb > 1.05 或 VRAM 暴漲 >20%，是別的東西壞了 — 在 discard
          前先重看 diff。
```

每行的意義：
- **Hypothesis** 把 hack 變成實驗。失敗也有資訊量。
- **Assumptions** 把「你**沒**在測的事」攤出來 — 假設錯時，結果是無法詮釋而不是負面。
- **Prediction** 是你的先驗；用 prediction vs 實際長期累積能校正你的判斷。
- **Tripwire** 區分「想法錯」與「實驗沒按預期跑」。觸發 tripwire 時，先做一次健全性檢查再 log discard。

如果你真的講不出機制，就寫 `Prediction: exploratory` — 但要把這類 commit 標記為探索性，這樣 journal 才能統計。

## Surgical experimental diffs（v3 新增）

**實驗 commit 中每一行改動都必須能追溯到當前的 hypothesis。** 順便的改動 — 排版微調、註解修飾、變數改名、清理死碼、順手 refactor — 都**不能**打包進實驗 commit。

為什麼：當一次 run 被 `discard`，乾淨的 diff 能讓你分辨**是哪一行**造成退步。打包的 diff 把「假設失敗」與「順便的改動弄壞東西」混在一起，可詮釋性就沒了。

**檢驗**：挑 diff 中任何一行。問：「如果只 revert 這一行，這次實驗還在測同一個 hypothesis 嗎？」如果是，這行就是順便的 — 從這個 commit 拿掉。

**清理是允許的，但要獨立**：如果你看到死碼或想 refactor，就獨立做一個 commit，subject 一行例如 `refactor: rename foo to bar (no-op expected)`。之後跑一次實驗；val_bpb 在 seed 噪音範圍內應該不變。如果變了，那次「refactor」其實是行為改動 — 排查並當成普通實驗處理。

這條規則與簡潔性原則互補。鼓勵刪 code，但刪 code 必須獨立站，不能搭便車進實驗 commit。

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

## 小幅進步的二次種子驗證

單一 run 把「真正的改進」與「seed 噪音」混在一起。如果不驗證，運氣好的 run 會被固化進 running-best branch，後續實驗就建在浮沙上。

**觸發條件**：一次 run 即將被標記為 `keep`，**且** `Δval_bpb < 0.002`（相對於目前最佳）。

**程序**：
1. 把 `train.py` 裡的 random seed 加 1（**其他什麼都不要改** — 這本身就是一個 surgical commit）。
2. 重跑：`uv run train.py > run.log 2>&1`。
3. 計算兩次 run 的 val_bpb 平均。
4. 如果平均仍打敗先前最佳 → `keep`。否則 → `discard` 並 `git reset`。

兩次 run 都要寫進 `results.tsv`。在第二列的 description 後加 `(2-seed avg=X.XXXXXX)`。

`Δ ≥ 0.002` 的進步可以跳過這個 step。

## 週期性回顧（v3 延伸）

每完成 **20 次實驗**，在迴圈的頂部執行一次 self-check。這是自主的 — **不要**停下來問人類，也**不要**跳過。

**程序**：
1. 讀 `results.tsv`。把最近 ~20 列按主題分組（架構 / optimizer / 超參數 / 正則化 / 資料 / 等）。
2. 在 `journal.md` 的 **Reflections** 區追加一段（~5 行）：
   - 目前最佳的 val_bpb。
   - 最近幾次 `keep` 的共同主題。
   - 已知的 dead end（只產生 `discard`/`crash` 的主題）。
   - 接下來要嘗試的 1–2 個未探索方向。
3. **處理 Open questions 區（v3 新增）**：掃過你累積的 bullets。最多挑一個在接下來的實驗中嘗試解決 — 讀 code、跑一個針對性的小實驗、或直接寫下暫時的答案。已解決的 question 連同答案搬到 Reflections 區。
4. 立刻回到迴圈。下一次 tune 應該從 journal 的建議或某個 open question 中取題。

**Open questions 紀律（v3 新增）**：在迴圈中對某件事感到不確定時 — 為什麼這段 code 存在？某個假設是否成立？幾個方法該先試哪個？— 你不能問人類。改成 **在 `journal.md` 的 Open questions 區追加一行 bullet**，然後繼續。這是 Karpathy 第 1 條（「不要藏疑惑」）對 autonomous loop 的改寫：問題被表面化、被持久紀錄，但迴圈不停下來。

## 實驗迴圈

實驗在一個專屬的 branch（例如 `autoresearch/mar5` 或 `autoresearch/mar5-gpu0`）上執行。

LOOP FOREVER：

1. **（每第 20 次迭代）** 執行週期性回顧（Reflections + Open questions），追加到 `journal.md`。見上面「週期性回顧」。
2. 看一下 git 狀態：目前在哪個 branch／commit。
3. 用一個實驗想法調整 `train.py`，**手術式** — 每一行改動都必須能追溯到 hypothesis。如果 `journal.md` 非空，從 reflections + open questions 中取題。
4. 為 commit message 寫好 `Hypothesis:`、`Assumptions:`、`Prediction:`、`Tripwire:` 四行。見上面「假設紀律」。
5. 用 step 4 的訊息 `git commit`。如果 diff > 50 行新增，在 body 中辯護。
6. 執行實驗：`uv run train.py > run.log 2>&1`（**全部都重導向** — 不要用 tee 或讓輸出灌進你的 context）。
7. 讀取結果：`grep "^val_bpb:\|^peak_vram_mb:" run.log`。
8. 如果 grep 輸出是空的，表示 crash 了。執行 `tail -n 50 run.log`，嘗試修。試了幾次還是不行就 log `crash` 繼續。
9. 如果一次 run 總時間超過 10 分鐘，就終止它，視為失敗（`discard` 並 revert）。
10. **Tripwire check**：如果結果命中你在 step 4 寫的 tripwire 條件（比預測糟很多、VRAM 異常暴漲），花一分鐘檢查實驗是否按預期跑（重看 diff、重 grep log）。發現 bug 就修並重跑；否則結果就此採信。
11. 把結果紀錄到 `results.tsv`。如果結果與 `Prediction` 強烈矛盾，在 `journal.md` 的 Open questions 加一行（「為什麼 X 讓我意外？」）。
12. **（若即將 keep 且 Δval_bpb < 0.002）** 在做決定前先執行二次種子驗證。
13. 如果 val_bpb 進步了，就推進 branch — 保留 commit。否則 `git reset`。

**Crash**：如果一次 run crash（OOM、bug 等等），用你的判斷力。簡單修復 → 修並重跑。想法本身就壞 → log `crash`，繼續。

**永遠不要停**：一旦實驗迴圈開始，**不要**停下來問人類是否該繼續。**不要**問「我該繼續嗎？」或「現在是好的停止點嗎？」。人類可能在睡覺，或離開電腦，他們期待你**無限期地**繼續工作直到被手動停止。你是自主的。如果你沒點子了，想得更深 — 讀 code 中參考的論文、重讀範圍內的檔案找新角度、嘗試組合先前差一點成功的東西、嘗試更激進的架構改動。**journal 是你的朋友** — Reflections（你學到的）和 Open questions（你還沒釐清的）都是。這個迴圈會一直跑，直到人類打斷你為止，就這樣。

舉個使用情境：使用者可能在睡覺時讓你跑著。每小時 ~12 次實驗，平均一晚 ~100 次。使用者醒來時就會看到你在他睡覺時完成的所有實驗結果！

## 變更總結

| # | 變更 | 來源 |
|---|---|---|
| 1 | 設定階段初始化 `journal.md` | v1 |
| 2 | 每次 commit 必須附 `Hypothesis:` + `Prediction:` 兩行 | v1 |
| 3 | 小幅進步（Δ < 0.002）做二次種子驗證 | v1 |
| 4 | 每 20 次實驗做一次週期性回顧 | v1 |
| 5 | Hypothesis 區塊延伸 `Assumptions:` + `Tripwire:` 兩行 | v3（Karpathy #1, #4）|
| 6 | 新章節 **Surgical experimental diffs** — 每行追溯到 hypothesis；清理走獨立 no-op commit | v3（Karpathy #3）|
| 7 | 簡潔性原則加 soft cap：diff > 50 行新增必須在 commit body 辯護 | v3（Karpathy #2）|
| 8 | `journal.md` 加 **Open questions** 區；tripwire 意外與不確定都進這裡；回顧時處理 | v3（Karpathy #1，延後問）|
| 9 | 迴圈 step 10：tripwire 觸發時做 sanity check 再 log discard | v3（Karpathy #4）|
