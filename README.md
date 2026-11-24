# Orca 多開 agent：讓一個 agent 叫 Claude、Gemini 一起做事

> 最後更新：2026-11-24。測過的版本：Orca 1.4.272、Claude Code、Antigravity CLI（`agy`）1.5.16。
> 每段都寫了實測結果；標「未實測」的是依官方文件推論。

這份文件教你：在 Claude Code 或 agy 裡，透過 Orca 的 `orchestration` skill 叫其他 code agent（Claude Code、agy／Gemini）一起工作，可以一次開好幾個。

最後一章是進階用法：我們自己寫的 `orca-implement` skill，把一份規格的票一張一張交給 worker 做、驗收。

---

## 1. 一張圖看懂

```text
你 ──對話──> 協調者（Claude Code 或 agy，裝了 orchestration skill）
                 │ orca orchestration worker-start --agent claude …
                 ├──> worker 1：Claude Code（Orca 開的新分頁，自己的 worktree）
                 │ orca orchestration worker-start --agent antigravity …
                 └──> worker 2：agy／Gemini
      worker 做完送 worker_done ──> Orca 叫醒協調者 ──> 協調者整理給你
```

| 名詞 | 白話 |
|---|---|
| 協調者 | 你在對話的那個 agent，負責派工、回答問題、收結果 |
| worker | Orca 幫你開的另一個 agent，做一件事就回報 |
| Run | 這一批工作的資料夾，所有 worker 的回報都進這裡 |
| `worker_done` | worker 做完了（附三句話摘要） |
| `question` | worker 有問題要問（worker 端叫 `ask`） |
| `escalation` | worker 卡住或出錯，沒有正常回報 |
| worktree | 每個 worker 自己的工作資料夾和分支，平行改程式時才不會互相踩到 |

你不用自己打 Orca 指令：用白話跟協調者說，協調者會照 `orchestration` skill 下指令。

---

## 2. 安裝（每台電腦做一次）

### 2-1. 工具

| 工具 | 安裝 | 檢查 | 官方文件 |
|---|---|---|---|
| Orca | `brew install --cask stablyai/orca/orca`（要打全名，`orca` 是別的套件） | `orca status --json` 的 `runtime.state` 是 `ready` | https://orca.dev/docs/install |
| Claude Code | `curl -fsSL https://claude.ai/install.sh \| bash` | `claude --version` | https://code.claude.com/docs/en/setup |
| Antigravity CLI | `curl -fsSL https://antigravity.google/cli/install.sh \| bash`，裝在 `~/.local/bin/agy` | `agy --version`、`agy models` | https://antigravity.google/docs/getting-started |

兩個 agent 不一定都要裝，但協調者和 worker 用到的都要裝好、登入過。

Orca 的派工功能文件寫在 **Settings → Experimental** 開啟（https://orca.dev/docs/cli/orchestration）；文件也註明這個開關暫時不在畫面上，以 `orca status --json` 正常為準。

### 2-2. Orca 的 skill

```bash
orca skills install --skill orchestration --skill orca-cli
```

會裝給這台電腦偵測到的每個 agent（https://orca.dev/docs/cli/skills）。

### 2-3. Skill 放在哪裡

| skill | 用途 | Claude Code 讀的位置 | agy 讀的位置 |
|---|---|---|---|
| `orchestration` | 派工、等回報、回答、追加、停止 | `~/.claude/skills/orchestration/` | `~/.gemini/config/skills/orchestration/` |
| `orca-cli` | 開 worktree、操作終端機、Orca 內建瀏覽器 | `~/.claude/skills/orca-cli/` | `~/.gemini/config/skills/orca-cli/` |

兩邊都是連到 `~/.agents/skills/<名稱>` 的捷徑，實際檔案只有一份。檢查：

```bash
ls -l ~/.claude/skills/orchestration ~/.gemini/config/skills/orchestration
```

---

## 3. 四個練習

照順序做一次就會懂。每個練習都寫了：跟協調者說什麼、協調者實際下什麼指令、實測結果。

### 練習 ①：開一個 worker（只讀）

跟協調者說：

```text
用 Orca 開一個 worker（agy，model 用 gemini-3.8-flash-high），
在目前的資料夾讀 README.md，用三句繁體中文回報這個專案在做什麼。不要改任何檔案。
```

協調者會做：

```bash
orca skills get orchestration                       # 先載入跟目前 Orca 版本相符的用法
orca orchestration worker-start --spec-file <派工說明檔> --worktree current \
  --agent antigravity --model gemini-3.8-flash-high --json
```

- Orca 會開一個新分頁跑 worker，你可以在 Orca 裡看它做事。
- worker 做完送 `worker_done`，Orca 會叫醒協調者，協調者再整理給你。

實測：用 Claude worker（`--agent claude --model claude-haiku-5-5`）跑，回報了三句繁中摘要，沒有改檔案。

### 練習 ②：同時開兩個，各做各的

跟協調者說：

```text
同時開兩個 worker：一個 Claude（claude-sonnet-5-5）、一個 agy（gemini-4-neon-high），
各自在新的 worktree 寫一個 word_count 函式和三個檢查，跑過後 commit。
兩個都做完後，比較兩邊的寫法。
```

協調者會用**一個指令**同時開兩個（分開下會一個等一個）：

```bash
orca orchestration worker-start --spec-file a.md --worktree new-child --name demo-a \
  --agent claude --model claude-sonnet-5-5 --json > a.out &
orca orchestration worker-start --spec-file b.md --worktree new-child --name demo-b \
  --agent antigravity --model gemini-4-neon-high --json > b.out &
wait
```

- `--worktree new-child --name <名稱>`：每個 worker 開一個自己的 worktree（在 `~/orca/workspaces/<repo>/<名稱>`），改的程式互不干擾。
- 一次最多開 4 個左右（Orca 的建議）。

實測：用 Claude Sonnet＋Claude Haiku 代替（當天 agy 沒有額度），兩個同時開、各自在自己的 worktree commit、都送回 `worker_done`。agy 當 worker 的部分見第 4 節。

### 練習 ③：做完再交給下一個（分工）

跟協調者說：

```text
等 A 做完，再開一個 worker 審查 A 的 commit。只讀不改，
回報通過或不通過，最多列三個問題，每個附一個例子。
```

協調者會在 A 的 `worker_done` 之後，指定 A 的 worktree 開審查的 worker：

```bash
orca orchestration worker-start --spec-file review.md \
  --worktree "id:<repo id>::<A 的 worktree 路徑>" --agent claude --model claude-haiku-5-5 --json
```

實測：審查 worker 判「通過」，並實際執行程式找出三個邊界情況（特殊控制字元會被當成空白、沒有空白的中文算一個字、傳 `None` 會出錯）。

常見的分工：Claude 寫功能、Gemini 審查；或反過來。

### 練習 ④：中途溝通

| 你想做的事 | 跟協調者說 | 協調者下的指令 | 實測 |
|---|---|---|---|
| 讓 worker 先問再做 | 派工說明寫「開始前先問協調者 XXX」 | worker 用 `ask` 提問 → 協調者收到 `question` → `orca orchestration reply --id <訊息 id> --body "…"`；需要你決定的會先轉問你 | 通過 |
| 做完再追加一件事 | 「請剛剛那個 worker 再回答 XXX」 | `orca orchestration send --to dispatch:<id> --subject "…" --body "…"` | 通過：同一個 worker 接著做（`delivery: follow_up`） |
| 看 worker 做到哪 | 「看一下 worker 現在在做什麼」 | `orca orchestration worker-read --dispatch <id>` | 通過 |
| 停掉 | 「停掉那個 worker」 | `orca orchestration worker-stop --dispatch <id>` | 通過 |
| 收尾 | 「都做完了，收掉」 | `orca orchestration worker-release --dispatch <id>`；不要的 worktree 用 `orca worktree rm --worktree <…> --delete-branch` | 通過 |

注意：

- 用 `new-child` 開的 worker 做完後**不會自己關**，要 `worker-release`。
- `orca worktree rm --delete-branch` 遇到還沒合併的 commit 會保留分支；確定不要了再用 `git branch -D <分支>` 刪。

---

## 4. 換 agent 怎麼寫

| agent | `--agent` | `--model` 例子 | 查可用 model | 額度從哪扣 | 實測 |
|---|---|---|---|---|---|
| Claude Code | `claude` | `claude-sonnet-5-5`、`claude-haiku-5-5`、`claude-opus-5-5` | https://code.claude.com/docs/en/model-config | Anthropic 方案 | 通過 |
| agy（Gemini） | `antigravity` | `gemini-4-neon-high`、`gemini-3.8-flash-high` | `agy models` | Google 帳號 | 啟動正常，當天額度不足沒做完 |
| agy（Claude model） | `antigravity` | `claude-sonnet-5-5-medium` | `agy models` | Google 帳號 | 通過 |

agy 當 worker 時確認到的事：

- Orca 開 agy 的指令是 `agy --dangerously-skip-permissions --model <id>`：`--model` 有傳到，worker 執行指令不會停下來等人按允許。
- `worker_done`、`ask` 都跟 Claude worker 一樣用。
- Gemini 額度不夠時，畫面會寫 `Your AI credits balance is too low to continue`，紀錄是 429（見第 7 節）。

---

## 5. 指令速查

協調者會自己下這些指令；想看底層在做什麼時查這裡。

| 指令 | 用途 |
|---|---|
| `orca status --json` | Orca 有沒有在跑 |
| `orca skills get orchestration` | 載入跟目前 Orca 版本相符的用法說明 |
| `orca orchestration run-create --objective "<目標>"` | 開一個 Run |
| `orca orchestration worker-start --spec-file <檔> --agent <agent> --model <model> --worktree <current｜new-child｜id:…> [--name <名稱>] --json` | 開 worker |
| `orca orchestration check --wait --types worker_done,escalation,question` | 等 worker 回報 |
| `orca orchestration reply --id <訊息 id> --body "…"` | 回答 worker 的問題 |
| `orca orchestration send --to dispatch:<id> --subject "…" --body "…"` | 再交代事情 |
| `orca orchestration worker-read --dispatch <id>` | 看 worker 做到哪 |
| `orca orchestration worker-list --run <run id>` | 列出這批 worker 的狀態 |
| `orca orchestration worker-stop --dispatch <id>` | 停掉 |
| `orca orchestration worker-release --dispatch <id>` | 收掉 |
| `orca worktree rm --worktree <…> --delete-branch` | 刪掉不要的 worktree |

完整說明：`orca orchestration <指令> --help`、https://orca.dev/docs/cli/orchestration 。

---

## 6. 注意事項

- **一定要指定 model**。開完看回傳的 `launch.effective` 或 `model`，確認真的是那個 model。Claude worker 沒指定時會沿用你的預設（通常是比較貴的 Opus）。
- **改程式的 worker 給它自己的 worktree**（`--worktree new-child`）。同一個 worktree 只放唯讀或依序的工作。
- **worker 不會先問就執行指令**：Orca 開 agy 會加 `--dangerously-skip-permissions`，所以派工說明要寫清楚「只能改哪些檔、不能做什麼」。
- **派工說明用白話寫**：Orca 會在派工說明前面加一段 worker 規則，所以裡面的 `/斜線指令` 不會被當成指令。要 worker 照某個 skill 做，就寫出 skill 檔的路徑，例如「讀 `~/.gemini/config/skills/implement/SKILL.md` 照做」。
- **每種 agent 讀自己的 skill 資料夾**：Claude 讀 `~/.claude/skills/`，agy 讀 `~/.gemini/config/skills/`。worker 要用的 skill，要裝在那個 agent 的資料夾。
- **Gemini worker 不要同時開太多**：同一個 Google 帳號的 agy session 共用額度，在 agy 裡選 Claude model 也是扣 Google 的。
- **共用的測試資料庫會互相干擾**：好幾個 worker 同時跑測試前，先確認專案的測試能不能平行跑。

---

## 7. 常見問題

| 狀況 | 原因 | 怎麼辦 |
|---|---|---|
| agy worker 一開始就停，畫面寫 `Your AI credits balance is too low to continue`，紀錄是 `RESOURCE_EXHAUSTED (code 429)` | 這個 model 的額度暫時用完，又沒有 AI credits 可以接著扣。哪個 model 還能用，幾分鐘內會變 | 等一下再叫它繼續、換 model，或改用 Claude worker。可以先用 `agy -p "Reply OK" --model <id>` 試哪個 model 還能用 |
| 協調者收到 `escalation`，`stopReason=errored` | worker 出錯停下，沒送 `worker_done` | 用 `worker-read` 看原因；修好後 `send` 叫它繼續，或 `worker-stop` 後重開 |
| worker 用的 model 不是你要的 | 派工沒帶 `--model` | 派工時明寫 `--model` |
| 追加工作後 worker 換了 model | worker 已經 released，再 `send` 會被重新叫醒但不帶 `--model` | 先用 `worker-list` 看狀態：`reclaimable` 可以直接 `send`；`released` 就重開一個 worker |
| worker 說找不到 skill | skill 沒裝在那個 agent 的資料夾 | 看第 2-3 節的位置；`orca skills install` 會裝給偵測到的每個 agent |

---

## 8. 進階：`orca-implement` 一張票一張票做

上面是「叫 worker 做一件事」。`orca-implement` 是把一整份規格拆成的票，一張一張交給 worker 做，協調者每張都自己驗收後才派下一張。

```text
討論需求 ──> /to-spec ──> /to-tickets ──> /orca-implement .scratch/<功能>
              寫規格        切成票           ├─ 票 01：開 worker 做 → 協調者自己跑測試、看畫面 → 通過／退回
                                             ├─ 票 02：同上（01 通過才開始）
                                             └─ 全部通過 → 你同意後才合併
```

### 8-1. 另外要裝的

需要 Node.js 22.20 以上（`npx skills` 要用）。

```bash
# Matt Pocock 的 skill：/to-spec、/to-tickets、/implement、/tdd、/code-review…
npx skills@latest add mattpocock/skills -g -a claude-code       # 給 Claude Code
npx skills@latest add mattpocock/skills -g -a antigravity-cli   # 給 agy

# 這個 repo 的 orca-implement
npx skills@latest add qaz26688442/orca-implement -g -a claude-code -a antigravity-cli
```

- Claude Code 也可以用 plugin 裝 Matt Pocock 的 skill：`claude plugin install mattpocock-skills@claude-plugins-official`。**plugin 和 `npx skills` 只選一種**，兩種都裝會出現兩份一樣的 skill。
- 第一次在一個 repo 用，要先跑一次 `/setup-matt-pocock-skills`，設定規格和票放哪裡。選「本機 markdown」時放在 `.scratch/<功能>/`，設定寫在該 repo 的 `docs/agents/issue-tracker.md`。

### 8-2. Skill 放在哪裡

| skill | 用途 | Claude Code 讀的位置 | agy 讀的位置 | 來源 |
|---|---|---|---|---|
| `to-spec` | 把討論整理成規格 | `~/.claude/skills/to-spec/` | `~/.gemini/config/skills/to-spec/` | [mattpocock/skills](https://github.com/mattpocock/skills/blob/main/skills/engineering/to-spec/SKILL.md) |
| `to-tickets` | 把規格切成票 | `~/.claude/skills/to-tickets/` | `~/.gemini/config/skills/to-tickets/` | [mattpocock/skills](https://github.com/mattpocock/skills/blob/main/skills/engineering/to-tickets/SKILL.md) |
| `implement` | worker 照票開發 | `~/.claude/skills/implement/` | `~/.gemini/config/skills/implement/` | mattpocock/skills |
| `tdd` | 先寫會失敗的測試再寫程式 | `~/.claude/skills/tdd/` | `~/.gemini/config/skills/tdd/` | mattpocock/skills |
| `code-review` | 依程式規範和規格審查 diff | `~/.claude/skills/code-review/` | `~/.gemini/config/skills/code-review/` | mattpocock/skills |
| `grill-me`、`prototype`、`domain-modeling` | 討論階段用 | `~/.claude/skills/<名稱>/` | `~/.gemini/config/skills/<名稱>/` | mattpocock/skills |
| `orca-implement` | 協調者跑接力 | `~/.claude/skills/orca-implement/`（`SKILL.md`、`CHILD-NOTES.md`） | `~/.gemini/config/skills/orca-implement/` | 這個 repo 的 `skills/orca-implement/` |

補充：

- `npx skills` 的共用資料夾是 `~/.agents/skills/`。各 agent 的位置見 https://github.com/vercel-labs/skills/blob/main/docs/agents.md 。
- agy 官方文件列的位置是專案的 `.agents/skills/` 和 `~/.gemini/config/skills/`（https://antigravity.google/docs/skills）。實測 agy 1.5.16 也讀得到 `~/.agents/skills/`，但文件沒寫，不要依賴。
- 標了 `disable-model-invocation: true` 的 skill（`to-spec`、`to-tickets`、`implement`、`grill-me`、`orca-implement`）只能由你打斜線指令叫出來，agent 不會自己用。

### 8-3. 流程

1. **討論**：`/grill-me` 讓 AI 一直追問到每個分支都有答案；`/prototype` 先做丟掉也沒關係的原型。原型核准後，規格和票都照原型。
2. **`/to-spec`**：把討論整理成 `.scratch/<功能>/spec.md`，不會再問你需求，只確認測試要從哪裡切入。我們的習慣是規格裡放**測試表**。
3. **`/to-tickets`**：切成 `.scratch/<功能>/issues/01-xxx.md`…，每張從資料到畫面到測試做完一小段，寫明被哪幾張票擋住。先列給你看，你同意後才寫檔。
4. **`/orca-implement .scratch/<功能>`**，後面接這次的規矩，例如：

```text
/orca-implement .scratch/<功能>
- worker 用 --agent claude --model claude-sonnet-5-5
- 一張驗收完直接派下一張，每張用繁中回報一行（結論、commit、代決、新增幾個測試、多幾秒）
- 小的設計決定照你的建議先做，回報時列出來；這些要先問我：要花錢的實測、動到正式環境、裝新套件、不能復原的資料、合併
- 測試只跑改到的模組，不跑全套
- 合併：等我同意，只 fast-forward，不 push、不部署
```

| 階段 | 做什麼 | 產出 |
|---|---|---|
| 準備 | 讀規格和所有票，找出現在能做的票；開**一個** worktree 給整份規格；寫 worker 必讀的說明；在起點跑一次相關測試，記下本來就紅的（基準） | `implement-notes.md`、Orca Run |
| 派工 | 開一個新 worker，派工說明寫 `/implement <票> <implement-notes.md>` 和 `implement` skill 檔的路徑 | — |
| 等 | worker 有技術問題，協調者自己回答；會影響畫面或資料的問題轉來問你 | — |
| 驗收 | 協調者**自己**看 diff 對每條驗收條件、跑相關測試和 lint、改畫面的票開瀏覽器看、算多了幾個測試和幾秒 | 票的最後加 `## Acceptance (<日期>)` |
| 結果 | 通過 → 下一張；不通過 → 寫明問題退回修正，再驗一次 | — |
| 收尾 | 全部票通過；盤點新增的測試（留／合併／刪，刪的要你看過）；你同意後才合併 | 每張票一行的總結 |

做完後的資料夾：

```text
.scratch/<功能>/
  spec.md
  issues/
    01-xxx.md          ← 最後會多兩段：## Implementation notes（worker 寫）、## Acceptance（協調者寫）
  implement-notes.md   ← worker 必讀：在哪裡工作、先讀什麼、測試指令、結束前要做的事
  acceptance/          ← 驗收腳本、截圖
```

### 8-4. 協調者和 worker 的組合

| 情境 | 你在哪裡打 `/orca-implement` | worker 參數 | 狀態 |
|---|---|---|---|
| A | Claude Code | `--agent claude --model claude-sonnet-5-5` | 主要用法，跑過很多次 |
| B | Claude Code | `--agent antigravity --model gemini-4-neon-high` | 流程走得通，試跑時 Gemini 額度不夠沒做完 |
| C | agy | `--agent antigravity`（不帶 model＝用 agy 設定的預設） | 未實測 |
| D | agy | `--agent claude --model claude-sonnet-5-5` | 未實測 |
| E | 任一 | `--agent antigravity --model claude-sonnet-5-5-medium` | 試跑通過（Claude Code 當協調者） |

- agy 會把 skill 變成同名的斜線指令，所以在 agy 裡打 `/orca-implement` 跟 Claude Code 一樣。
- 協調者要自己跑測試、看畫面、寫驗收紀錄，工作量比 worker 大，建議用強一點的 model。
- 情境 C、D 要同時裝好兩邊的 skill（8-1 兩行都跑）。

### 8-5. 試跑結果（2026-11-24，情境 B／E）

在一個專案裡開一個不合併的 worktree，寫一份很小的規格和一張票（給票號和標題產生票檔名），由 Claude Code 當協調者，派給 agy worker。

| 次數 | worker 參數 | 結果 |
|---|---|---|
| 1 | `--agent antigravity --model gemini-4-neon-high` | 一開始就停：額度不足（429） |
| 2 | `--agent antigravity --model gemini-3.8-flash-high` | 同上 |
| 3 | `--agent antigravity --model claude-sonnet-5-5-medium` | 成功：約 4 分鐘做完、兩個 commit、送出 `worker_done`，驗收通過 |

- worker 有讀 `implement-notes.md`，照 `implement` → `tdd` → `code-review` 的步驟做；`code-review` 真的開了兩個子審查，並採納其中兩項。
- `/implement` 沒有展開成指令（原因見第 6 節「派工說明用白話寫」），worker 是讀派工說明裡寫的 skill 檔路徑照做的。`orca-implement` 派工本來就會寫這個路徑。
- 協調者自己跑的測試、規則檢查、lint 都通過，驗收紀錄寫在試跑分支的票檔最後。

### 8-6. 經驗

- **worker 一定要真的跑 `code-review`**，只寫「自我審查過了」不算，驗收時看到就退回。
- **每張票只跑改到的模組的測試**，全套留到最後；協調者不要在 worker 跑測試時同時跑。
- **測試全過不代表程式載入得起來**：pytest 的載入順序可能藏住 import 循環。改到產品程式的票，驗收時在新程序做一次載入檢查（Django 用 `python manage.py check`）。
- **專案自己的規矩**（測試指令、要重用的元件、不能碰的檔案）寫在該 repo 的 `AGENTS.md`，協調者會把 worker 需要的部分寫進 `implement-notes.md`。

---

## 9. 官方文件

| 主題 | 連結 |
|---|---|
| Orca | https://github.com/stablyai/orca 、https://orca.dev/docs |
| Orca 派工指令（orchestration） | https://orca.dev/docs/cli/orchestration |
| Orca skill 安裝 | https://orca.dev/docs/cli/skills |
| Claude Code 安裝 | https://code.claude.com/docs/en/setup |
| Claude Code skill（放哪裡、`/名稱` 呼叫、`disable-model-invocation`） | https://code.claude.com/docs/en/skills |
| Claude Code 指定 model | https://code.claude.com/docs/en/model-config |
| Antigravity CLI 安裝 | https://antigravity.google/docs/getting-started |
| Antigravity CLI 指令 | https://antigravity.google/docs/cli/reference.md |
| Antigravity skill 位置 | https://antigravity.google/docs/skills |
| Antigravity model | https://antigravity.google/docs/models |
| `skills` 安裝工具、各 agent 的 skill 位置 | https://github.com/vercel-labs/skills 、https://github.com/vercel-labs/skills/blob/main/docs/agents.md |
| Matt Pocock skills | https://github.com/mattpocock/skills |

## 10. 改這個 skill

改 `skills/orca-implement/` 底下的檔案，commit、push 到這個 repo。其他人重跑 8-1 的安裝指令就會更新。
