# 對話與 Cowork 任務合併為同一個實體

v0.1 規劃將 Cowork 任務（`agent_tasks`）設計成與對話平行的獨立實體，參照 Claude 把 Cowork 與聊天分開的做法。但任務實際上都是透過對話中的指令開始的，所以改為只保留 Conversation：每則 user 訊息觸發一個 Run，一般聊天是沒有工具的 Run，Cowork 則是綁定 Workspace、可呼叫工具的對話。這樣就少一個實體，也只需要一套事件流與前端時間軸。

## Consequences

- Workspace 與 Permission Ceiling 綁定在對話上，不再綁定在任務上。
- 有工具的 Run 可能長時間執行或停下來等待核准，所以 Run 不能依附在單一 HTTP 請求的生命週期上。
- 「重新生成」只會取代訊息，已經寫進 Workspace 的檔案不會因此還原。
