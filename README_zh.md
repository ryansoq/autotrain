# autotrain — autoresearch 協定

讓 LLM agent 在你睡覺時自主跑一整晚 ML 訓練實驗的協定。

這個 repo **不是**訓練程式碼，而是一份規格：agent 開跑前讀的 playbook，加上迴圈中要遵守的紀律。實際的訓練程式碼（`train.py`、`prepare.py`、`README.md`）放在另一個 repo。Agent 會在那個 repo 上開一個新 branch `autoresearch/<tag>` 操作。

迴圈一句話：調整 `train.py` → 跑固定 5 分鐘實驗 → 用單一指標（`val_bpb`）判 `keep` 或 `revert` → 無限重複，直到人類打斷。

## 檔案

| 檔案 | 用途 |
|---|---|
| `program.md` / `program_zh.md` | 原始協定（英 / 繁中）。 |
| `program_v1.md` / `program_v1_zh.md` | v1：每次 commit 必附 Hypothesis + Prediction、小幅進步（Δ < 0.002）做二次種子驗證、每 20 次實驗做週期性回顧。 |
| `program_v3.md` / `program_v3_zh.md` | **目前使用版本。** v1 + Karpathy 四條 LLM coding 準則，改寫成不會停下迴圈的版本。 |
| `CLAUDE.md` | 給在此目錄開啟的 Claude Code 實例的入門指引。 |
| `skills/karpathy-guidelines/SKILL.md` | 啟發 v3 的源 skill。保留供參考；**刻意不在實驗迴圈內載入** — 它的第 1 條原則「不確定就問」會凍住一個永不停的迴圈。 |
| `progress.png` | 一次先前 run 的範例圖 — 83 次實驗、15 個被保留的進步、baseline 0.998 → 0.977 val_bpb。 |

## 該用哪個版本

預設用 `program_v3.md`。v1 與原始版保留作為 diff 錨點，不是現役指令。

## v1、v3 在原始版上加了什麼

| 關注點 | 原始 | v1 | v3 |
|---|---|---|---|
| 可稽核性 — 每次 run 為什麼這樣試？ | 隱含在 commit subject | 明確的 `Hypothesis:` + `Prediction:` 兩行 | + `Assumptions:`（哪些前提是吃重的） |
| 抗噪性 — 小幅進步是真的嗎？ | 未處理 | Δ < 0.002 時做二次種子驗證 | 不變 |
| 漂移控制 — 搜尋還在往某個方向走嗎？ | 未處理 | 每 20 次回顧一次 → 寫進 `journal.md` | + `journal.md` 加 *Open questions* 區，疑問延後問 |
| 可詮釋性 — 失敗的 run，到底是想法錯還是順手改的東西壞了？ | 未處理 | 未處理 | **Surgical experimental diffs** 規則 + `Tripwire:` |
| 簡潔偏好 | 質性原則 | 質性原則 | + soft cap：diff > 50 行新增必須在 commit body 辯護 |

所有加法都維持「永不停」的不變式：agent 把疑問書面化，但**永遠不**停下來問人類。

## 怎麼用

1. 讀 `program_v3_zh.md`（或 `program_v3.md`）。
2. 依序走「設定 → 實驗 → 實驗迴圈」。Agent 會自主執行直到被手動停止。

第一次 run 永遠是未修改的 baseline。之後 `train.py` 上一切都是公平遊戲，以規格中的紀律為界。
