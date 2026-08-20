
https://www.aihero.dev/workshops/ai-coding-crash-course 結合官方文件的紀錄

# Smart zone / dumb zone

what the context window is, why it matters, and how to recognise when an agent is still reasoning sharply and when its context has degraded — and how to keep your work in the smart zone

## Context Window 是什麼

Context window 是模型單次對話能「看到」的全部 token 總量，包含：system prompt、工具定義、對話歷史、你貼的程式碼、工具回傳的結果（檔案內容、grep 輸出、bash log）等。在 Claude Code 裡，介面上通常會顯示目前 context 使用率（例如接近上限時會提示 auto-compact）。

## Smart zone / dumb zone 的核心概念

重點不是「有沒有超過 token 上限」，而是**有效推理品質會隨 context 中無關、重複、矛盾、過時的資訊增加而下降**——這常被稱為 _context rot_。

- **Smart zone**：context 內容高度相關、精簡、一致，模型能準確追蹤約束條件、記得先前決策、給出聚焦且正確的回應。
- **Dumb zone**：context 塞滿雜訊（大量原始檔案內容、失敗嘗試的殘留、互相矛盾的指示、離題的對話分支），模型開始「稀釋注意力」——即使技術上還沒到 token 上限，回答品質已經明顯變差。

## 如何辨識 agent 已經進入 dumb zone

留意以下徵兆：

- 重複問已經回答過的問題，或忘記幾輪前設定的限制條件
- 對檔案內容的描述與實際不符（context 太長導致早期讀取的內容被「擠掉注意力」）
- 回應開始前後矛盾，或修正一個問題又引發另一個
- 出現大量「來回打補丁」式的對話——每次修正只解決當下症狀，卻沒解決根因
- 執行了與當前任務無關的動作（引用了早已過時的目標）
- 回應變得又長又發散，抓不到重點

## 在 Claude Code 中如何維持在 smart zone

1. **主動管理 context，不要等 auto-compact**：auto-compact 是安全網，但壓縮過程會有資訊損失。任務告一段落、要換主題時，主動用 `/clear`；context 變大但同一任務要繼續時，用 `/compact`。
    
2. **用 subagent 隔離「重研究、輕結論」的工作**：用 Explore / general-purpose agent 做大範圍搜尋或閱讀大量檔案，讓龐大的中間過程留在 subagent 自己的 context，只把精煉後的結論帶回主線程，避免主 context 被原始搜尋結果灌爆。
    
3. **精準讀取，不要整包塞入**：優先用 grep/Explore 定位到具體行號範圍，再讀取該範圍，而不是 `cat` 整個大檔案；工具回傳結果越精簡，噪音越少。
    
4. **把穩定知識放進 CLAUDE.md / memory，而不是每次重講**：專案慣例、規則、使用者偏好寫進 `CLAUDE.md` 或記憶檔案，讓它在對話一開始就以精簡形式載入，不用每輪重新解釋，也不佔用寶貴的對話輪次。
    
5. **善用 Plan mode 對齊方向再動手**：在生成大量程式碼/工具呼叫之前，先用 plan 確認方向，避免走錯路後產生大量需要在對話中「撤銷」的殘留內容。
    
6. **任務轉向時，開新對話而非硬拗舊的**：如果任務性質整個變了（例如從 debug 換成完全不相關的新功能），與其在同一條長對話裡繼續，不如 `/clear` 重開——舊的錯誤嘗試、過時假設留在 context 裡只會拖累新任務的推理。
    
7. **觀察對話節奏**：如果發現自己（或 agent）陷入「越改越亂」的來回修正循環，這通常是 dumb zone 的訊號——這時停下來、`/compact` 或重新以精簡指令描述現況，往往比繼續在原地打補丁更有效。
    

一句話總結：**context window 的大小是硬限制，但「smart zone」的邊界通常來得更早**——目標是讓進入 context 的每一段內容都保持高相關性、低冗餘，並主動而非被動地清理，而不是把管理 context 的責任全部交給 auto-compact。

---

# Managing context

what's actually eating up the agent's context, why it fills up, how to keep it slim, and how to read its exact status instead of guessing.

## Context 都被什麼吃掉?

Claude Code 的 context window(預設約 200K tokens,部分模型有 1M beta 版本)主要被以下幾類東西佔用:

1. **System prompt** — Claude Code 本身的固定指令
2. **工具定義(schema)** — 每個內建工具(Read/Edit/Bash…)加上您連接的每個 MCP server 提供的工具,都要把完整 schema 塞進 context。MCP server 接越多,這塊佔用越大
3. **記憶體檔案** — CLAUDE.md、專案/使用者層級的自動記憶檔案,啟動時就載入
4. **對話歷史** — 每一輪的使用者訊息、Claude 的回覆、以及所有 tool_use / tool_result(工具呼叫與其回傳結果)
5. **檔案讀取內容** — Read 工具讀進來的完整檔案內容會整包留在 context 裡
6. **大型工具輸出** — 例如冗長的 bash 輸出、grep 全庫掃描結果、MCP 查詢回傳的大量 JSON
7. **Subagent 回傳結果** — 這裡有個關鍵:subagent(如 Explore、general-purpose)在背景做的所有搜尋、讀檔過程都在「它自己的」context 裡,**不會**污染主 context;只有它最後的文字總結會傳回主對話。這正是您常看到我用 Agent 工具做大範圍搜尋的原因

## 為什麼會滿

- 長時間單一 session、多輪來回累積
- 直接讀大檔案而非用行號範圍鎖定
- 用 cat/大量 bash 輸出而非精準的 grep/搜尋
- 一次連接太多 MCP server(每個都佔用 schema 空間,即使沒用到)
- 反覆貼上大段 log 或大型 JSON 到對話中

## 如何保持精簡

- **善用 subagent 隔離**:大範圍程式碼探索交給 Explore/general-purpose agent,只拿回摘要
- **精準讀檔**:用 Read 的 offset/limit 只讀需要的區段,而非整檔
- **善用 Grep/搜尋工具**取代直接 cat 大檔
- **`/clear`**:切換到不相關任務時清空對話歷史(但保留 CLAUDE.md 等專案設定)
- **`/compact`**:手動觸發摘要壓縮舊對話,保留關鍵資訊釋放空間
- 系統也有 **auto-compaction**:context 使用量接近上限時會自動摘要壓縮;更輕量的 **microcompact** 機制會先嘗試移除舊的工具結果來騰出空間,不必真的做完整摘要
- 只連接當下任務真正需要的 MCP server,減少工具 schema 常駐佔用

## 如何查看「確切」狀態,而非用猜的

用 **`/context`** 指令 —— 會顯示目前 context window 的精確分解表(system prompt、系統工具、MCP 工具、custom agents、memory 檔案、對話訊息等各佔多少 tokens 與百分比),以及已用量/剩餘可用量。這是官方且精確的方式,不需要靠感覺猜測「是不是快滿了」。

另外 **`/usage`** 可查看目前 session 的 token 用量與費用估算,對追蹤消耗量也有幫助但著重在成本而非 context 佔比細節。

---

# Compaction vs. handoff

three ways to continue past a full context, when to use each, and why mid-task compaction goes wrong.


在 Claude Code 中,延續超出 context 上限的工作主要有三種機制,設計目的與適用時機各不相同:

## 三種延續方式

### 1. 自動壓縮(Auto-compaction)

當對話 context 接近上限(約 75% 視窗)時自動觸發,執行與 `/compact` 相同的摘要流程,不需使用者介入。

- **適用時機**:作為安全網,處理你沒預期到、突然逼近上限的情況。

### 2. 手動 `/compact`

主動在對話中下指令,可選擇加上重點方向,例如 `/compact focus on the auth bug`。會把對話歷史轉換成結構化摘要,同時保留啟動內容(CLAUDE.md、MCP 工具設定、auto memory)。

- **適用時機**:**任務與任務之間的空檔**,例如一個功能做完、準備開始下一個獨立任務前。

### 3. Session 分支 / 切換(Handoff)

不壓縮既有內容,而是換一個乾淨或不同的 context:

- `/clear`:清空重來
    
- `--continue` / `--resume`:接續或挑選過去的 session
    
- `/branch`:從目前狀態分叉出新的 context
    
- 搭配 **subagent**:把研究密集型的工作(大量讀檔、探索)丟到獨立的 context 執行,不佔用主 session 的視窗
    
- **適用時機**:切換到性質不同的工作、或需要保留主線乾淨、把大量探索性讀檔隔離開來時。
    

## 兩者的核心差異

| -    | Compaction                  | Handoff(分支/切換)                   |
| ---- | --------------------------- | -------------------------------- |
| 做法   | 把舊對話「壓縮成摘要」,保留在同一 session 內 | 「另起」一個 context,舊的保留不動或被捨棄        |
| 保留內容 | 摘要後的重點,細節被概括化               | 依機制而定,可完整保留原 session(可回頭 resume) |
| 適合情境 | 任務之間的自然斷點                   | 工作性質切換、探索性任務、需要乾淨 context        |
## 為什麼「任務進行到一半」做 compaction 容易出問題

因為壓縮的本質是把逐步的執行細節概括成摘要,而任務進行中最關鍵的正是這些細節:

1. **中間狀態遺失**:工具呼叫的實際輸出、debug 過程中的錯誤訊息,會被濃縮成簡短描述,細節消失。
2. **決策脈絡消失**:「為什麼選這個做法」「哪個方案試過但失敗了」這類推理過程會被泛化,之後不容易還原。
3. **重複做工 / 行為不一致**:恢復後因為缺乏詳細的執行歷史,可能重跑已經失敗過的嘗試,或誤判目前實際進度,導致後續動作偏離原本計畫。

**建議做法**:在任務之間(而非任務中途)主動 `/compact`;研究密集的階段改用 subagent 隔離 context;不相關的新工作用 `/clear`;非預期的溢出就交給自動壓縮處理,而不是自己在關鍵步驟中途強制壓縮。

---

# Codebase exploration

how to help your agent understand an existing codebase so it uses the code that exists instead of reinventing and repeating.


根據 Claude 官方文件整理出的重點，分成「核心心智模型」「工具/機制」「使用者操作技巧」「常見失敗模式」四塊。

## 核心心智模型：agentic search，不是 RAG

Claude Code 不維護中央向量索引，而是像人類工程師一樣「即時探索」：走訪檔案系統、讀檔、用 grep 找東西。這樣的好處是不會有嵌入索引跟不上程式碼變動的問題，但代價是探索本身會大量消耗 context window——這是所有後續建議的出發點。

## 用來「幫助 agent 理解既有程式碼」的七種機制

| 機制               | 定位                                     | 何時用                                                                           |
| ---------------- | -------------------------------------- | ----------------------------------------------------------------------------- |
| **分層 CLAUDE.md** | 每次 session 自動載入的靜態上下文，根目錄放大局觀、子目錄放局部慣例 | 「Claude 猜不到」的規則（指令、命名慣例、環境變數），不是可以從程式碼推得的東西                                   |
| **Skills**       | 按需載入的可重用知識/工作流程包                       | 只在特定任務類型才相關的專業知識，避免塞爆每次都載入的 CLAUDE.md                                         |
| **Subagents**    | 獨立 context window 的子代理，只回傳結果摘要         | 把「探索」跟「編輯」分開——子 agent 負責讀一堆檔案畫出架構圖，主 agent 帶著精簡結論去改程式碼，不會把大量原始檔案內容塞進主 context |
| **MCP Servers**  | 連接 Claude 原生讀不到的內部工具/資料源               | 進階用法是把「結構化搜尋」本身包成一個 MCP tool 讓 Claude 直接呼叫，而非每次都用 grep 硬翻                     |
| **LSP 整合**       | 符號級（symbol-level）而非文字級的程式碼導航           | 大型/編譯型語言（如 C/C++）特別重要，避免同名不同函式互相誤導（看起來是透過類似型別解析的方式探索程式碼）                      |
| **Hooks**        | 在關鍵時機強制執行的腳本，具確定性                      | 讓「探索完要更新文件」「進場先載入團隊上下文」這類事情不靠 Claude 記得，而是自動發生                                |
| **Plugins**      | 把上述東西打包成可分發的單元                         | 避免好的探索設置停留在個人/單一團隊，變成組織級標準配置                                                  |

此外還建議維護一份 **codebase map**（目錄結構的 Markdown「目錄表」），存進 repo、跟著版本控管，可以避免每個 session 都重新探索一次同樣的架構。

## 促使 Claude「用既有的、別重造」的具體操作技巧

官方 best-practices 文件特別點出的 prompting 模式：

1. **Explore → Plan → Code → Commit** 四階段工作流。用 plan mode（`Shift+Tab`）先讓 Claude 純讀檔案、回答問題，不動手改，逼它先建立對既有程式碼的理解再進入實作。
    
2. **明確指向既有 pattern**，而不是描述目標本身。文件給的對比範例很直接：
    
    > 差：「add a calendar widget」 
    > 
    > 好：「look at how existing widgets are implemented on the home page... HotDogWidget.php is a good example. follow the pattern... build from scratch without libraries other than the ones already used in the codebase.」
    
    關鍵在於**主動點名一個已存在的範例檔案**當作 pattern 來源，而不是讓 Claude 自己去猜要不要重用。
    
3. **用 subagent 做「重用性調查」**，把探索跟後續實作的 context 隔開：
    
    > 「Use subagents to investigate how our authentication system handles token refresh, and whether we have any existing OAuth utilities I should reuse.」
    
    這樣一來，主對話的 context 不會被一堆讀檔案的過程污染，只拿到「有沒有現成東西可用」的結論。
    
1. **像問資深工程師一樣直接發問**：「How does logging work?」「What edge cases does X handle?」——不需要特殊 prompt 技巧，直接問即可。
    
## 常見失敗模式（文件明確列出）

- **infinite exploration**：叫 Claude 去「調查」卻沒有界定範圍，結果讀了幾百個檔案把 context 塞爆 → 對策：narrow scope 或改用 subagent 隔離。
- **over-specified CLAUDE.md**：塞太多東西進去，重要規則反而被稀釋、被忽略 → 對策：每條規則自問「拿掉這行 Claude 會不會做錯」，不會就刪。
- **探索與編輯混在同一個 context**：導致主 session 的 context 被大量讀檔內容佔滿，後續編輯品質下降 → 對策：探索交給 subagent，只留結論。

## 治理層面的建議（組織規模）

文件建議統一管理 CLAUDE.md、權限、marketplace、慣例，避免各自重複打造同樣的探索設置；並建議每 3–6 個月或模型大版本升級後，重新審視 CLAUDE.md 是否有過時規則


----

# Progressive disclosure 漸進式揭露

structuring those instruction files so the agent loads only what it needs, keeping context lean.

根據 Anthropic 官方文檔（Agent Skills Overview 與 Skill Authoring Best Practices），progressive disclosure 是靠檔案系統架構 + 三層載入模型實作出來的，具體做法如下：

## 核心機制：三層載入

Skill 是一個資料夾，Claude 用 bash 在檔案系統上導覽它，而不是把整包內容塞進 context。三層分別是：

|層級|何時載入|Token 成本|內容|
|---|---|---|---|
|**Level 1：Metadata**|啟動時就載入|每個 skill 約 100 tokens|YAML frontmatter 的 `name` + `description`|
|**Level 2：Instructions**|Skill 被觸發時|建議 < 5k tokens|SKILL.md 本文|
|**Level 3：Resources/Code**|按需載入|不讀就是 0|額外的 .md 參考檔、腳本、範本|

關鍵在於：description 是 Claude 用來比對使用者請求、決定「要不要觸發」的依據，所以在觸發之前，不管你 bundle 了多少內容，都只佔那 ~100 tokens。一旦觸發，Claude 是用 `cat SKILL.md` 這類 bash 指令把內容讀進 context——這代表它是「主動選擇要讀什麼」，不是被動接收。

## 實作上的四個具體做法

**1. description 要同時寫「做什麼」和「何時用」**

這是唯一決定 Level 1→2 是否觸發的欄位，必須用第三人稱、包含具體觸發詞：

```yaml
description: Extract text and tables from PDF files, fill forms, merge documents. Use when working with PDF files or when the user mentions PDFs, forms, or document extraction.
```

避免「Helps with documents」這種模糊寫法——Claude 要從可能上百個 skill 裡選對的,description 的精確度直接決定準確率。

**2. SKILL.md 本文控制在 500 行以內，只放「目錄式」導引**

超過這個量就該拆檔。SKILL.md 的角色像 onboarding guide 的目錄，不是把所有細節寫滿，而是指向細節檔：

```markdown
## Advanced features
**Form filling**: See [FORMS.md](FORMS.md)
**API reference**: See [REFERENCE.md](REFERENCE.md)
```

**3. 參考檔只能從 SKILL.md 「一層深」引用，不能巢狀**

文檔特別強調這點：如果 SKILL.md → advanced.md → details.md 這樣多層引用，Claude 可能只用 `head -100` 預覽就不繼續深挖,導致資訊不完整。正確做法是所有參考檔都直接掛在 SKILL.md 底下同一層。

**4. 依「領域」而非「檔案序號」拆分參考檔**

例如一個查資料的 skill,依主題拆成 `reference/finance.md`、`reference/sales.md`,使用者問 sales 問題時 Claude 只讀 sales.md,finance.md 完全不佔 token——這是文檔中「Pattern 2: Domain-specific organization」的做法,比單純為了不超過 500 行而任意切檔更有效。

**5. 腳本用「執行」而非「讀取」來省 token**

如果邏輯可以寫成 deterministic 的 script（如 `validate_form.py`）,指示 Claude 用 bash 執行它、只把輸出（如 "Validation passed"）帶進 context,腳本本身的程式碼永遠不進 context window——這比讓 Claude 讀完腳本邏輯再自己生成等效程式碼省得多。

## 驗證方法

文檔建議先用「evaluation-driven development」而不是先寫文件：先讓 Claude 在沒有 skill 的情況下跑代表性任務、記錄哪裡失敗，再針對這些落差寫最精簡的內容,並用 Haiku / Sonnet / Opus 分別測試(Haiku 測試「指引夠不夠」,Opus 測試「有沒有過度解釋」）


---

# Crystal clear requirements

how to turn a grilling session into a clear spec the agent can build from, to deliver the results you actually want.

## 從「討論」到「Spec」再到「Agent 實作」

### 1. 整體工作流：Explore → Plan → Code → Commit

Claude Code 官方推薦的四階段流程：

- **Explore**：先讓 agent（或人）讀懂現有程式碼/系統脈絡，不急著寫方案
- **Plan**：切到 **Plan Mode**（`Shift+Tab` 切換），這階段只讀取不修改檔案，用來反覆釐清需求、討論邊界情況、產出書面計畫
- **Code**：計畫確認後才進入實作
- **Commit**：完成後整理提交

官方建議把心力配置在「**80% 時間規劃、20% 時間監督執行**」——這正是「grilling session」該發生的地方：把模糊需求在動手前榨乾。

### 2. 判斷 Spec 是否夠清楚的標準

出自 [Demystifying Evals for AI Agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)，這是最貼近「Crystal clear requirements」主題的官方文章。核心測試法：

> **「兩位互相不知情的領域專家，看了同一份 Spec，應該會得出同樣的 pass/fail 判定。」**

如果做不到這件事，代表 Spec 還不夠清楚，需要繼續追問。這篇文章也指出好的 Spec 要包含三要素：

1. **無歧義的任務描述**（instructions）
2. **明確的成功標準**（success criteria）
3. **參考解**（reference solution，用來驗證這個 Spec 真的可達成）

以及一條關鍵原則：「**Grader 會檢查的每一件事，都應該在任務描述裡寫清楚**」——換句話說，不要讓 agent 去猜驗收標準沒寫出來的部分。

### 3. 「約束交付物,而非規定實作路徑」

出自 [Building Effective AI Agents](https://www.anthropic.com/research/building-effective-agents)：

- 如果人（或 Spec 撰寫者）在事前就試圖規定所有技術細節,一旦其中有錯,錯誤會一路級聯到下游實作
- 更好的做法是：**清楚定義「要達成的成果」和「驗收條件」，把「怎麼做」的空間留給 agent 去探索**
- 建議用「功能展開」的方式描述需求，例如：「使用者可以開新聊天、輸入查詢、按 Enter、看到串流回應」——這是在描述行為與結果，不是在規定程式碼結構

### 4. Eval-driven：先寫測試案例，再讓 agent 動手

官方建議反直覺的順序：**先寫評估用例(比如寫出 20–50 個會出錯的案例)去逼出真正的需求，再讓 agent 去滿足這些用例**，而不是先寫一份看似完整的文字 Spec 就直接開工。這個過程本身就是「grilling」——寫 eval case 的過程會強迫你把模糊地帶具體化成可驗證的例子。

### 5. CLAUDE.md / Skill 描述的明確度指引

[Skill authoring best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices) 也給了可以套用在 Spec 撰寫上的具體技巧：

- 避免模糊詞彙（如 "helper"、"utils"、"適當處理"）
- 同時說明「做什麼」和「什麼情境下用」
- 測試方法：這份說明是否能讓不同能力等級的模型（Haiku/Sonnet/Opus）都得出一致的理解

### 綜合成一份可套用的 Spec 骨架

```markdown
## 背景 / 為什麼要做
## 使用者行為展開（功能描述，非技術細節）
## 驗收標準（Success Criteria）— 具體到「兩個獨立的人看了會做出同樣判斷」
## 範圍界定 / Non-goals（明確不做什麼）
## 已知邊界情況 / 約束
## 參考解或範例（如果有）
```

最核心的一句話可以總結官方立場：**Spec 的清晰度，不是用「寫了多少字」衡量，而是用「兩個不看你討論過程的人，能不能得出同樣的驗收結論」來衡量。** 這也正好是把「grilling session」轉成 spec 時該用來自我檢驗的判準。


---

# Decomposing complex work into session-sized chunks

how to break down big projects into phases your agent can execute one at a time, and how (and when) to create a new session.

## 「拆解大型專案為 Session 級任務」的建議

### 1. Context 視窗限制與管理機制

Context 會隨對話歷史、檔案讀取、工具輸出快速填滿,進而影響效能。官方提供的機制：

- **自動 Compaction**：接近 context 上限時系統自動摘要對話歷史，保留核心程式碼與決策
- `/clear`：重置 context，開啟全新獨立對話（前段對話仍可用 `/resume` 找回）
- `/compact [instructions]`：手動觸發摘要,可指定要保留的重點
- `/context`：檢視目前 context 消耗狀況
- `/rewind`：回溯到歷史訊息點

**官方建議何時開新 session**：(1) 要處理完全無關的任務時用 `/clear`；(2) 同一 session 中同一錯誤已被糾正兩次以上、context 已被汙染時，開新 session 並給更精準的初始提示。

### 2. Plan Mode + CLAUDE.md 做為階段拆解工具

官方工作流是 **Explore → Plan → Implement → Commit** 四階段：

- 先用 plan mode（`--permission-mode plan` 或 Shift+Tab）只讀探索,避免倉促下手
- 產出詳細計畫後才實作
- **CLAUDE.md** 會在每個 session 自動載入（透過 prompt caching,不額外耗費 token）,適合存放專案慣例、建置指令
- 跨多 session 的大型任務,官方建議把階段性計畫寫入 `PLAN.md` 等 markdown 檔,供後續 session 接續讀取

### 3. Subagent 作為 Context 隔離手段

- 每個 subagent 有**獨立、全新的 context**（不含父 session 對話歷史）
- subagent 完整工作過程只以摘要形式回傳父 session,保持主 context 精簡
- 適合用來隔離大量檔案讀取、測試輸出、或需要 fresh context 的驗證任務（如 code review）

若需要**真正跨 session 並行協作**,官方另提供 **Agent Teams**：多個獨立 session 共享任務清單並互相溝通,成本較高,適合需跨 session 協調的複雜任務（如多角度除錯、跨模組重構）。

### 4. 長時間執行、Checkpoint、跨 Session 交接

- **Checkpoint**：每次使用者提示自動建立檢查點,保留最新 100 個快照,可用 `/rewind` 選擇歷史點復原程式碼或對話
- **Task Budgets**（Claude API 層）：可對長時間 agentic loop 設定 token 預算（如 `task_budget: {total: 64000}`）,模型會看到預算計數並自我調節,預算將盡時主動摘要而非中斷；`remaining` 欄位可在 session 間傳遞預算狀態
- session 超過 100K token 且逾 1 小時未活動,恢復時會跳出「從摘要開始」的選項

### 5. 決定何時開新 Session 的判斷準則

| 情境                                | 建議                           |
| --------------------------------- | ---------------------------- |
| 任務完全無關、前一任務已完成                    | `/clear` 或新 session          |
| 同一錯誤已被糾正 2 次以上,context 已污染        | 新 session + 更精準提示            |
| 需要並行但獨立的工作（如多角度 review）           | Agent Team 或 Worktree        |
| 單一 session 逾 100K token + 1 小時未活躍 | 恢復時考慮從摘要開始                   |
| 大型單體專案需分包處理                       | 從子目錄啟動 Claude + 多層 CLAUDE.md |
| 需要檔案隔離的並行工作                       | Worktree（`--worktree`）       |

**核心原則**：以「**任務邊界的變化**」與「**context 內容的相關性**」而非單純時間長度或 token 用量來決定 session 邊界, 即使 context 還有餘量,無關的工作也應該分離開新 session。

---

# Handoffs

how to leave a breadcrumb trail for your agent, so you can both pick up exactly where you left off — whether it's the next hour, or next week.

讓 agent 之後（不論是一小時後還是一週後）能準確接續工作——是由好幾層機制搭配起來完成的。最容易理解的分法，是按「要接續的時間跨度」來看官方提供了什麼工具。

**第一層：同一台機器上，幾分鐘到幾小時內接續**

這是最基本的情況，靠的是 Claude Code 內建的 session 存檔機制。每個 session 結束後都會自動落地保存完整對話歷史、tool 呼叫紀錄、model 設定與最近讀過的檔案，不需要你手動處理：

```bash
claude --continue          # 接續最近一次的 session
claude --resume <name>     # 接續你指定名字的 session
```

官方建議在開始時就用 `claude -n <name>` 給 session 取一個有意義的名字（例如 `oauth-migration`），或用 `/rename` 中途改名。這個名字本身就是最直接的 breadcrumb——之後不管是你自己還是別人，看到名字就知道這條線在做什麼、該接到哪裡。

另外還有更細粒度的 `/rewind`（或按 Esc 兩下）checkpoint 機制：每次你送出 prompt，系統都會自動存一次快照，可以隨時回到過去任一個節點。它還有一個好用的選項——「Summarize from here」或「Summarize up to here」，讓 Claude 主動把當下的進度寫成一段摘要，等於是你請 Claude 自己留一份交接筆記。

**第二層：跨天接續，或是上下文快被壓縮掉時**

當一個 session 閒置超過一小時、累積 token 數又超過 100K，Claude Code 恢復時會跳出選擇：要「從摘要恢復」還是「恢復完整內容」。這其實反映了官方對 breadcrumb 的核心思路——與其把所有細節都留著，不如主動決定「哪些東西一定要留下來」。

這也是為什麼 CLAUDE.md 在這一層扮演關鍵角色：它每次啟動都會被自動載入，不會被壓縮或摘要掉。官方建議直接在裡面寫清楚壓縮時該保留什麼，例如：

```markdown
When compacting, always preserve:
- The full list of modified files
- Any test commands and their output
- Key architectural decisions
```

換句話說，CLAUDE.md 就是那份「不會隨時間流失」的跨 session 交接文件，負責放規則、workflow、和常踩的坑，而不是會過時的細節資訊。

**第三層：跨機器、跨更長時間的接續**

如果你的情境是換一台機器、或是要在完全不同的執行環境接續，Claude Agent SDK 提供了 `SessionStore` 這個介面，可以把 session 內容鏡像存到 S3、Redis 或 Postgres 之類的外部後端。寫入時是本地優先、再非同步同步到遠端；之後不管在哪台機器上，只要指定同一個 store 和 session ID 就能接回：

```typescript
for await (const message of query({
  prompt: "Continue where we left off",
  options: { sessionStore: store, resume: sessionId },
})) { ... }
```

這等於是把 breadcrumb trail 從「單一機器的生命週期」抽離出來，變成一份獨立存在的紀錄。

**另外兩個補充機制**

- **Cross-session messaging**：如果是多個 session 平行工作、互相需要交接的情況，一個 session 可以用 `SendMessage`（配合 `/list-agents` 找到目標、`@mention` 指定對象）直接傳一段純文字訊息給另一個 session，對方不需要吃下你完整的對話歷史，只收到你想告訴它的重點。
- **Skills**（`.claude/skills/*/SKILL.md`）：如果某個交接的內容其實是一套固定流程，官方建議乾脆把它寫成 Skill 並檢入 git。這樣「breadcrumb」就不只是文字紀錄，而是一段下次（甚至換人）都能直接執行 `/xxx` 跑一遍的可重複流程。

**整體來說**

短期用 session resume 加 checkpoint；中期靠 CLAUDE.md 主動指定要保留的重點；長期或跨機器則用 SessionStore 把記錄外部化；平行協作用 cross-session messaging 互相留言；而可重複的流程則直接固化成 Skill。
