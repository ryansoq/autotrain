# autotrain — 自主迴圈協定

讓 LLM agent 對著明確目標**手術式**地反覆做實驗，直到觸發停止條件的協定。

最初為 ML 超參數搜尋而寫（一晚最小化 `val_bpb`）。在另一個遊戲開發專案上實際跑過後（一個 Godot RPG demo 跑了 108 次 iter），把**兩個領域都活下來**的紀律萃取成單一通用基底：`program_base.md`。

模式一句話：每個專案 **一個 `program.md` + 一個 `results.tsv`**。下游專案複製 `program_base.md`、填好 `{SLOT}`、跑迴圈。

## 檔案

### 標準基底（從這裡開始）

| 檔案 | 用途 |
|---|---|
| **`program_base.md`** / `program_base_zh.md` | **通用基底。** 從兩次實際 run 萃取。複製 → 填 `{SLOT}` → 跑。 |
| `progress.png` | autotrain 一次 run 的範例圖：83 次實驗、15 個被保留、baseline 0.998 → 0.977 val_bpb。 |
| `CLAUDE.md` | 給在此目錄開啟的 Claude Code 實例的入門指引。 |

### 歷史版（保留作 diff／設計沿革）

| 檔案 | 世代 |
|---|---|
| `program.md` / `program_zh.md` | v0 — 原始 ML-only 協定。 |
| `program_v1.md` / `program_v1_zh.md` | v1 — 加上 Hypothesis 紀律、seed re-check、週期性回顧。 |
| `program_v3.md` / `program_v3_zh.md` | v3 — 加上 Karpathy LLM-coding 準則（Surgical diffs、Tripwire、Open Questions、soft cap）。 |
| `skills/karpathy-guidelines/SKILL.md` | 啟發 v3 的源 skill。**刻意不在迴圈內載入** — 它的第 1 條「不確定就問」會凍住一個永不停的迴圈。 |

歷史鏈：`program.md → v1 → v3 → program_base.md`。新專案請用 `program_base.md`；v0/v1/v3 只保留讓設計沿革可稽核。

## `program_base.md` 是什麼

單一檔案兩個半部：

**通用核心**（逐字複製到你專案的 `program.md`）
- 每次 commit 的 Hypothesis 區塊（四行：Hypothesis / Assumptions / Prediction / Tripwire）
- Surgical diffs — 每一行都追溯到 hypothesis；清理走獨立 commit
- Simplicity bias 加 50 行 soft cap，用 `git diff --stat -w` 量
- Open Questions 紀律 — 把疑問寫進 `journal.md`，永遠不問人類
- Journal 是活的記憶（Open Questions + Reflections）
- 迴圈骨架

**Project slots**（你必須填）

`{MISSION}`、`{SCOPE}`、`{HARD_CONSTRAINTS}`、`{METRIC}`+`{DECISION_RULE}`、`{VALIDATION}`、`{ITERATION_BUDGET}`、`{NOISY}`+`{SMALL_WIN_THRESHOLD}`、`{STOPPING_CONDITIONS}`、`{REFLECTION_CADENCE}`、`{EXPLORATORY_EVERY}`+`{CONSOLIDATION_EVERY}`、`{IDEA_SOURCES}`。

每個 slot 都有兩個案例的填法範例（autotrain ML 與 pikachu-godot game-dev），新專案 pattern-match 就能填好自己的 `program.md`。

## 怎麼用

1. 讀 `program_base_zh.md`（或 `program_base.md`）。
2. 複製到你的專案根目錄為 `program.md`。
3. 填好每一個 `{SLOT}` — 大多有兩個參考範例可以模仿。
4. 填完後刪掉「如何使用這個基底」標頭與案例範例。
5. 開始迴圈。每次 iter 重讀 `program.md` 作為唯一真相、追加到 `results.tsv`、依設定的 cadence 更新 `journal.md`。

迴圈永遠不問人類。只在你定義的 `{STOPPING_CONDITIONS}` 任一觸發時才停。

## 從實戰回流到基底的學到

pikachu-godot 那次 run 產出 8 條通用 lessons，現在都寫進 `program_base.md`。最吃重的幾條：

- **100% keep 率是異味，不是勝利。** 通常代表 agent 只挑經過預先過濾的安全項目。基底現在強制一條 `EXPLORATORY_EVERY` cadence，定期強迫降低接受門檻的 iter。
- **只增不減也是異味。** 108 次遊戲 iter 沒有任何一次「刪 code 拿 keep」。基底現在強制一條 `CONSOLIDATION_EVERY` cadence，目標是負 LOC delta。
- **README rot 是真的。** pikachu-godot 的 README 宣稱 14 個功能「not in scope」，但其實都已 ship。基底建議把 README 同步列入硬規則。
- **直接的使用者回饋是最高 ROI 的輸入。** pikachu 一句 user note 在三個 iter 之中揭出 4 個隱藏 bug。基底現在建議 `feedback_inbox.md` 通道作為最高優先序的 idea source。
- **size cap 用 `git diff --stat -w`。** 縮排重排會讓原始 stats 暴漲（純空白變化）。
