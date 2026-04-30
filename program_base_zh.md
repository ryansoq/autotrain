# program.md — 通用基底

這個檔案是任何「自主迴圈」型專案的通用基底：一個 LLM agent 對著明確目標**手術式**地反覆做實驗／改進，直到觸發停止條件。

它從兩次實際 run 提煉而成：

- **autotrain**（ML 超參數搜尋）— 最小化 `val_bpb`，每次 iter 固定 5 分鐘訓練預算，永不停（直到人類打斷）。~80+ 次實驗，~18% keep 率。
- **pikachu-godot**（遊戲開發內容建構）— 沿多軸品質 rubric 改進 Godot RPG demo，無時間預算，backlog 收斂時停止。108 次 iter，100% keep 率。

兩次都活下來的紀律就是通用部分。其餘都是 slot，每個專案自己填。

## 如何使用這個基底

1. **複製**這個檔案到你的專案根目錄為 `program.md`。
2. **「通用核心」逐字保留** — 那些就是紀律。
3. **填好每一個 `{SLOT}`** — 在「Project slots」區。
4. **填完後刪掉**這個「如何使用」標頭與案例範例。
5. **啟動前 slot-leak 檢查**：執行 `grep -nE '\{[A-Z_]+\}' program.md`。應該無輸出。任何回傳的行都是未填的 `{SLOT}` 字面殘留 — 開始迴圈前先填好。
6. 開始迴圈。每次 iter 都重讀 `program.md` 作為唯一真相來源。

唯一配對的檔案是 `results.tsv`（untracked、append-only log）。可選佐料：`journal.md`（永遠建議）、`backlog.md`（有限內容任務建議）、`feedback_inbox.md`（如果有人類會給回饋的話建議）。

---

## 通用核心（逐字複製到你專案的 program.md）

### Hypothesis 區塊 — 每次 commit

每次 commit message 在 subject 之後必須包含四行：

```
Hypothesis: <一句話 — 這個改動推進目標的機制>
Assumptions: <一行 — 我所依賴的當前 repo 狀態；如果這個假設錯，結果就無法詮釋>
Prediction: <預期效果：方向、幅度、或質性描述>
Tripwire: <一個結果，代表實驗本身壞了（不是想法錯）— 在 discard 前先排查>
```

如果你真的講不出機制，在 `Prediction:` 前加 `exploratory:`，並對結果格外懷疑。

### Surgical diffs

實驗 commit 中**每一行改動都必須能追溯到當前 Hypothesis**。順手的改動 — 排版、改名、順手 refactor、清死碼 — 都**不能**打包進實驗 commit。它們會在 discard 時模糊掉「是哪一行壞了」。

檢驗：挑 diff 中任何一行。問：「如果只 revert 這一行，這個 commit 還在測同一個 hypothesis 嗎？」如果是，這行就是順便的 — 從這個 commit 拿掉。

清理是允許的，但要獨立 commit，subject 用 `refactor: ... (no behavior change expected)`。這種 commit 後跑一次驗證；指標應該在噪音範圍內不變。

衡量 diff 大小用 `git diff --stat -w` — 縮排重排會讓原始 stats 暴漲（純空白變化）。

### Simplicity bias 加 soft cap

其他條件相同時，越簡單越好。一點點進步但加上醜陋複雜的程式碼是不值得的。**移除程式碼**而指標持平，永遠是 keep。

**Soft cap**：實驗 diff 超過 50 行（不算空白）新增就必須在 commit body 為大小辯護。

### Open Questions 紀律

迴圈中對某件事不確定（「為什麼這段 code 存在？」、「這個假設成立嗎？」、「X 應該等於 Y 嗎？」）且無法廉價解決時：在 `journal.md` 的 Open Questions 區追加一行 bullet，然後繼續。**不要停下迴圈。不要問人類。** 下次 reflection cycle 處理。

### Journal 是活的記憶

`journal.md` 由迴圈寫入，分兩區：

```markdown
# Journal
## Open Questions
- (迴圈中累積的疑問；一行一個)

## Reflections
- (週期性綜合整理；每次 reflection 一塊)
```

每 `{REFLECTION_CADENCE}` 次 keep 觸發一次 Reflection。內容包括最近一批的綜合整理（哪些 land、哪些沒、共同主題、dead end、接下來 1–2 個方向），並處理 Open Questions list。

### 永遠不要問人類

迴圈一旦開始，**不要**停下來問「我該繼續嗎？」人類可能在睡覺。把疑問寫進 Open Questions 然後繼續。迴圈只在 `{STOPPING_CONDITIONS}` 觸發時才結束。

### 迴圈

```
LOOP UNTIL {STOPPING_CONDITIONS} 任一為真：

  0. 如果 iteration_count % {REFLECTION_CADENCE} == 0 且本次是 keep：
       執行 reflection — 更新 journal.md Reflections，處理 Open Questions。

  0a. 如果 feedback_inbox.md 存在且非空：
       優先順序高於 backlog 與 reflection-generated ideas。

  1. Orient：先確認 `grep -nE '\{[A-Z_]+\}' program.md` 為空（slot-leak
     哨兵 — 未填 slot 字面殘留要立刻 fail loud）；然後 git status、
     讀 journal.md、讀 results.tsv 最後 5 列。

  2. 依 {IDEA_SOURCES} 挑下一個項目；cadence 覆寫：
       - 每 {EXPLORATORY_EVERY} 次 iter：強迫挑一個非 backlog、非安全的項目
         （預期 discard 機率較高；對抗 100% keep 失能舒適區）
       - 每 {CONSOLIDATION_EVERY} 次 keep：強迫一次目標為「LOC 負成長」的
         iter（refactor、dedupe、刪除；指標必須維持）

  3. 寫 Hypothesis / Assumptions / Prediction / Tripwire — 在 coding **之前**。

  4. 手術式實作。檢查 `git diff --stat -w` 合理。

  5. 跑 {VALIDATION}。

  6. Tripwire check：如果結果命中 tripwire，先做健全性檢查（重看 diff、重 grep
     log）確認實驗按預期跑了，再決定是否 discard。

  7. 如果 {NOISY} 且改進 < {SMALL_WIN_THRESHOLD}：seed re-check（用不同 seed
     多跑一次；要求兩次平均仍打敗 baseline 才 keep）。否則跳過。

  8. 依 {DECISION_RULE} 決定 keep / discard。

  9. Commit（只有 keep 才 commit），body 含四行 hypothesis 區塊。
     如果 diff > 50 行非空白新增，在 body 中辯護。

 10. 在 results.tsv 追加一列（每次嘗試都記錄，包含 discard 與 crash）。

 11. 只有結果讓你意外或揭露了值得記下的事，才寫 journal。不是每次都寫。

 12. 更新 backlog.md（劃掉做完的、加入發現的後續）。

 13. 檢查 {STOPPING_CONDITIONS}；任一為真就寫 final journal entry 並停止。
```

---

## Project slots — 你必須填這些

### {MISSION}

一段話。這個專案的「成功」長什麼樣？

- *autotrain*：「在固定的評估 harness 上拿到最低的 `val_bpb`。每次訓練 run 上限 5 分鐘 wall clock。迴圈無限期執行；`progress.png` 那條曲線就是交付物。」
- *pikachu-godot*：「把 `examples/pikachu-godot/` 變成一個更完整的遊戲 — 玩法系統、內容、機制 — 一次一個小的手術式改動。當作長 run 的內容建構，不是 polish pass。」

### {SCOPE}

可動（in-scope）vs 不可動（out-of-scope）。

- *autotrain*：in = [`train.py`]；out = [`prepare.py`、評估 harness、相依、時間預算]。
- *pikachu-godot*：in = [`scenes/`、`scripts/`、`tools/make_assets.py`、專案 README、journal/results/program]；out = [`skills/`、`src/`、頂層 repo 文件、`CLAUDE.md`]。

### {HARD_CONSTRAINTS}

不可違背的規則。明確列出。

- *autotrain*：不能新增 `pyproject.toml` 之外的套件；不能修改評估 harness；專案必須在時間預算內不 crash 跑完。
- *pikachu-godot*：不能新增 `Pillow`+`numpy` 之外的相依；不能用 `image_gen`（用 Pillow placeholder）；不能用音訊檔案（只能 `AudioStreamGenerator`）；保持 SKILL replacement points 乾淨；專案必須能透過 `godot4 --headless --import` 啟動。

**通用建議**：如果你的專案有面向使用者的 README 列出 features 或 limitations，加一條：「任何 commit 若 ship 一個使用者可見的 feature，必須在同一 commit 更新 README 的 Limitations 區。」這防止 README rot — pikachu-godot 的 README 到 iter15 仍然列出 14 條已經 ship 但被標為「not in scope」的功能，因為迴圈從不更新它。

### {METRIC} 與 {DECISION_RULE}

從三種形狀挑一個：

**Scalar metric**（如 autotrain 的 `val_bpb`）：
- 方向：lower-is-better 或 higher-is-better。
- 決策：指標嚴格進步才 keep。
- 設定下方的 `{NOISY}` 與 `{SMALL_WIN_THRESHOLD}`。

**Multi-axis rubric**（如 pikachu-godot 的 depth/content/polish/code/skill-ready/bug）：
- 列出你的 axes。
- 優先順序（例如 depth > content > code > polish）：模稜兩可時偏向高優先序。
- 決策：至少一軸進步且無一軸退步才 keep。
- 建議的硬規則：同 axis 連續次數上限（預設 3），對抗 axis 漂移（特別是「polish trap」）。

**Pass-fail**（驗證通過或失敗）：
- 決策：`{VALIDATION}` 通過且改動有對應追蹤目標就 keep。
- 用於只做 refactor / cleanup 的迴圈；實務上少見。

### {VALIDATION}

可從 command line 跑、決定性的 go/no-go check。判讀**不應**需要人類判斷。

- *autotrain*：`uv run train.py > run.log 2>&1` 然後 `grep "^val_bpb:\|^peak_vram_mb:" run.log`（grep 空 ⇒ crash；用 `tail -n 50 run.log` 看 traceback）。
- *pikachu-godot*：`godot4 --headless --import` + 用 `--quit-after` 啟動每個 scene ~4 秒，grep stderr 找 `error|invalid|fail`（排除無害的 "ObjectDB instances leaked"）。

### {ITERATION_BUDGET}

- *autotrain*：固定 5 分鐘訓練（腳本內強制）；總 wall-clock 超過 10 分鐘就 kill。
- *pikachu-godot*：best-effort（無時間上限；iter 在驗證通過時結束）。

如果你的領域有自然的 per-iter 時間，固定它。如果沒有，用 best-effort 但**寫一個 kill 門檻**讓失控 iter 能被中止。

### {NOISY} 與 {SMALL_WIN_THRESHOLD}

指標每次 iter 是否有隨機性？

- *autotrain*：noisy=true、threshold=0.002。低於門檻時做一次 seed re-check，要求兩次平均仍打敗 baseline 才 keep。
- *pikachu-godot*：noisy=false（遊戲改動是決定性的）→ 跳過 seed re-check。

### {STOPPING_CONDITIONS}

- *autotrain*：永不停（只在人類打斷時）。
- *pikachu-godot*：以下**任一**為真就停：
  - Backlog 未做項目少於 3 個 **且** 一次完整 reflection 後仍想不出新點子。
  - 連續兩次 iter crash 且當下無法修。
  - revert 後 baseline 仍硬性驗證失敗（環境壞了）。

有限內容領域要設真正的停止條件。開放式搜尋領域，「永不停」是對的。

### {REFLECTION_CADENCE}

預設：每 **5 keeps**（pikachu-godot 採用；那邊 lessons 累積得快）。autotrain 原本「每 20」太稀疏 — 除非你的 iter 極度便宜，否則回頭採 5。

### {EXPLORATORY_EVERY} 與 {CONSOLIDATION_EVERY}

可選，但都是經驗換來的。

- `{EXPLORATORY_EVERY}`（預設 10）：每 N 次 iter 強迫挑**不在 backlog** 上的點子。接受門檻放低。對抗舒適區 — pikachu-godot 跑 108 次 iter 100% keep 率，失敗探索率太低。
- `{CONSOLIDATION_EVERY}`（預設 15）：每 N 次 keep 強迫一次目標為**負 LOC delta** 的 iter（刪除、dedupe、抽取）。指標必須維持。對抗「只增不減」 — pikachu-godot 的 `Battle.gd` 漲到 50KB、`Overworld.gd` 漲到 41KB，沒有任何一次 deletion-as-win iter。

### {IDEA_SOURCES}（依優先序）

預設順序：

1. `feedback_inbox.md`（如果存在且非空）— 最高 ROI，pikachu-godot iter47–49 驗證過。
2. `backlog.md` — 顯式 ranked 任務池。有限內容任務必備。
3. Reflection-generated ideas（在 `journal.md` Reflections）。
4. Exploratory（依 `{EXPLORATORY_EVERY}` 觸發）。

開放式搜尋領域（autotrain）可以沒有 backlog；reflection + exploratory 撐起來。

---

## results.tsv

Tab 分隔。Untracked（`.gitignore` 它）。Append-only — 每次嘗試都記一列，包含 `discard` 與 `crash`。

欄位形狀依 `{METRIC}` 變化：

- **scalar**：`commit  metric  memory_or_cost  status  description`
- **multi-axis**：`commit  axis  status  summary`
- **pass-fail**：`commit  status  summary`

共通欄位：
- `commit`：短 hash（7 字元）。discard / crash 沒 commit 時用 `-`。
- `status`：`keep | discard | crash`。
- 最後的自由文字欄位不能有 tab。

Crash 時數字欄位都 log 為 `0`。

---

## 建議的佐料檔

- `journal.md` — 永遠建立；Open Questions + Reflections 兩區在迴圈各處被引用。
- `backlog.md` — 有限內容領域建議。純 markdown；分 tier；做完用刪除線而不是真刪（歷史本身有用）。
- `feedback_inbox.md` — 有人類在身邊就建議。Append-only 檔案，人類在 sessions 之間寫入；迴圈在 iter 開頭讀。最高 ROI 的回饋通道。

---

## 兩次案例的 lessons（通用）

從兩次 run 出來、值得在任何地方再用的：

1. **框架是複利。** 一筆 ~50-LOC 的可重用框架投資（如 pikachu 的 `DialogueBox`、quest framework、held-item axes）通常能解鎖 5–10 次後續 ~30-LOC 的 iter。要分辨一次 iter 是在「鋪基礎」還是「擴充已有基礎」，能解鎖未來表面時優先前者。

2. **「做完了」大概 80% 是錯的。** Agent 宣稱某軸 tapped out 時，強制再做一次 reflection scan — pikachu 在 iter50、iter65、iter90 的「depth tapped out」判斷各自都錯（多個標準機制其實還沒做）。用結構化 prompt 重新 review 比直覺「應該夠了吧」可靠。

3. **縮排重排騙過 diff stats。** size cap 用 `git diff --stat -w`。一個原始 130/119 但實際只有 11 行新邏輯的 diff 不該觸發 50 行辯護規則。

4. **自傷路徑需要兜底的勝負判定。** 通則：當你新增一種「回合結束時 state 會變化」的途徑（recoil、confusion、poison 等），確保現有的終局檢查能看到這條新路徑。純加性 feature 仍需做整合掃描。

5. **跨場景框架重複到 2 個是可接受的。** 第 3 個 consumer 出現時才 refactor 成共用 autoload。提前抽象是常見的過度工程陷阱。

6. **單一最高 ROI 的輸入是直接的使用者回饋。** pikachu 中一句使用者回饋（「attack 那畫面感覺卡住」）揭露了 4 個隱藏 bug 與正確性問題。在你需要前先建好 `feedback_inbox.md` 通道。

7. **README rot 是真風險。** 一個會改 code 但沒有「同步 README」硬規則的迴圈，會讓 README 持續宣稱專案缺乏一些早就 ship 的 feature。專案有面向使用者文件時，把 README 同步列入 hard constraint。

8. **100% keep 率是異味，不是勝利。** 通常代表 agent 只挑經過預先過濾的安全項目。即使要付 discard 增加的代價，也要強迫探索性 iter。
