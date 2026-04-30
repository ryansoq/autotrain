---
# 開始迴圈前，下方每一個值都要編輯。
# 驗證指令：grep -nE '\{FILL' program.md   （必須無輸出）。

system:
  validation:
    command: |
      {FILL: shell 命令 — 決定性的 go/no-go check}
    on_failure: fix_or_revert

goal:
  mission: "{FILL: 一句話 — 這個專案的「成功」長什麼樣？}"
  metric:
    # 從三種挑一個分支：scalar | multi_axis | pass_fail。刪掉沒用到的分支。
    type: scalar

    # --- if scalar ---
    name: "{FILL: 指標名稱，例如 val_bpb}"
    direction: lower_is_better        # 或 higher_is_better
    noisy: false                      # 若為 true，需設 small_win_threshold
    small_win_threshold: 0.0

    # --- if multi_axis ---
    # axes: [depth, content, polish, code]
    # priority: [depth, content, code, polish]   # 排序，最高優先在前
    # consecutive_same_axis_max: 3

    # --- if pass_fail ---
    # （不需額外欄位；validation 通過即決策）

constraints:
  scope:
    in:  ["{FILL: agent 可修改的路徑}"]
    out: ["{FILL: agent 不可修改的路徑}"]
  hard_constraints:
    - "{FILL: 規則 1}"
    - "{FILL: 規則 2}"
  iteration_budget:
    type: best_effort                 # 或 fixed_time
    time_minutes: 0                   # 只在 type: fixed_time 時使用
    kill_minutes: 10                  # 任何 type 都套用的硬性 kill 門檻

loop:
  stopping_conditions:
    never_stop: false                 # 開放式搜尋（autotrain 風格）設 true
    conditions:
      - "{FILL: 例如 backlog 變稀薄且 reflection 也想不出新方向}"
      - "{FILL: 例如 連續兩次 iter crash 且無法修}"
  cadence:
    reflection_every_n_keeps: 5
    exploratory_every_n_iters: 10
    consolidation_every_n_keeps: 15
  idea_sources:                       # 依序嘗試；不存在或空的 source 跳過
    - feedback_inbox
    - backlog
    - reflection_generated
    - exploratory
---

# program.md — 通用基底

這個檔案是任何「自主迴圈」型專案的通用基底：LLM agent 對著明確目標**手術式**地反覆做實驗／改進，直到觸發停止條件。

從兩次實際 run 提煉而成：**autotrain**（ML 超參數搜尋；最小化 `val_bpb`；永不停）與 **pikachu-godot**（Godot RPG 內容建構；多軸 rubric；有限停止）。本文按 5 個主軸組織 — **系統 / 目標 / 規格 / 約束 / 迴圈** — YAML frontmatter 對應其中 4 個（規格是通用，不需專案填）。

## 如何使用這個基底

1. **複製**這個檔案到你的專案根目錄為 `program.md`。
2. **編輯檔頭 YAML frontmatter**：把每一個 `{FILL: ...}` placeholder 換掉。
3. **挑一個 `goal.metric.type`**（`scalar` | `multi_axis` | `pass_fail`），刪掉沒用的分支。
4. **決定 `loop.stopping_conditions.never_stop`**：開放式搜尋設 `true`，有限任務設 `false` 並寫明 `conditions`。
5. **啟動前 slot-leak 檢查**：執行 `grep -nE '\{FILL' program.md`，必須無輸出。
6. *(填完後可選)* 刪掉這個「如何使用」區段與下方的「Reference examples」區段。
7. 開始迴圈。每次 iter 都重讀 `program.md` 作為唯一真相來源。

---

## 1. 系統（System / 環境）

Agent 假設：

- 專案根目錄是個 git repository。
- 一個能執行 `system.validation.command` 的 shell。
- 一個配對 log 檔：`results.tsv`（untracked、append-only）。
- 基底檔案本身：`program.md`（這個檔；每次 iter 開頭重讀）。

**可選佐料檔**（迴圈會檢查存在性）：

- `journal.md` — 永遠建議。兩個區段：`Open Questions`、`Reflections`。
- `backlog.md` — 有限內容領域建議（`loop.stopping_conditions.never_stop = false`）。
- `feedback_inbox.md` — 人類在 sessions 之間寫入的 append-only 檔；迴圈在 iter 開頭讀（最高優先 idea source）。
- `STOP` — kill-switch。專案根目錄空檔；iter 開頭存在就退出。

**Validation**：`system.validation.command` 必須是決定性的，回傳明確的 OK/FAIL 訊號。每次 commit 前執行；改動後失敗就修或 revert，再 log iter。

---

## 2. 目標（Goal）

`goal.mission` 是一句話，說明這個專案的「成功」長什麼樣。整個迴圈都依這個 mission 判斷。

`goal.metric` 定義每次 iter 怎麼衡量。三種形狀：

| `goal.metric.type` | 決策規則 |
|---|---|
| `scalar` | 嚴格進步才 keep（依 `direction`）。若 `noisy: true` 且改進 < `small_win_threshold`，先做 seed re-check。 |
| `multi_axis` | 至少一軸進步且無一軸退步才 keep。遵守 `consecutive_same_axis_max`。 |
| `pass_fail` | `system.validation.command` 通過且改動有對應追蹤目標才 keep。 |

指標是**唯一**的量化錨點。沒被 `goal.metric` 捕捉的東西，要不放進 `constraints.hard_constraints`，否則對迴圈來說等於不存在。

---

## 3. 規格（Spec — 文件與格式）

通用格式。不要客製。

### Hypothesis 區塊 — 每次 commit

每次 commit message 在 subject 之後必須包含四行：

```
Hypothesis: <一句話 — 這個改動推進目標的機制>
Assumptions: <一行 — 我所依賴的當前 repo 狀態；如果這個假設錯，結果就無法詮釋>
Prediction: <預期效果：方向、幅度、或質性描述>
Tripwire: <一個結果，代表實驗本身壞了（不是想法錯）>
```

如果你真的講不出機制，在 `Prediction:` 前加 `exploratory:`，並對結果格外懷疑。

### results.tsv schema（依 `goal.metric.type` 推導）

Tab 分隔、untracked、append-only。每次嘗試都記一列，包含 `discard` 與 `crash`。

| `goal.metric.type` | 欄位 |
|---|---|
| `scalar` | `commit  metric  memory_or_cost  status  description` |
| `multi_axis` | `commit  axis  status  summary` |
| `pass_fail` | `commit  status  summary` |

共通：`commit` 是短 hash（7 字元），無 commit 用 `-`。`status ∈ {keep, discard, crash}`。最後自由文字欄位不能有 tab。Crash 時數字欄位都用 `0`。

### journal.md 格式

```markdown
# Journal
## Open Questions
- (迴圈中累積的疑問；一行一個)

## Reflections
- (週期性綜合整理；每次 reflection 一塊)
```

### Placeholder 慣例

每個專案要填的位置都標 `{FILL: <提示>}`。`grep -nE '\{FILL' program.md` 在填完後必須無輸出。Agent 在每次 iter 開頭也會重跑這個 grep 當哨兵。

---

## 4. 約束（Constraints）

什麼能動 / 什麼不能動；限制行為的條件。

### 專案定義（frontmatter）

- `constraints.scope.in` / `constraints.scope.out` — agent 可 / 不可修改的路徑。
- `constraints.hard_constraints` — 專案不可違背的規則（不新增 deps、專案必須 boot、README 同步等）。
- `constraints.iteration_budget` — 每次 iter 的時間限制。`kill_minutes` 是無論 `type` 為何都套用的硬上限。

### 通用（不要編輯）

**Surgical diffs。** 實驗 commit 中每一行改動都必須能追溯到當前 Hypothesis。順手的改動 — 排版、改名、順手 refactor、清死碼 — 都**不能**打包進實驗 commit。

檢驗：挑 diff 中任何一行。問：「如果只 revert 這一行，這個 commit 還在測同一個 hypothesis 嗎？」如果是，這行就是順便的 — 拿掉。

清理是允許的，但要獨立 commit，subject 用 `refactor: ... (no behavior change expected)`。這種 commit 後跑一次驗證；指標應該在噪音範圍內不變。衡量 diff 大小用 `git diff --stat -w` — 縮排重排會讓原始 stats 暴漲。

**Simplicity bias 加 soft cap。** 其他條件相同時，越簡單越好。**移除程式碼**而指標持平，永遠是 keep。**Soft cap**：實驗 diff 超過 50 行（不算空白）新增就必須在 commit body 為大小辯護。

**永遠不問人類，但永遠尊重 `STOP`。** 迴圈一旦開始，**不要**停下來問「我該繼續嗎？」把疑問寫進 `journal.md` 的 Open Questions。迴圈只在以下三件事任一發生時結束：
1. `loop.stopping_conditions.conditions` 任一述詞觸發。
2. 專案根目錄存在 `STOP` 檔（決定性人類覆寫）。
3. 當次 iter 超過 `constraints.iteration_budget.kill_minutes`（視為 crash）。

---

## 5. 迴圈（Loop）

每次 iter 怎麼跑。

### 專案定義（frontmatter）

- `loop.stopping_conditions` — 迴圈何時正常結束（或 `never_stop: true` 開放式搜尋）。
- `loop.cadence` — 迴圈中的 meta cycles（reflection / 強迫探索 / 強迫 consolidation）。
- `loop.idea_sources` — 挑下一項的優先序。

### 通用：Open Questions 紀律（in-loop 行為）

迴圈中對某件事不確定且無法廉價解決時：在 `journal.md` 的 Open Questions 區追加一行 bullet，然後繼續。**不要停下迴圈。不要問人類。** 下次 reflection cycle 處理。

### 迴圈

```
LOOP:

  0. Pre-iteration checks（永遠執行）：
       - 若 STOP file 存在 → 寫 final journal entry「loop terminated by STOP」，退出。
       - 確認 `grep -nE '\{FILL' program.md` 為空（slot-leak guard）。
       - 若 keeps_count > 0 且 keeps_count % loop.cadence.reflection_every_n_keeps == 0
         且 last_reflection_at_keeps != keeps_count：
           執行 reflection — 更新 journal.md Reflections，處理 Open Questions。

  1. Orient：git status；讀 journal.md；讀 results.tsv 最後 5 列。

  2. 讀 feedback_inbox.md 若存在且非空（最高優先 idea source）。

  3. 依 loop.idea_sources 優先序挑下一項；cadence 覆寫：
       - 每 loop.cadence.exploratory_every_n_iters 次 iter：強迫挑非 backlog、
         接受門檻較低的項目（對抗 100%-keep 失能舒適區）。
       - 每 loop.cadence.consolidation_every_n_keeps 次 keep：強迫一次目標為
         負 LOC delta 的 iter（refactor、dedupe、刪除；指標必須維持）。

  4. 寫 Hypothesis / Assumptions / Prediction / Tripwire — 在 coding **之前**。

  5. 手術式實作。檢查 `git diff --stat -w` 合理。

  6. 跑 system.validation.command。

  7. Tripwire check：若結果命中 tripwire，先做健全性檢查（重看 diff、重 grep
     log）確認實驗按預期跑了，再決定是否 discard。

  8. 若 goal.metric.noisy 且改進 < goal.metric.small_win_threshold：
     seed re-check（用不同 seed 多跑一次；要求兩次平均仍打敗 baseline）。
     否則跳過。

  9. 依 goal.metric.type 決定 keep / discard：
       - scalar：嚴格進步才 keep。
       - multi_axis：至少一軸進步且無一軸退步才 keep；遵守
         goal.metric.consecutive_same_axis_max。
       - pass_fail：system.validation.command 通過才 keep。

 10. Commit（只有 keep 才 commit），body 含四行 hypothesis 區塊。
     若 diff > 50 行非空白新增，在 body 中辯護。

 11. 在 results.tsv 追加一列（每次嘗試都記錄，包含 discard 與 crash）。

 12. 只有結果讓你意外或揭露了值得記下的事，才寫 journal。不是每次都寫。

 13. 更新 backlog.md（劃掉做完的、加入發現的後續）。

 14. 檢查 loop.stopping_conditions.conditions；任一為真 → final journal entry，退出。
```

---

## Reference examples — 兩個案例怎麼填 frontmatter

### autotrain（ML 超參數搜尋）

```yaml
system:
  validation:
    command: |
      uv run train.py > run.log 2>&1
      grep "^val_bpb:\|^peak_vram_mb:" run.log
    on_failure: fix_or_revert

goal:
  mission: "在固定的評估 harness 上拿到最低的 val_bpb；每次訓練 run 上限 5 分鐘；迴圈無限期執行。"
  metric:
    type: scalar
    name: val_bpb
    direction: lower_is_better
    noisy: true
    small_win_threshold: 0.002

constraints:
  scope:
    in:  [train.py]
    out: [prepare.py, evaluation harness, dependencies, time budget]
  hard_constraints:
    - "no new packages beyond pyproject.toml"
    - "do not modify the eval harness"
    - "project must run without crashing within the time budget"
  iteration_budget:
    type: fixed_time
    time_minutes: 5
    kill_minutes: 10

loop:
  stopping_conditions:
    never_stop: true
    conditions: []
  cadence:
    reflection_every_n_keeps: 5
    exploratory_every_n_iters: 10
    consolidation_every_n_keeps: 15
  idea_sources:
    - feedback_inbox
    - reflection_generated
    - exploratory
```

### pikachu-godot（遊戲開發內容建構）

```yaml
system:
  validation:
    command: |
      cd examples/pikachu-godot
      rm -rf .godot
      godot4 --headless --import 2>&1 | grep -iE '^ERROR' | grep -v leaked
      timeout 6 godot4 --headless --quit-after 240 res://scenes/TitleScreen.tscn 2>&1 | grep -iE 'error|invalid|fail' | grep -v leaked
      timeout 6 godot4 --headless --quit-after 240 res://scenes/Overworld.tscn   2>&1 | grep -iE 'error|invalid|fail' | grep -v leaked
      timeout 6 godot4 --headless --quit-after 240 res://scenes/Battle.tscn      2>&1 | grep -iE 'error|invalid|fail' | grep -v leaked
    on_failure: fix_or_revert

goal:
  mission: "把 examples/pikachu-godot/ 變成更完整的遊戲 — 玩法系統、內容、機制 — 一次一個小的手術式改動。"
  metric:
    type: multi_axis
    axes: [depth, content, code, skill_ready, polish, bug]
    priority: [depth, content, code, skill_ready, polish, bug]
    consecutive_same_axis_max: 3

constraints:
  scope:
    in:  [scenes/, scripts/, tools/make_assets.py, README.md, journal.md, results.tsv, program.md]
    out: [skills/, src/, top-level repo docs, CLAUDE.md]
  hard_constraints:
    - "no new dependencies beyond Pillow + numpy"
    - "no image_gen — use Pillow placeholder shapes"
    - "no audio files — synthesized AudioStreamGenerator only"
    - "SKILL replacement points must remain swappable"
    - "project must boot via godot4 --headless --import"
    - "ship user-visible feature → must update README Limitations in same commit"
  iteration_budget:
    type: best_effort
    kill_minutes: 10

loop:
  stopping_conditions:
    never_stop: false
    conditions:
      - "backlog 未做項目少於 3 個 AND 一次完整 reflection 仍想不出新點子"
      - "連續兩次 iter crash 且無法修"
      - "revert 後 baseline 仍硬性驗證失敗（環境壞）"
  cadence:
    reflection_every_n_keeps: 5
    exploratory_every_n_iters: 10
    consolidation_every_n_keeps: 15
  idea_sources:
    - feedback_inbox
    - backlog
    - reflection_generated
    - exploratory
```

---

## 兩次案例的 lessons（appendix）

1. **框架是複利。** 一筆 ~50-LOC 的可重用框架投資通常能解鎖 5–10 次後續 ~30-LOC 的 iter。
2. **「做完了」大概 80% 是錯的。** Agent 宣稱某軸 tapped out 時，強制再做一次 reflection scan。
3. **縮排重排騙過 diff stats。** size cap 用 `git diff --stat -w`。
4. **state-mutation 路徑需要兜底的勝負判定。** 新增一種「step 結束時 state 會變化」的途徑時，確保現有的終局檢查能看到這條新路徑。
5. **跨場景框架重複到 2 個是可接受的。** 第 3 個 consumer 出現時才抽共用 module。
6. **單一最高 ROI 的輸入是直接的使用者回饋。** 一句 user note 可在數個 iter 之中揭出多個隱藏 bug。在你需要前先建好 `feedback_inbox.md` 通道。
7. **README rot 是真風險。** 一個會改 code 但沒有「同步 README」硬規則的迴圈，會讓 README 持續宣稱專案缺乏一些早就 ship 的 feature。
8. **100% keep 率是異味，不是勝利。** 通常代表 agent 只挑經過預先過濾的安全項目。`loop.cadence.exploratory_every_n_iters` 是解藥 — 接受較高 discard 率作為代價。
