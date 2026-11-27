# L2 程式執行使用獨立的 sandbox 服務，不掛載 docker.sock

worker 本身跑在 Docker 內，若要為每次 L2 執行動態啟動容器，就必須把 `docker.sock` 掛給 worker，等同把主機 root 權限交給一個依 LLM 輸出行事的程式。因此改為在 compose 中常駐一個獨立的 `sandbox` 服務：只接在 `internal: true` 的網路（無對外連線）、只掛載 workspace、以 HTTP 接收要執行的程式碼並限制 CPU / 記憶體 / 時間。worker 透過該服務執行程式，永遠不接觸 Docker daemon。

## Considered Options

- **worker 掛載 docker.sock，每次執行起新容器**：隔離最乾淨，但 worker 一旦被 prompt injection 誘導，就能控制整台主機。
- **v1 不做 L2**：最安全，但 Excel 分析等需求（F1）就無法實現。

## Consequences

- sandbox 是常駐容器，前後兩次執行之間可能殘留狀態（暫存檔、行程），每次執行後需清理，或定期重啟。
