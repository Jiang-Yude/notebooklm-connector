# notebooklm-connector

讓 [Codex CLI](https://github.com/openai/codex) 與 [Claude Code](https://docs.anthropic.com/claude-code) 透過 MCP 直接連到你自己的 [Google NotebookLM](https://notebooklm.google.com/) 做問答查詢的技能包。

**Codex 與 Claude Code 同等支援**，兩邊都有對應安裝路徑。

## 為什麼做這個技能包

「把一份資料做成自己的 AI 顧問」現在大致有三條主線：

1. **最簡單**：直接把專案丟進 ChatGPT。
2. **NotebookLM + Gemini**：NotebookLM 本身只負責「找」（記憶層，可以塞很多、找得到），思考分析靠 Gemini，兩者已連動。問題是 Gemini 目前在深度文件分析上偏弱。
3. **本技能包這條**：用 **Claude / Codex 連 NotebookLM**。NotebookLM 當記憶層，Claude / Codex 負責推理與深度分析。

但第 3 條有個現實成本：**資料量太大時，直接讓 Claude 連 NotebookLM 很燒 token**。所以實際工作流會分大小：

- **少量資料**：Claude / Codex 直接讀 NotebookLM
- **大量資料**（例如把一個 300 支影片的 YouTube 頻道全部用 ExportKit 加進 NotebookLM）：先用 **Codex 做前段清理**（拔紋、分類、上標籤、存進本地知識庫），再用 **Claude** 讀整理後的本地知識庫做深度分析

換句話說，**NotebookLM 是記憶與檢索層，Codex 是清理層，Claude 是推理層**。本技能包做的事就是把「記憶層 ↔ 推理層」這一段橋接好，讓你在 Claude Code 或 Codex 對話裡可以直接呼叫到自己的 NotebookLM。

## 它能做什麼

- 在 Codex / Claude Code 對話裡直接問你 NotebookLM 裡的內容
- 把 NotebookLM 內容匯出回本地（chats、notes、reports、mindmap、影片摘要等）
- 批次管理 NotebookLM 的 sources（列、加、刪）

## 它不做什麼

- 不取代 NotebookLM 本身
- 不做高頻、大量、無人值守的商用流程（這些請走 Google 官方方案）
- 不夾帶任何個人筆記本內容、帳號、Cookie

## MCP 是什麼？（如果你沒聽過）

MCP（Model Context Protocol）是一個開放協議，讓 LLM Agent（像 Codex / Claude Code）可以連到外部資料來源或工具。一句話：**裝了 MCP server 之後，Codex / Claude Code 就能用「工具」去讀寫外部系統**，本來只能聊天的 LLM 變成能真的查你 NotebookLM 的內容。

本技能包做的事：幫你把 `PleasePrompto/notebooklm-mcp` 這個社群維護的 MCP server 接到你的 Codex / Claude Code。

## 安裝前置

- macOS 或 Linux（Windows 走 [WSL](https://learn.microsoft.com/zh-tw/windows/wsl/install)）
- [Codex CLI](https://github.com/openai/codex) **或** [Claude Code](https://docs.anthropic.com/claude-code) 任一個（兩個都裝也可以）
- Node.js 18+（macOS 用 `brew install node`，或從 [nodejs.org](https://nodejs.org/) 下載）
- Google Chrome（給 MCP 做 browser automation 用，目前**只支援** Chrome 本體）
- 一個能登入 NotebookLM 的 Google 帳號
- （可選）[`uv`](https://docs.astral.sh/uv/) — 給安裝 `nlm` CLI 用

⚠️ **安全說明**：`PleasePrompto/notebooklm-mcp` 是用 Chrome automation 登入你的 Google 帳號。登入 session 存在你本機 Chrome profile，**不經第三方伺服器**。你的 Google 密碼也不會輸入到任何腳本裡。但這仍是非官方工具，請自己評估是否接受。

## 安裝步驟

### Step 1：把這個技能包放進你的 skills 目錄

#### Codex 用戶

依你 Codex 配置的 skills 目錄放（看 Codex 文件，多半是 `~/.codex/skills/` 或專案目錄）。例如：

```bash
mkdir -p ~/.codex/skills
cd ~/.codex/skills
git clone https://github.com/<owner>/notebooklm-connector.git
```

> 註：`~/` 在 macOS / Linux 是你的家目錄（例如 `/Users/yourname/` 或 `/home/yourname/`）。

#### Claude Code 用戶

```bash
mkdir -p ~/.claude/skills
cd ~/.claude/skills
git clone https://github.com/<owner>/notebooklm-connector.git
```

#### 沒有 git 的話

去 GitHub repo 下載 zip → 解壓 → 把整個 `notebooklm-connector/` 資料夾搬到上面對應的 skills 目錄。

### Step 2：跑安裝腳本

```bash
cd ~/.codex/skills/notebooklm-connector  # 或 ~/.claude/skills/notebooklm-connector
bash scripts/install.sh
```

⚠️ **第一次跑會比較慢**：腳本會去 npm 拉 `notebooklm-mcp` package（30 秒到 2 分鐘，看網速），不要中斷。

腳本會：

- 偵測你裝的是 Codex CLI、Claude Code 還是兩者都有
- 對 Codex 自動跑 `codex mcp add`（直接寫 Codex 設定檔）
- 對 Claude Code 印出要手動貼進 `~/.claude/settings.json` 的 JSON 片段（不會自動覆蓋你既有設定）
- 偵測 `uv`，若有則安裝或升級 `nlm` CLI

### Step 3：設定 MCP

#### Codex 用戶（推薦）

腳本已自動跑 `codex mcp add notebooklm -- npx -y notebooklm-mcp@latest`，**不用手動編輯設定檔**。

驗證：

```bash
codex mcp list
```

應該看到 `notebooklm` 出現在清單裡，Status: enabled。

#### Claude Code 用戶（手動）

依 [`references/mcp-setup.md`](references/mcp-setup.md) 把 JSON 片段貼進 `~/.claude/settings.json`。**先複製備份再改**。

### Step 4：NotebookLM ExportKit（可選，若需匯出回本地）

去這裡裝瀏覽器 extension：

- GitHub：https://github.com/kristol07/NotebookLM-ExportKit
- 官網：https://notebooklmexportkit.linknows.fun/

這個是純前端 extension，本技能包只負責提示，不能代你裝。

### Step 5：第一次登入

完整關閉並重新開啟你的 Codex 或 Claude Code（很重要，舊 session 不會載新 MCP server）。

在新對話裡輸入：

```text
幫我登入 NotebookLM
```

或英文：

```text
Log me in to NotebookLM
```

會跳出 Chrome 視窗讓你登入 Google。完成登入後關閉視窗即可，session 會被本機 Chrome profile 保存。

## 驗證安裝是否成功

### Codex

```bash
codex mcp list
```

`notebooklm` 應該在清單裡，Status: enabled。

### Claude Code

進入 Claude Code 後輸入 `/mcp` slash command，應該看到 `notebooklm` 在清單裡。

### 端到端測試

開新對話：

```text
列出我的 NotebookLM 筆記本
```

應該回傳你 Google 帳號下所有 notebook 的清單。沒有回傳就去看 [`references/troubleshooting.md`](references/troubleshooting.md)。

## 使用範例

```text
列出我所有的 NotebookLM 筆記本
```

```text
選擇 notebook「研究筆記」，列出裡面所有 source 的標題
```

```text
在 notebook「研究筆記」裡找跟 X 主題相關的 5 個 source
```

```text
把 notebook「會議記錄」的最新 chat 匯出
```

## 第三方工具與授權

本技能包整合三個獨立專案，本身不重新分發其代碼。各專案授權請自行查證：

| 工具 | Repo | 用途 |
|---|---|---|
| notebooklm-mcp | https://github.com/PleasePrompto/notebooklm-mcp | MCP 對話查詢 |
| NotebookLM-ExportKit | https://github.com/kristol07/NotebookLM-ExportKit | 瀏覽器匯出 |
| notebooklm-mcp-cli | https://github.com/jacob-bd/notebooklm-mcp-cli | Sources 管理 CLI |

⚠️ **重要**：上述工具皆**非 Google 官方** API。`notebooklm-mcp-cli` 明確聲明使用 internal APIs 與 browser cookie extraction，作者表明 *Use at your own risk for personal/experimental purposes*。請自行評估風險。

## 授權

MIT — 見 [LICENSE](LICENSE)。

本技能包是 wrapper / 整合工具，與 Anthropic、OpenAI、Google 皆無從屬關係。

## 問題排查

裝不起來、登入失敗、MCP server 連不上、安裝完還是看不到 notebooklm，先看 [`references/troubleshooting.md`](references/troubleshooting.md)。

## 致謝

感謝 PleasePrompto、kristol07、jacob-bd 三位開發者開源了讓我們能用程式碼接上 NotebookLM 的工具。

---

## 維護者

江昱德（Jiang Yude）<br>
隱性知識提煉師<br>
AI 知識架構師

[知識官網](https://jiangyude.com/) · [Threads](https://www.threads.com/@jiang_yude_coach)
