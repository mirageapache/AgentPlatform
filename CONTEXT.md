# 地端 AI 助理平台

單機、單人使用的 AI 助理：以地端模型提供對話、知識庫檢索，以及能在指定資料夾內動手做事的 agent。

## Language

### 知識庫

**Knowledge Base（知識庫）**:
所有 Knowledge Source 的集合，是對話時「搜尋筆記」的範圍。
_Avoid_: DevNote（那只是其中一個 Source）、KB（僅限程式內縮寫）

**Knowledge Source（知識來源）**:
一個被同步進知識庫的筆記來源，例如一個 Git 倉庫或本機資料夾；各自有排除清單。DevNote 是第一個 Source。
_Avoid_: 倉庫、資料源

**Category（分類）**:
筆記在其 Source 內的資料夾路徑，用來分組與過濾；取代 tags。
_Avoid_: tag、標籤

**Attachment Excerpt（附件片段）**:
對話中上傳的長文件被切出的片段，只屬於該對話、只在該對話內檢索，不屬於知識庫。
_Avoid_: 知識庫片段、kb chunk

**Auto Retrieval（自動檢索）**:
未綁定 Workspace 的對話中，系統在每個 Run 開始前自動檢索知識庫並把片段放進 prompt。綁定 Workspace 的對話不做自動檢索，改由模型透過工具自行查詢。
_Avoid_: RAG（泛稱，不特指這個機制）

**Embedding Model（嵌入模型）**:
全系統唯一的向量化模型；更換時知識庫與所有附件片段一律重算。

### 對話

**Conversation（對話）**:
使用者與助理之間的一串訊息。一般聊天與 agent 工作都是對話；差別只在對話是否綁定 Workspace，以及綁定時的 Permission Ceiling。
_Avoid_: 任務、Task、Cowork 任務

**Run（執行）**:
為了回應一則 user 訊息而產生一則 assistant 回覆的完整過程，由一或多個 Step 組成。一般聊天就是只有一個 Step、沒有工具的 Run。
_Avoid_: 任務、job

**Step（步驟）**:
Run 中的一個單位：一次模型呼叫，或一次工具呼叫（含其 Approval 與觀察結果）。

**Cowork**:
功能名稱，指「綁定 Workspace 的對話」，不是一個獨立的實體。

**Superseded Message（被取代的訊息）**:
因「重新生成」或「編輯後重送」而失效的訊息；保留紀錄但不再屬於對話的有效歷史。
_Avoid_: 已刪除訊息

### Agent 與安全

**Workspace（工作區）**:
一個具名、可被多個對話重複使用的資料夾，是 agent 唯一能讀寫檔案的範圍。
_Avoid_: 工作資料夾、workspace_path

**Artifact（產出檔案）**:
某個 Run 在 Workspace 中新增或修改的檔案。未綁定 Workspace 的對話不會產生 Artifact。
_Avoid_: 輸出檔、可下載檔案

**Tool Level（工具等級）**:
工具依危險程度分為 L0（唯讀）、L1（寫入 Workspace）、L2（執行程式）。等級屬於工具本身，決定是否需要核准。

**Permission Ceiling（權限上限）**:
一個對話被允許使用的最高 Tool Level；超過上限的工具不會提供給模型。
_Avoid_: 任務等級

**Approval（核准）**:
使用者對單一工具呼叫的放行或拒絕。拒絕不會終止工作，而是連同理由回饋給模型。

**Allowlist Rule（白名單規則）**:
「工具 + Workspace 內路徑 pattern」的組合，符合者免核准；L2 工具永遠不能列入。
_Avoid_: 白名單（單獨使用時語意不明）
