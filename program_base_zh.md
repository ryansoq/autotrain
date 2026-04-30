---
# 開始迴圈前，下方每一個值都要編輯。
# 驗證指令：grep -nE '\{FILL' program.md   （必須無輸出）。

mission: "{FILL: 一句話 — 這個專案的「成功」長什麼樣？}"

scope:
  in:  ["{FILL: agent 可修改的路徑}"]
  out: ["{FILL: agent 不可修改的路徑}"]

hard_constraints:
  - "{FILL: 規則 1}"
  - "{FILL: 規則 2}"

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

validation:
  command: |
    {FILL: shell 命令 — 決定性的 go/no-go check}
  on_failure: fix_or_revert

iteration_budget:
  type: best_effort                 # 或 fixed_time
  time_minutes: 0                   # 只在 type: fixed_time 時使用
  kill_minutes: 10                  # 任何 type 都套用的硬性 kill 門檻

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

從兩次實際 run 提煉而成：**autotrain**（ML 超參數搜尋；最小化 `val_bpb`；永不停）與 **pikachu-godot**（Godot RPG 內容建構；多軸 rubric；有限停止）。兩個 run 都活下來的紀律就是這份檔案的本文。檔頭 frontmatter 是每個專案要填的部分。

## 如何使用這個基底

1. **複製**這個檔案到你的專案根目錄為 `program.md`。
2. **編輯檔頭 YAML frontmatter**：把每一個 `{FILL: ...}` placeholder 換掉。
3. **挑一個 `metric.type`**（`scalar` | `multi_axis` | `pass_fail`），刪掉沒用的分支。
4. **決定 `stopping_conditions.never_stop`**：開放式搜尋設 `true`，有限任務設 `false` 並寫明 `conditions`。
5. **啟動前 slot-leak 檢查**：執行 `grep -nE '\{FILL' program.md`，必須無輸出。
6. *(填完後可選)* 刪掉這個「如何使用」區段與下方的「Reference examples」區段。
7. 開始迴圈。每次 iter 都重讀 `program.md` 作為唯一真相來源。

唯一配對的檔案是 `results.tsv`（untracked、append-only）。可選佐料：`journal.md`（永遠建立）、`backlog.md`（有限任務建議）、`feedback_inbox.md`（有人類在身邊建議）、`STOP`（kill-switch — 見下）。

---

## 通用核心（不要編輯）

### Hypothesis 區塊 — 每次 commit

每次 commit message 在 subject 之後必須包含四行：

```
Hypothesis: <一句話 — 這個改動推進目標的機制>
Assumptions: <一行 — 我所依賴的當前 repo 狀態；如果這個假設錯，結果就無法詮釋>
Prediction: <預期效果：方向、幅度、或質性描述>
Tripwire: <一個結果，代表實驗本身壞了（不是想法錯）>
```

如果你真的講不出機制，在 `Prediction:` 前加 `exploratory:`，並對結果格外懷疑。

### Surgical diffs

實驗 commit 中**每一行改動都必須能追溯到當前 Hypothesis**。順手的改動 — 排版、改名、順手 refactor、清死碼 — 都**不能**打包進實驗 commit。

檢驗：挑 diff 中任何一行。問：「如果只 revert 這一行，這個 commit 還在測同一個 hypothesis 嗎？」如果是，這行就是順便的 — 拿掉。

清理是允許的，但要獨立 commit，subject 用 `refactor: ... (no behavior change expected)`。這種 commit 後跑一次驗證；指標應該在噪音範圍內不變。

衡量 diff 大小用 `git diff --stat -w` — 縮排重排會讓原始 stats 暴漲。

### Simplicity bias 加 soft cap

其他條件相同時，越簡單越好。**移除程式碼**而指標持平，永遠是 keep。**Soft cap**：實驗 diff 超過 50 行（不算空白）新增就必須在 commit body 為大小辯護。

### Open Questions 紀律

迴圈中對某件事不確定且無法廉價解決時：在 `journal.md` 的 Open Questions 區追加一行 bullet，然後繼續。**不要停下迴圈。不要問人類。** 下次 reflection cycle 處理。

### Journal 是活的記憶

`journal.md` 分兩區：

```markdown
# Journal
## Open Questions
- (迴圈中累積的疑問；一行一個)

## Reflections
- (週期性綜合整理；每次 reflection 一塊)
```

每 `cadence.reflection_every_n_keeps` 次 keep 觸發一次 Reflection。內容綜合最近一批（哪些 land、哪些沒、共同主題、dead end、接下來 1–2 個方向）並處理 Open Questions list。

### 永遠不問人類，但永遠尊重 `STOP`

迴圈一旦開始，**不要**停下來問「我該繼續嗎？」把疑問寫進 Open Questions。迴圈只在以下三件事任一發生時結束：

1. `stopping_conditions.conditions` 任一述詞觸發。
2. 專案根目錄存在名為 `STOP` 的檔案（決定性 kill-switch — 人類用這個在不編輯 `program.md` 的情況下停下迴圈）。
3. 當次 iter 超過 `iteration_budget.kill_minutes`（視為 crash）。

### 迴圈

```
LOOP:

  0. Pre-iteration checks（永遠執行）：
       - 若 STOP file 存在 → 寫 final journal entry「loop terminated by STOP」，退出。
       - 確認 `grep -nE '\{FILL' program.md` 為空（slot-leak guard）。
       - 若 keeps_count > 0 且 keeps_count % cadence.reflection_every_n_keeps == 0
         且 last_reflection_at_keeps != keeps_count：
           執行 reflection — 更新 journal.md Reflections，處理 Open Questions。

  1. Orient：git status；讀 journal.md；讀 results.tsv 最後 5 列。

  2. 讀 feedback_inbox.md 若存在且非空（最高優先 idea source）。

  3. 依 idea_sources 優先序挑下一項；cadence 覆寫：
       - 每 cadence.exploratory_every_n_iters 次 iter：強迫挑非 backlog、
         接受門檻較低的項目（對抗 100%-keep 失能舒適區）。
       - 每 cadence.consolidation_every_n_keeps 次 keep：強迫一次目標為
         負 LOC delta 的 iter（refactor、dedupe、刪除；指標必須維持）。

  4. 寫 Hypothesis / Assumptions / Prediction / Tripwire — 在 coding **之前**。

  5. 手術式實作。檢查 `git diff --stat -w` 合理。

  6. 跑 validation.command。

  7. Tripwire check：若結果命中 tripwire，先做健全性檢查（重看 diff、重 grep
     log）確認實驗按預期跑了，再決定是否 discard。

  8. 若 metric.noisy 且改進 < metric.small_win_threshold：
     seed re-check（用不同 seed 多跑一次；要求兩次平均仍打敗 baseline）。
     否則跳過。

  9. 依 metric.type 決定 keep / discard：
       - scalar：嚴格進步才 keep。
       - multi_axis：至少一軸進步且無一軸退步才 keep；遵守
         consecutive_same_axis_max。
       - pass_fail：validation.command 通過才 keep。

 10. Commit（只有 keep 才 commit），body 含四行 hypothesis 區塊。
     若 diff > 50 行非空白新增，在 body 中辯護。

 11. 在 results.tsv 追加一列（每次嘗試都記錄，包含 discard 與 crash）。

 12. 只有結果讓你意外或揭露了值得記下的事，才寫 journal。不是每次都寫。

 13. 更新 backlog.md（劃掉做完的、加入發現的後續）。

 14. 檢查 stopping_conditions.conditions；任一為真 → final journal entry，退出。
```

---

## Reference examples — 兩個案例怎麼填 frontmatter

### autotrain（ML 超參數搜尋）

```yaml
mission: "在固定的評估 harness 上拿到最低的 val_bpb；每次訓練 run 上限 5 分鐘；迴圈無限期執行。"

scope:
  in:  [train.py]
  out: [prepare.py, evaluation harness, dependencies, time budget]

hard_constraints:
  - "no new packages beyond pyproject.toml"
  - "do not modify the eval harness"
  - "project must run without crashing within the time budget"

metric:
  type: scalar
  name: val_bpb
  direction: lower_is_better
  noisy: true
  small_win_threshold: 0.002

validation:
  command: |
    uv run train.py > run.log 2>&1
    grep "^val_bpb:\|^peak_vram_mb:" run.log
  on_failure: fix_or_revert

iteration_budget:
  type: fixed_time
  time_minutes: 5
  kill_minutes: 10

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
mission: "把 examples/pikachu-godot/ 變成更完整的遊戲 — 玩法系統、內容、機制 — 一次一個小的手術式改動。"

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

metric:
  type: multi_axis
  axes: [depth, content, code, skill_ready, polish, bug]
  priority: [depth, content, code, skill_ready, polish, bug]
  consecutive_same_axis_max: 3

validation:
  command: |
    cd examples/pikachu-godot
    rm -rf .godot
    godot4 --headless --import 2>&1 | grep -iE '^ERROR' | grep -v leaked
    timeout 6 godot4 --headless --quit-after 240 res://scenes/TitleScreen.tscn 2>&1 | grep -iE 'error|invalid|fail' | grep -v leaked
    timeout 6 godot4 --headless --quit-after 240 res://scenes/Overworld.tscn   2>&1 | grep -iE 'error|invalid|fail' | grep -v leaked
    timeout 6 godot4 --headless --quit-after 240 res://scenes/Battle.tscn      2>&1 | grep -iE 'error|invalid|fail' | grep -v leaked
  on_failure: fix_or_revert

iteration_budget:
  type: best_effort
  kill_minutes: 10

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

## results.tsv — 形狀依 `metric.type` 推導

Tab 分隔、untracked、append-only。每次嘗試都記一列，包含 `discard` 與 `crash`。

| `metric.type` | 欄位 |
|---|---|
| `scalar` | `commit  metric  memory_or_cost  status  description` |
| `multi_axis` | `commit  axis  status  summary` |
| `pass_fail` | `commit  status  summary` |

共通：`commit` 是短 hash（7 字元），無 commit 用 `-`。`status ∈ {keep, discard, crash}`。最後自由文字欄位不能有 tab。Crash 時數字欄位都用 `0`。

---

## 建議的佐料檔

- `journal.md` — 永遠建立；Open Questions + Reflections 在迴圈各處被引用。
- `backlog.md` — 有限內容領域建議（即 `stopping_conditions.never_stop = false`）。純 markdown、分 tier、做完用刪除線而不是真刪。
- `feedback_inbox.md` — 有人類在身邊就建議。Append-only，人類在 sessions 之間寫入；迴圈在 iter 開頭讀。
- `STOP` — kill-switch。專案根目錄空檔即可；agent 只檢查存在性。

---

## 兩次案例的 lessons（通用）

1. **框架是複利。** 一筆 ~50-LOC 的可重用框架投資通常能解鎖 5–10 次後續 ~30-LOC 的 iter。
2. **「做完了」大概 80% 是錯的。** Agent 宣稱某軸 tapped out 時，強制再做一次 reflection scan。
3. **縮排重排騙過 diff stats。** size cap 用 `git diff --stat -w`。
4. **state-mutation 路徑需要兜底的勝負判定。** 新增一種「step 結束時 state 會變化」的途徑時，確保現有的終局檢查能看到這條新路徑。
5. **跨場景框架重複到 2 個是可接受的。** 第 3 個 consumer 出現時才抽共用 module。
6. **單一最高 ROI 的輸入是直接的使用者回饋。** 一句 user note 可在數個 iter 之中揭出多個隱藏 bug。在你需要前先建好 `feedback_inbox.md` 通道。
7. **README rot 是真風險。** 一個會改 code 但沒有「同步 README」硬規則的迴圈，會讓 README 持續宣稱專案缺乏一些早就 ship 的 feature。
8. **100% keep 率是異味，不是勝利。** 通常代表 agent 只挑經過預先過濾的安全項目。`cadence.exploratory_every_n_iters` 是解藥 — 接受較高 discard 率作為代價。
