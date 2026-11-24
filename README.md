# orca-implement：用 Orca 把規格一張一張票做完

> 最後更新：2026-11-24。測過的版本：Orca 1.4.272、Claude Code、Antigravity CLI（`agy`）1.5.16。
> 標「未實測」的段落是依官方文件推論，還沒實際跑過。

這個 repo 放我們自己寫的 `orca-implement` skill，還有整套開發流程的說明。

```text
討論需求 ──> /to-spec ──> /to-tickets ──> /orca-implement .scratch/<功能>
（一題一題）   寫規格        切成票           │
                                              ├─ 票 01：開一個 child 做 → 驗收者自己跑測試、看畫面 → 通過／退回
                                              ├─ 票 02：同上（01 通過才開始）
                                              └─ 全部通過 → 你同意後才合併到 main
```

| 角色 | 是誰 | 做什麼 | 不做什麼 |
|---|---|---|---|
| 驗收者（acceptor） | 你正在對話的 agent（Claude Code 或 agy） | 派工、回答 child 的技術問題、**自己**跑測試和看畫面、寫驗收紀錄、合併（要你同意） | 不寫產品程式 |
| child | Orca 開的另一個 agent（Claude Code／agy／Codex） | 一次做一張票：照 `implement` skill 做（裡面會用 `tdd`、`code-review`），commit 在同一個分支，做完回報 `worker_done` | 不合併、不 push、不碰 main |

---

## 1. 安裝（每台電腦做一次）

### 1-1. 基本工具

| 工具 | 安裝 | 檢查 | 官方文件 |
|---|---|---|---|
| Orca | `brew install --cask stablyai/orca/orca`（要打全名，`orca` 是別的套件） | `orca status --json` 的 `runtime.state` 是 `ready` | https://orca.dev/docs/install |
| Node.js 22.20 以上 | 用你習慣的方式（nvm、brew） | `node --version` | — |
| Claude Code（要用 Claude 才裝） | `curl -fsSL https://claude.ai/install.sh \| bash` | `claude --version` | https://code.claude.com/docs/en/setup |
| Antigravity CLI（要用 agy 才裝） | `curl -fsSL https://antigravity.google/cli/install.sh \| bash`，裝在 `~/.local/bin/agy` | `agy --version`、`agy models` | https://antigravity.google/docs/getting-started |

Orca 的派工功能（orchestration）文件寫在 **Settings → Experimental** 開啟（https://orca.dev/docs/cli/orchestration）；文件也註明這個開關暫時不在畫面上，以 `orca status --json` 正常為準。

### 1-2. Skill

每個 agent 讀 skill 的資料夾不一樣，所以安裝時要用 `-a` 指定 agent。**兩種 agent 都用的人，兩行都要跑。**

```bash
# Matt Pocock 的 skill（/to-spec、/to-tickets、/implement、/tdd、/code-review…）
npx skills@latest add mattpocock/skills -g -a claude-code       # 給 Claude Code
npx skills@latest add mattpocock/skills -g -a antigravity-cli   # 給 agy

# Orca 的 skill（orchestration、orca-cli），會裝給這台電腦偵測到的每個 agent
orca skills install --skill orchestration --skill orca-cli

# 這個 repo 的 orca-implement
npx skills@latest add qaz26688442/orca-implement -g -a claude-code -a antigravity-cli
```

- Claude Code 也可以改用 plugin 裝 Matt Pocock 的 skill：`claude plugin install mattpocock-skills@claude-plugins-official`。**plugin 和 `npx skills` 只選一種**，兩種都裝會出現兩份一樣的 skill（例如 `tdd` 和 `mattpocock-skills:tdd`）。
- 第一次在一個 repo 用 Matt Pocock 的 skill，要先跑一次 `/setup-matt-pocock-skills`，設定規格和票放哪裡。選「本機 markdown」時，規格和票放在 `.scratch/<功能>/`，設定寫在該 repo 的 `docs/agents/issue-tracker.md`。

### 1-3. Skill 放在哪裡

| skill | 用途 | Claude Code 讀的位置 | agy 讀的位置 | 來源 |
|---|---|---|---|---|
| `to-spec` | 把討論整理成規格 | `~/.claude/skills/to-spec/SKILL.md` | `~/.gemini/config/skills/to-spec/SKILL.md` | [mattpocock/skills](https://github.com/mattpocock/skills/blob/main/skills/engineering/to-spec/SKILL.md) |
| `to-tickets` | 把規格切成票 | `~/.claude/skills/to-tickets/SKILL.md` | `~/.gemini/config/skills/to-tickets/SKILL.md` | [mattpocock/skills](https://github.com/mattpocock/skills/blob/main/skills/engineering/to-tickets/SKILL.md) |
| `implement` | child 照票開發 | `~/.claude/skills/implement/SKILL.md` | `~/.gemini/config/skills/implement/SKILL.md` | mattpocock/skills |
| `tdd` | 先寫會失敗的測試再寫程式 | `~/.claude/skills/tdd/SKILL.md` | `~/.gemini/config/skills/tdd/SKILL.md` | mattpocock/skills |
| `code-review` | 依程式規範和規格審查 diff | `~/.claude/skills/code-review/SKILL.md` | `~/.gemini/config/skills/code-review/SKILL.md` | mattpocock/skills |
| `grill-me`、`prototype`、`domain-modeling` | 討論階段用 | `~/.claude/skills/<名稱>/` | `~/.gemini/config/skills/<名稱>/` | mattpocock/skills |
| `orchestration`、`orca-cli` | 讓 agent 會用 `orca orchestration …` | `~/.claude/skills/<名稱>/` | `~/.gemini/config/skills/<名稱>/` | Orca（`orca skills install`） |
| `orca-implement` | 驗收者跑接力 | `~/.claude/skills/orca-implement/SKILL.md`、`CHILD-NOTES.md` | `~/.gemini/config/skills/orca-implement/` | 這個 repo 的 `skills/orca-implement/` |

補充：

- `npx skills` 的共用資料夾是 `~/.agents/skills/`。只裝給 Claude Code 時，`~/.claude/skills/<名稱>` 會是連到 `~/.agents/skills/<名稱>` 的捷徑；只裝給 agy 時，直接複製到 `~/.gemini/config/skills/`。各 agent 的位置見 https://github.com/vercel-labs/skills/blob/main/docs/agents.md 。
- agy 官方文件列的位置是專案的 `.agents/skills/` 和全域的 `~/.gemini/config/skills/`（https://antigravity.google/docs/skills）。實測 agy 1.5.16 也讀得到 `~/.agents/skills/`，但文件沒寫，不要依賴。
- 標了 `disable-model-invocation: true` 的 skill（`to-spec`、`to-tickets`、`implement`、`grill-me`、`orca-implement`）只能由你打斜線指令叫出來，agent 不會自己用。

---

## 2. 開發流程

### 步驟 1：討論

把需求講清楚。常用：

- `/grill-me`：讓 AI 一直追問，直到每個分支都有答案。
- `/prototype`：做一個丟掉也沒關係的原型，先看畫面或流程對不對。**原型核准後，規格和票都照原型。**
- `/domain-modeling`：要寫 ADR 或更新 `CONTEXT.md` 的用語時用。

### 步驟 2：`/to-spec` 寫規格

- 把剛剛的討論整理成規格，**不會再問你問題**，只會跟你確認「測試要從哪裡切入」（seam）。
- 產出 `.scratch/<功能>/spec.md`：問題、解法、使用者故事、實作決定、測試決定、不做的範圍。
- 我們的習慣：規格裡放**測試表**（每個測試對到一列），需要的話一起寫 ADR，commit 到 main。

### 步驟 3：`/to-tickets` 切票

- 切成「垂直切片」：每張票從資料、程式、畫面到測試都做完一小段，做完就能驗。
- 每張票寫「被哪幾張票擋住」（Blocked by）。
- 先列給你看，問你切太粗還是太細、擋住的關係對不對；你同意後才寫檔。
- 產出 `.scratch/<功能>/issues/01-xxx.md`、`02-xxx.md`…（一張票一個檔）。

### 步驟 4：`/orca-implement .scratch/<功能>`

指令後面可以接這次的規矩，例如：

```text
/orca-implement .scratch/<功能>
- child 用 --agent claude --model claude-sonnet-5-5
- 一張驗收完直接派下一張，每張用繁中回報一行（結論、commit、代決、新增幾個測試、多幾秒）
- 小的設計決定照你的建議先做，回報時列出來；這些要先問我：要花錢的實測、動到正式環境、裝新套件、不能復原的資料、合併
- 測試只跑改到的模組，不跑全套
- 合併：等我同意，只 fast-forward，打 tag <功能>-before／-after，不 push、不部署
```

它會照這個順序做：

| 階段 | 做什麼 | 產出 |
|---|---|---|
| 準備 | 讀規格和所有票，找出現在能做的票；開**一個** worktree 給整份規格；寫 child 必讀的說明；在起點跑一次相關測試，記下本來就紅的（基準） | `implement-notes.md`、Orca Run |
| 派工 | 開一個新的 child，第一段是 `/implement <票> <implement-notes.md>`，並寫出 `implement` skill 檔的路徑；確認 child 真的用指定的 agent 和 model（看回傳的 `launch.effective`） | — |
| 等 | child 有技術問題，驗收者自己回答；會影響畫面或資料的問題，轉來問你 | — |
| 驗收 | 驗收者**自己**看 diff 對每一條驗收條件、跑相關測試和 lint、改畫面的票開瀏覽器看、算這張票多了幾個測試和幾秒 | 票的最後加 `## Acceptance (<日期>)` |
| 結果 | 通過 → 下一張；不通過 → 寫明問題退回修正，再驗一次 | — |
| 收尾 | 全部票通過；盤點這次新增的測試（留／合併／刪，刪的要你看過）；你同意後才合併 | 每張票一行的總結 |

做完後的資料夾：

```text
.scratch/<功能>/
  spec.md
  issues/
    01-xxx.md          ← 最後會多兩段：## Implementation notes（child 寫）、## Acceptance（驗收者寫）
  implement-notes.md   ← child 必讀：在哪裡工作、先讀什麼、測試指令、結束前要做的事
  acceptance/          ← 驗收腳本、截圖
```

---

## 3. 各種組合怎麼用

驗收者和 child 可以是不同的 agent。下面五種情境，第一欄是你在哪裡打 `/orca-implement`。

| 情境 | 驗收者 | child | child 參數 | 狀態 |
|---|---|---|---|---|
| A | Claude Code | Claude Code | `--agent claude --model claude-sonnet-5-5` | 主要用法，跑過很多次 |
| B | Claude Code | agy（Gemini） | `--agent antigravity --model gemini-4-neon-high` | 流程走得通，試跑時 Gemini 額度不夠沒做完 |
| C | agy | agy | `--agent antigravity`（不帶 model＝用 agy 設定的預設） | 未實測 |
| D | agy | Claude Code | `--agent claude --model claude-sonnet-5-5` | 未實測 |
| E | 任一 | agy 裡的 Claude model | `--agent antigravity --model claude-sonnet-5-5-medium` | 試跑通過（Claude Code 驗收） |

### 情境 A：全部用 Claude Code（目前主要用法）

```text
# 在 Claude Code 裡
/orca-implement .scratch/<功能>
- child 用 --agent claude --model claude-sonnet-5-5
```

- skill 預設不帶 `--model`，child 會沿用你的預設 model（通常是 Opus，比較貴）。**請明寫 `claude-sonnet-5-5`。**
- 用的是 Anthropic 的方案額度。

### 情境 B：Claude Code 驗收、agy 做（child 用 Gemini）

```text
# 在 Claude Code 裡
/orca-implement .scratch/<功能>
- child 用 --agent antigravity --model gemini-4-neon-high
```

- 先確認 agy 讀得到 skill：`ls ~/.gemini/config/skills/` 要看到 `implement`、`tdd`、`code-review`、`orchestration`。
- Orca 開 agy 時會自動加 `--dangerously-skip-permissions`，child 不會卡在等你按允許。
- 可用的 model id 用 `agy models` 查。
- 用的是 Google 帳號的 Antigravity 額度，額度不夠會出現 429（見第 4 節）。

### 情境 C：全部用 agy（未實測）

```text
# 在 agy 裡
/orca-implement .scratch/<功能>
- child 用 --agent antigravity --model gemini-4-neon-high
```

- agy 會把 skill 變成同名的斜線指令，所以打法跟 Claude Code 一樣。
- 驗收者要自己跑測試、看畫面、寫驗收紀錄，工作量比 child 大，建議用強一點的 model。
- 驗收者和 child 共用同一個 Google 帳號額度，同時跑容易撞到 429。

### 情境 D：agy 驗收、Claude Code 做（未實測）

```text
# 在 agy 裡
/orca-implement .scratch/<功能>
- child 用 --agent claude --model claude-sonnet-5-5
```

- 適合 Gemini 額度要留給驗收、寫程式交給 Claude 的時候。
- 這台電腦要同時裝好、登入 Claude Code 和 agy，兩邊的 skill 都要裝（第 1-2 節兩行都跑）。

### 情境 E：用 agy 跑 Claude model

agy 的 model 清單裡也有 Claude（`claude-sonnet-5-5-medium`、`claude-opus-5-5-high` 等），走的是 Google 帳號的額度，不是 Anthropic 的。Gemini 額度用完時可以改用這個。

```text
- child 用 --agent antigravity --model claude-sonnet-5-5-medium
```

### 試跑結果（2026-11-24，情境 B／E）

在一個專案裡開一個不合併的 worktree，寫一份很小的規格和一張票（給票號和標題產生票檔名），由 Claude Code 當驗收者，派給 agy child。

| 次數 | child 參數 | 結果 |
|---|---|---|
| 1 | `--agent antigravity --model gemini-4-neon-high` | 一開始就停：`Your AI credits balance is too low to continue`（紀錄是 429 額度用完） |
| 2 | `--agent antigravity --model gemini-3.8-flash-high` | 同上 |
| 3 | `--agent antigravity --model claude-sonnet-5-5-medium` | 成功：約 4 分鐘做完、兩個 commit、送出 `worker_done`，驗收通過 |

確認到的事：

- Orca 開 agy 的指令是 `agy --dangerously-skip-permissions --model <id>`，`--model` 有傳到，也不會卡在等允許。
- child 有讀 `implement-notes.md`、照 `implement` → `tdd` → `code-review` 的步驟做，`code-review` 真的開了兩個子審查，並採納其中兩項。
- **斜線指令沒有展開**：Orca 會在派工內容前面加一段 worker 說明，`/implement` 不在訊息開頭，所以 agy 不會把它當成指令。child 是靠派工內容寫的 skill 檔路徑（`~/.gemini/config/skills/implement/SKILL.md`）自己讀來照做。`orca-implement` 派工時本來就會寫這個路徑，不用另外處理。
- 驗收者自己跑的測試、規則檢查、lint 都通過，驗收紀錄寫在試跑分支的票檔最後。
- Gemini 額度：同一個 Google 帳號的每個 agy session 共用額度，哪個 model 能用在幾分鐘內會變。試跑當時另一個 agy session 也在跑 `/orca-implement`，兩邊都撞到 429。

---

## 4. 常見問題

| 狀況 | 原因 | 怎麼辦 |
|---|---|---|
| agy child 一開始就停，畫面寫 `Your AI credits balance is too low to continue`，紀錄是 `RESOURCE_EXHAUSTED (code 429)` | 這個 model 的額度暫時用完，免費額度之外又沒有 AI credits 可以接著扣。同一個 Google 帳號的所有 agy session 共用額度 | 等幾分鐘再叫它繼續，或換一個 model（`agy -p "Reply OK" --model <id>` 可以先試哪個還能用）。不要讓好幾個 agy session 同時跑 |
| child 用的 model 不是你要的 | 派工沒帶 `--model` | 派完看回傳的 `launch.effective`；Claude child 可用 `ps -o command= -p <pid>` 看有沒有 `--model` |
| 退回修正時 child 換了 model | child 已經 released，再 `send` 會被重新叫醒但不帶 `--model` | 先 `orca orchestration worker-list` 看狀態：reclaimable 可以直接 `send`；released 就另開一個修正派工 |
| child 說「自我審查過了」 | 沒真的跑 `code-review` skill | 驗收時退回同一個 child 補跑 |
| agy child 回報 `/implement` 沒展開 | Orca 在派工內容前面加了一段說明，斜線指令不在訊息開頭（Claude child 也一樣） | 正常。`orca-implement` 派工會寫 skill 檔路徑，child 自己讀來照做；確認 `~/.gemini/config/skills/implement/` 存在即可 |

其他經驗：

- 每張票只跑改到的模組的測試，全套留到最後；驗收者不要在 child 跑測試時同時跑，共用的測試資料庫會互相干擾。
- 測試全過不代表程式載入得起來：pytest 的載入順序可能藏住 import 循環。改到產品程式的票，驗收時在新程序做一次載入檢查（Django 用 `python manage.py check`）。
- 專案自己的規矩（測試指令、要重用的元件、不能碰的檔案）寫在該 repo 的 `AGENTS.md`，驗收者會把 child 需要的部分寫進 `implement-notes.md`。

---

## 5. 官方文件

| 主題 | 連結 |
|---|---|
| Claude Code 安裝 | https://code.claude.com/docs/en/setup |
| Claude Code skill（放哪裡、`/名稱` 呼叫、`disable-model-invocation`） | https://code.claude.com/docs/en/skills |
| Claude Code 指定 model | https://code.claude.com/docs/en/model-config |
| Orca | https://github.com/stablyai/orca 、https://orca.dev/docs |
| Orca 派工指令 | https://orca.dev/docs/cli/orchestration |
| Orca skill 安裝 | https://orca.dev/docs/cli/skills |
| `skills` 安裝工具 | https://github.com/vercel-labs/skills |
| 各 agent 的 skill 位置 | https://github.com/vercel-labs/skills/blob/main/docs/agents.md |
| Matt Pocock skills | https://github.com/mattpocock/skills |
| Antigravity CLI 安裝 | https://antigravity.google/docs/getting-started |
| Antigravity CLI 指令 | https://antigravity.google/docs/cli/reference.md |
| Antigravity skill 位置 | https://antigravity.google/docs/skills |
| Antigravity model | https://antigravity.google/docs/models |

## 6. 改這個 skill

改 `skills/orca-implement/` 底下的檔案，commit、push 到這個 repo。其他人重跑第 1-2 節的安裝指令就會更新。
