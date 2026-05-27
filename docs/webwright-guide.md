# Webwright 研究與個人使用指南

> 個人研究筆記 — 整理 Microsoft Research 於 2026 年 5 月發布的 Webwright,做為日後本機安裝與使用的參考。

## 一、什麼是 Webwright?

**Webwright** 是 Microsoft Research 開源的 **terminal-native web agent framework**。

核心理念:**把 agent 跟 browser 解耦** — agent 不再每一步預測一個瀏覽器動作,而是在 terminal loop 中 **撰寫並執行 Playwright Python 程式碼** 來操作瀏覽器。Browser 是 agent 可以隨時啟動、檢查、丟棄的「工作區」,真正持久的產出是 **程式碼與 log**。

- 授權: MIT
- 語言: Python 3.10+
- 瀏覽器: Chromium(Playwright)
- Repo: https://github.com/microsoft/Webwright

## 二、架構(極簡四件套,約 1300 行)

| 元件 | 行數 | 角色 |
| --- | --- | --- |
| Agent Loop | ~450 | observe → predict code → execute 循環 |
| Playwright Environment | ~570 | 瀏覽器 workspace 管理 |
| CLI | ~150 | 任務進入點 |
| Model Backends | ~150–200 each | OpenAI / Anthropic / OpenRouter |

沒有 multi-agent orchestration、沒有複雜 planning hierarchy — 單一 agent loop 為主。

## 三、Benchmark 表現

| Benchmark | 任務數 | GPT-5.4 | Claude Opus |
| --- | --- | --- | --- |
| Online-Mind2Web | 300 | 86.7% | 84.7% |
| Odysseys(long-horizon) | 200 | 60.1% | — |

Odysseys 60.1% 較 base GPT-5.4 的 33.5% 提升 **26.6 點**,較 prior SOTA 提升 **15.6 點**。

## 四、系統需求

- Python 3.10 或以上
- 能跑 Chromium 的作業系統(macOS / Linux / Windows)
- 任一 LLM provider 的 API key:
  - `OPENAI_API_KEY`(OpenAI)
  - `ANTHROPIC_API_KEY`(Anthropic)
  - 或 OpenRouter token

## 五、安裝步驟(本機)

```bash
git clone https://github.com/microsoft/Webwright.git
cd Webwright

# 建立虛擬環境
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate

# 安裝 Webwright(editable)+ Chromium
pip install -e .
playwright install chromium
```

依賴會自動帶入 `httpx`、`pydantic`、`typer`、`playwright` 等。

## 六、配置 API key

```bash
export OPENAI_API_KEY=sk-...
# 或
export ANTHROPIC_API_KEY=sk-ant-...
```

Repo 內附對應的 config 檔:
- `model_openai.yaml`
- `model_claude.yaml`

可疊加 `base.yaml`(共用設定)+ 選一個 model config。

## 七、跑第一個任務

```bash
python -m webwright.run.cli \
    -c base.yaml -c model_claude.yaml \
    -t "Search for flights from SEA to JFK on 2026-08-15 to 2026-08-20" \
    --start-url https://www.google.com/flights \
    --task-id demo_first \
    -o outputs/default
```

### Flag 說明

| Flag | 用途 |
| --- | --- |
| `-c` | Config 檔(可堆疊,通常 base + model 各一) |
| `-t` | 任務指令(自然語言) |
| `--start-url` | Agent 啟動的初始頁面 |
| `--task-id` | 輸出資料夾名稱(在 `-o` 之下) |
| `-o` | 輸出根目錄(會收 log、screenshots、產生的 Playwright code) |

執行完到 `outputs/default/demo_first/` 看 agent 寫出來的程式碼與 trajectory。

## 八、個人使用情境

### 1. 重複性 web 操作自動化
- 每週/每日從某 dashboard 擷取數據後存 CSV
- 整理多個 GitHub Release notes 成週報
- 批次填寫表單、提交申請

### 2. 長 horizon 研究任務
- 跨多頁面 + 多步驟的資料蒐集(屬 Odysseys 類型,Webwright 強項)
- 比價、訂位、調研競品功能矩陣

### 3. 學習 minimal agent loop 設計
- 全 codebase 約 **1300 行**,適合精讀
- 重點看 Agent Loop 怎麼把「LLM 寫 code → 執行 → 看結果 → 再寫」串成一個收斂的迴圈
- 對比一下 ReAct / function-calling agent 的差異

## 九、使用心得 / 注意事項(預先整理)

- **持久產出是程式而非 session**:跑完一次的價值在於拿到一段可重用的 Playwright 程式碼,下次相同任務可以直接跑程式不再 call LLM
- **長任務成本**:Odysseys 等級的任務 LLM token 用量會偏高,建議先在便宜的 model(Claude Haiku / GPT-mini 類)上 dry run
- **網站防爬**:Cloudflare、Google reCAPTCHA 等仍會擋,跟一般 Playwright 自動化相同的反爬問題都會遇到
- **Headless vs headed**:debug 時打開 headed mode,正式跑再切 headless

## 十、後續可探索方向

- [ ] 把跑出來的 Playwright 程式 cron 化(脫離 LLM,純自動化)
- [ ] 自訂 model backend(例如接本機 Ollama)
- [ ] 寫一兩個自己常用的 task config 模板存起來
- [ ] 對照 OpenAI Operator / Anthropic Computer Use 的差異

## 參考來源

- GitHub: <https://github.com/microsoft/Webwright>
- 官方介紹頁: <https://microsoft.github.io/Webwright/>
- Microsoft Research 文章: <https://www.microsoft.com/en-us/research/articles/webwright-a-terminal-is-all-you-need-for-web-agents/>
- MarkTechPost 報導: <https://www.marktechpost.com/2026/05/24/microsoft-research-releases-webwright-a-terminal-native-web-agent-framework-that-scores-60-1-on-odysseys-up-from-base-gpt-5-4s-33-5/>
- WinBuzzer 報導: <https://winbuzzer.com/2026/05/25/microsoft-webwright-turns-web-agents-into-reusable-code-xcxwbn/>
