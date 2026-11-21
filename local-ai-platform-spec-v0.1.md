# 地端 AI 助理平台｜系統規劃書 v0.1


## 1. 目標與範圍

**目標**：基於 Ollama 提供之模型，搭配自建 Vue 3 前端與 FastAPI 後端，實作 ChatGPT / Claude 的網頁應用，並藉此學習地端模型的架設、串接與維運。

**三大功能**

1. 多模態對話
2. 開發筆記庫（DevNote）知識庫 + RAG
3. Cowork：可在指定工作資料夾內自主執行多步驟任務的 agent

**非目標（v1 不做）**：多人協作、商業化、行動 App、模型微調、雲端部署。

**設計原則**：單機優先；模型層可抽換（只依賴 Ollama HTTP API）；所有 agent 動作可追蹤、可中斷、可人工核准。

## 2. 模型與硬體策略

| 角色 | 用途 | 選型原則 |
|---|---|---|
| Chat 模型 | 對話、RAG 回答、agent 規劃 | 4B–12B 量化，需支援 tool calling |
| Vision 模型 | 圖片理解 | 可與 Chat 同一模型（若支援圖片） |
| Embedding 模型 | RAG 向量化 | 小型專用模型 |

- 24GB 統一記憶體，GPU 可用約 16GB 上下。模型加 KV cache 建議控制在 10–12GB 內。
- 同時載入多個模型會吃記憶體。Ollama 會依 `OLLAMA_KEEP_ALIVE` 卸載閒置模型，Chat 與 Embedding 切換會有載入延遲，需實測調整。
- 具體型號（Gemma 世代、是否支援 vision / tools、embedding 維度）待 P0 實測後定案，寫入設定檔，不寫死在程式。
- Ollama 裝在 macOS 本機（才能用 Metal GPU），其餘服務走 Docker。

## 3. 整體架構

```
[Vue 3 SPA] --HTTP/SSE--> [FastAPI api] --HTTP--> [Ollama @ macOS host]
                              |   |
                              |   +--enqueue--> [Redis] --> [Celery worker]
                              |                              (ingest / agent)
                              +--> [PostgreSQL + pgvector]
                                   workspace/ (掛載給 worker 的工作資料夾)
```

容器內以 `OLLAMA_BASE_URL=http://host.docker.internal:11434` 連到本機 Ollama。

## 4. 功能分析與可行性

### F1 多模態對話（可行性：高）

- **輸入**：文字、圖片（貼上 / 拖曳 / 上傳）、文字類檔案（txt、md、程式碼、PDF 先轉文字）。
- **輸出**：地端模型原生只輸出文字，v1 定義的「多模態輸出」為 Markdown 富文字、程式碼區塊、表格、Mermaid 圖、可下載檔案（由 Cowork 產生）。
- **選配（後期）**：圖片生成與語音（STT/TTS）需另外的模型，與 Chat 模型同時跑會吃光 24GB，列為選配，不放進 v1。
- 對話體驗：串流、停止、重新生成、編輯後重送、自動產生標題。

### F2 DevNote 知識庫 + RAG（可行性：高）

> 注意：目前沒能讀到你的 DevNote 倉庫內容（抓取被限流）。以下假設筆記是 Markdown 檔案放在 Git 倉庫，請補充實際目錄結構與格式（有無 front matter、圖片、是否含中文）後再調整切塊策略。

管線：

1. **取得來源**：`git clone / pull` 到本機資料夾（唯讀）。
2. **解析**：讀取 `.md`，解析 front matter（title、tags）、標題階層。
3. **切塊**：依 H1–H3 標題切，每塊約 500–800 tokens、小幅重疊；程式碼區塊不可被切斷；每塊附 metadata（檔案路徑、標題路徑、tags、commit、mtime）。
4. **向量化**：呼叫 Ollama embedding，存入 pgvector。
5. **檢索**：混合檢索——向量 top 20 + 全文檢索 top 20，以 RRF 合併，取前 5–6 塊（可選：再用 chat 模型重排）。
6. **生成**：把片段以 `[1] [2]` 編號放入 prompt，要求回答引用編號；後端回傳 citations 事件，前端顯示來源晶片，可點開看原文。
7. **增量同步**：以檔案內容 hash 判斷新增 / 修改 / 刪除，只重算變動檔案。

注意：PostgreSQL 內建全文檢索對中文分詞很弱。若筆記大量中文，可用 `pg_bigm` / `zhparser`，或 v1 先只靠向量檢索加關鍵字 `ILIKE` 備援。

### F3 Cowork（可行性：中，需分級達成）

定義：使用者指定一個工作資料夾與目標，agent 自行規劃、呼叫工具、產出檔案，過程可見、可中斷、危險動作需核准。

最大風險是 **4B–12B 小模型的 tool calling 與長流程穩定度**，不如雲端大模型。因此採分級：

| 等級 | 能力 | 工具 | 核准 |
|---|---|---|---|
| L0 | 唯讀研究：搜尋筆記、讀檔、整理摘要 | `search_notes`、`list_files`、`read_file` | 不需要 |
| L1 | 寫入工作資料夾：產出報告、改檔 | `write_file`、`edit_file` | 預設需核准，可設白名單 |
| L2 | 執行程式 | `run_python` / `shell`（Docker sandbox，無網路，限 CPU / 記憶體 / 時間） | 一律需核准 |

Agent 迴圈：規劃 → 選工具 → 執行 → 觀察 → 重複，直到完成或達到最大步數。

可靠度設計（也是你的 agent reliability 方向的練習場）：

- 每步有逾時、重試（退避）、最大步數與總時間上限。
- 工具輸入用 Pydantic 驗證，格式錯誤時回饋給模型重試有限次數。
- 所有步驟寫入 `agent_steps`，可重播、可除錯。
- 任務可取消，取消時中止對 Ollama 的請求與 sandbox。
- 路徑一律限制在 workspace 內（防 path traversal 與 symlink 逃逸）。
- 筆記或檔案內容視為「資料」，不得觸發 L1 以上動作（防 prompt injection）。

## 5. 後端規格（FastAPI）

**技術**：Python 3.12、FastAPI、SQLAlchemy 2.0（async）、Alembic、httpx、Celery + Redis、Pydantic v2。

### 模組結構

```
api/app/
  main.py  core/(config, logging)
  routers/(chat, conversations, uploads, models, kb, tasks)
  services/
    ollama_client.py   # 唯一與 Ollama 溝通的地方
    chat_service.py    # prompt 組裝、context 截斷
    rag/(ingest, chunker, retriever)
    agent/(loop, tools, sandbox, policy)
  db/(models, session)   schemas/   workers/(celery_app, tasks)
```

### API 清單

| Method | Path | 說明 |
|---|---|---|
| POST | /api/chat/stream | 送出訊息，SSE 串流回應 |
| GET / POST | /api/conversations | 列表 / 建立 |
| GET / PATCH / DELETE | /api/conversations/{id} | 讀取 / 改名 / 刪除 |
| GET | /api/conversations/{id}/messages | 訊息列表 |
| POST | /api/uploads | 上傳圖片 / 檔案 |
| GET | /api/models | Ollama 模型清單 |
| POST | /api/kb/sources | 新增知識來源（git / 資料夾） |
| POST | /api/kb/sources/{id}/sync | 觸發同步（背景工作） |
| GET | /api/kb/search?q= | 檢索除錯 |
| POST | /api/tasks | 建立 Cowork 任務 |
| GET | /api/tasks/{id}/events | SSE 任務事件 |
| POST | /api/tasks/{id}/steps/{n}/approve | 核准 / 拒絕動作 |
| POST | /api/tasks/{id}/cancel | 取消任務 |

### 串流事件（SSE）

- 對話：`token`、`citation`、`title`、`message_done`、`error`
- 任務：`plan`、`step_start`、`tool_call`、`tool_result`、`approval_required`、`artifact`、`done`、`error`

### 對話流程

1. 存 user 訊息
2. （若開啟知識庫）檢索
3. 組 prompt：system + 歷史 + 檢索片段，依 token 預算截斷最舊訊息
4. 串流呼叫 Ollama
5. 轉發 SSE
6. 完成後存 assistant 訊息與耗時、tokens/s

客戶端中斷時，後端必須取消對 Ollama 的請求，避免白佔 GPU。

### 併發與資源

- 單機 GPU 只適合低併發：以 semaphore 限制同時推論數（預設 1–2），其餘排隊並回報排隊狀態。
- 設定集中管理：模型名稱、`num_ctx`、temperature、檢索參數、`KEEP_ALIVE`。

## 6. 前端規格（Vue 3）

**技術**：Vue 3 + Vite + TypeScript、Pinia、Vue Router、Tailwind（或你的設計系統）、markdown-it + Shiki（程式碼高亮）、Mermaid。

串流：因為是 POST，用 `fetch` + `ReadableStream` 自行解析 SSE（`EventSource` 只支援 GET）。

### 頁面

| 路由 | 內容 |
|---|---|
| /chat/:id? | 對話主畫面（側邊欄歷史 + 訊息區 + 輸入區） |
| /knowledge | 知識來源管理、同步狀態、檢索測試 |
| /tasks/:id | Cowork 任務：目標、步驟時間軸、核准、產出檔案 |
| /settings | 模型、參數、system prompt、主題 |

### 主要元件

`ConversationList`、`MessageList`、`MessageItem`（Markdown 渲染）、`Composer`（多行輸入、圖片預覽、拖曳 / 貼上、送出 / 停止）、`CitationChip`、`ModelPicker`、`TaskTimeline`、`ApprovalDialog`、`FileTree`、`ArtifactViewer`。

### 狀態（Pinia）

`conversationStore`、`chatStore`（串流中狀態、abort controller）、`kbStore`、`taskStore`、`settingsStore`。

### UX 要點

- 串流中逐 token 渲染但避免整段重繪；自動捲動可被使用者打斷。
- 顯示模型載入中、排隊中、出錯的狀態。
- 引用來源以晶片顯示，Cowork 每一步可展開看輸入與輸出。

## 7. 資料庫規格（PostgreSQL 16 + pgvector）

使用 Alembic 管理遷移。單人使用，但各主表預留可為空的 `user_id`，方便日後擴充。

| 表 | 主要欄位 |
|---|---|
| conversations | id (uuid), title, model, system_prompt, created_at, updated_at |
| messages | id, conversation_id (FK), role, content, status (streaming / complete / error / cancelled), model, meta (jsonb：citations、tokens/s、耗時), created_at |
| attachments | id, message_id (FK), kind (image / file), path, mime, size, sha256, created_at |
| kb_sources | id, name, type (git / folder), uri, branch, last_commit, last_synced_at, status, embedding_model, config (jsonb) |
| kb_documents | id, source_id (FK), path, sha256, title, tags (text[]), mtime, status |
| kb_chunks | id, document_id (FK), chunk_index, heading_path, content, token_count, embedding (vector(N)), tsv (tsvector) |
| agent_tasks | id, conversation_id (nullable), goal, workspace_path, status, model, error, created_at, finished_at |
| agent_steps | id, task_id (FK), step_no, type (plan / tool_call / observation / final), tool_name, input (jsonb), output (jsonb), status (pending / approved / rejected / running / ok / failed), duration_ms, created_at |

**索引**

- `messages (conversation_id, created_at)`
- `kb_documents` 唯一 `(source_id, path)`
- `kb_chunks.embedding` 使用 HNSW（cosine），`tsv` 使用 GIN
- `agent_steps (task_id, step_no)`

**注意**：向量維度 N 取決於 embedding 模型，更換模型需全部重算，因此在 `kb_sources` 記錄 `embedding_model`。圖片檔存在 Volume，資料庫只存路徑。

## 8. 專案結構與部署

```
local-ai-platform/
  docker-compose.yml
  api/    (app/, alembic/, tests/)
  web/    (src/)
  workspace/      # Cowork 工作資料夾（掛載）
  data/uploads/   # 使用者上傳
  data/kb/        # 知識庫 git 副本
```

Compose 服務：`db`（pgvector 映像）、`redis`、`api`、`worker`；前端開發時直接在本機 `npm run dev`。Ollama 不放進 Docker（Mac 容器用不到 GPU）。

## 9. 開發路線（時間為粗估）

| 階段 | 內容 | 驗收 |
|---|---|---|
| P0（約 1 週） | 安裝 Ollama，實測 4B–12B 模型的記憶體、速度、vision、tool calling、embedding | 決定模型清單與 `num_ctx` |
| P1（2–3 週） | FastAPI 串流 + 前端聊天 + 對話存 DB | 能穩定多輪對話、可停止、可載入歷史 |
| P2（1–2 週） | 圖片 / 檔案輸入、附件儲存 | 貼圖提問可得到合理回答 |
| P3（2–3 週） | 知識庫同步、切塊、混合檢索、引用顯示 | 針對筆記提問，回答附正確來源 |
| P4（2–3 週） | Cowork L0：唯讀 agent、步驟時間軸 | 能完成「整理某主題筆記成摘要」 |
| P5（3 週以上） | Cowork L1 / L2：寫檔、核准流程、sandbox | 產出檔案並可核准或拒絕每個動作 |

## 10. 風險與待決事項

1. **小模型 agent 穩定度**：可能需要更嚴格的工具 schema、更短的步驟與更多重試；必要時 Cowork 只做 L0–L1。
2. **記憶體壓力**：RAG 片段與長歷史會放大 KV cache，需限制 `num_ctx` 與檢索片段數。
3. **DevNote 結構未知**：需要你確認目錄、格式、語言，才能定案切塊與全文檢索方案。
4. **安全**：sandbox 隔離、workspace 路徑限制、核准機制，任何一項偷懶都可能讓 agent 動到不該動的檔案。
5. **模型選型**：Gemma 新世代的尺寸、vision 與 tool calling 支援度，以 P0 實測為準。

### 下一步

- 補充 DevNote 倉庫的目錄與檔案格式。
- 完成 P0 實測，回填第 2 節的模型清單。
- 之後可再細化：API 請求 / 回應 schema、資料表 DDL、Cowork 工具定義與 prompt。
