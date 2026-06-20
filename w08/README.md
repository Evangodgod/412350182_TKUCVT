# W08｜容器生產實踐

## Healthcheck 故障測試
- **停 db 後幾秒被標 unhealthy**：約 30 秒（依據 `compose.yaml` 設定之 `interval: 10s`，並在連續失敗 `retries: 3` 次後，核心正式將其標記為不健康）。
- **對應的 log 訊息**：
  * **Docker Engine 層級取證**（透過 `docker inspect` 檢視）：
    `"Status": "unhealthy", "FailingStreak": 3, "Output": "pg_isready - U postgres -d student_db -> no response / connection refused"`
  * **App 容器層級日誌**（透過 `docker compose logs app` 檢視）：
    `psycopg2.OperationalError: could not connect to server: Connection refused`

## Log 失控估算
- **noisy 容器 30s log 大小**：約 1.5 MB（在高頻率惡意請求或無窮迴圈噴吐日誌狀態下）。
- **預估 24h 大小**：約 4.32 GB（計算公式：$1.5 \text{ MB} \times 2 \times 60 \times 24 = 4320 \text{ MB}$）。
- **套 rotation 後穩定上限**：最高限制為 **40 MB**（依據配置 `max-size: "10m"` 與 `max-file: "3"`，單一容器最多保留 3 個 10M 歷史檔案，加上當前正在寫入的 1 個活動檔案 10M，不論運作幾年，硬碟空間皆會被壓制在 40M 以下，完全免疫日誌型 DoS 攻擊）。

## 資源限制實驗
| 實驗 | 命令 | 觀察結果 | 對應 cgroup 檔 | 值 |
|---|---|---|---|---|
| **OOM** | `stress-ng --vm 1 --vm-bytes 200m` | 容器遭核心強制擊殺，退出碼 `exit 137`, `OOMKilled=true` | `memory.max` | `134217728` (128M基準) 或 `268435456` (256M實測值) |
| **CPU throttle** | `stress-ng --cpu 4` | 透過 `docker stats` 觀察 CPU 使用率被死死壓制在 `≈ 50%` | `cpu.max` | `50000 100000` |

## 權限四階對照
| 階梯 | 安全配置狀態 | id (UID/GID) | CapEff (實效核心特權位元) | NoNewPrivs (阻斷提權) | curl /healthz (服務狀態) |
|:---:|---|---|---|:---:|:---:|
| **0** | **特權容器 (Privileged)** | `uid=0(root) gid=0(root)` | `ffffffffffffffff` (解鎖全部特權) | `0` (false) | 200 OK |
| **1** | **標準 Root 容器** | `uid=0(root) gid=0(root)` | `00000000a80425fb` (預設裁剪特權) | `0` (false) | 200 OK |
| **2** | **非特權使用者 (appuser)** | `uid=1000(appuser) gid=1000` | `0000000000000000` (無核心特權) | `0` (false) | 200 OK |
| **3** | **非特權 + 阻斷提權** | `uid=1000(appuser) gid=1000` | `0000000000000000` | `1` (true) | 200 OK |
| **4** | **縱深防禦最高規格 (Cap Drop + Read-Only)** | `uid=1000(appuser) gid=1000` | `0000000000000000` (全面剥奪) | `1` (true) | 200 OK |

## 排錯紀錄

### 🔍 故障演練 1：資料庫認證損毀 (F1)
- **症狀**：網頁端發送請求時，客戶端噴出 `curl: (52) Empty reply from server` 或是 `HTTP 500/503`。執行 `docker compose logs app` 高階取證，日誌爆出致命錯誤：`psycopg2.OperationalError: password authentication failed for user "postgres"`。
- **診斷**：Flask 網頁框架在啟動並呼叫根路由時，會透過驅動向 PostgreSQL 進行身份驗證。當 `.env` 傳遞之環境變數密碼與資料庫實體內部密碼不對稱時，傳輸層連線會被直接拒絕對稱，導致容器行程崩潰。
- **修正**：重新編輯 `.env` 檔案恢復正確密碼，並執行 `docker compose down -v` 徹底抹除因錯誤密碼殘留的舊具名磁碟區（Volume），確保資料庫全新對齊正確變數。
- **驗證**：重新執行 `docker compose up -d` 重新初始化，發動 `curl -i http://localhost:8080/` 成功回歸 `HTTP/1.1 200 OK` 狀態並吐出正確資料。

### 🔍 故障演練 2：主機連接埠衝突 (F2)
- **症狀**：嘗試拉起環境時，終端機拋出大範圍紅色嚴重報錯：`Bind for 0.0.0.0:8080 failed: port is already allocated`，`app` 容器宣告建立失敗。
- **診斷**：根據 TCP 傳輸層原理，同一個作業系統 IP 介面上的同一個 Port 在同一時間只能被單一 Socket 進行 `Bind` 監聽。當 Host 主機本地端已有程序（如 Python 測試伺服器）捷足先登佔用了 `8080` 埠後，Docker Daemon 便無法代表容器向核心申請該埠口的流量映射轉發。
- **修正**：在 Host 本地端執行排錯指令 `sudo ss -lntp | grep :8080` 精準揪出佔用該埠口的實體行程 PID，並使用 `sudo kill -9 <PID>` 強制超渡、關閉該本地衝突程序。
- **驗證**：重新執行 `docker compose up -d`，容器瞬間復活綠燈放行，經 `curl` 驗證順利吐出學號網頁。

## 設計決策

### 1. 關於資源限制 (`mem_limit` / `cpus`) 的配置理由
- **`mem_limit: 256m`**：本專案為輕量級的 Python Flask 微服務與 PostgreSQL。Flask 服務在正常運作下記憶體僅消耗約 30M~50M，配置 256M 既能保留充足的記憶體緩衝區（Buffer）以應對突發性的流量，又能建立鋼鐵邊界。當容器內代碼發生記憶體洩漏（Memory Leak）或遭受 Fork Bomb 攻擊時，Linux 核心 cgroup 會在達到 256M 瞬間觸發 OOM Killer 將其擊殺，防止單一容器無限制榨乾 Host 端整台實體主機的記憶體。
- **`cpus: "0.5"`**：限制該容器在每 100ms (100000 μs) 的 cgroup 週期內，最多只能分配到 50000 μs 的 CPU 時間（即精確的 0.5 核）。這能有效實施多租戶資源審計，防止容器因無窮迴圈（Infinite Loop）異常或遭駭客控制進行惡意挖礦時，100% 啃食 Host 的所有 CPU 算力，確保同台主機上其他核心微服務的絕對可用性。

### 2. 關於 `read_only: true` 之後補上的 `tmpfs` 決策與理由
- **補上的配置**：在 `compose.yaml` 中針對 app 服務補上了 `tmpfs: /tmp` 記憶體暫存區。
- **理由**：生產加固啟用了安全規格極高的 `read_only: true`，這會直接將容器的根檔案系統（RootFS）完全鎖定為唯讀模式，阻斷任何駭客入侵後試圖寫入惡意後門、木馬或修改核心原始碼的攻擊路徑。然而，Python 運行環境與 Flask 框架在執行過程中，仍需要一個可寫入的地方來處理 `.pyc` 字節碼快取、客戶端上傳暫存或系統暫存檔（如 `PID` 檔）。為了兼顧「極致安全」與「應用程式正常運作」，我們開闢了 `/tmp` 並將其掛載為 `tmpfs`（直接使用主機實體記憶體）。這樣既滿足了 Flask 的寫入需求，又確保了所有暫存檔案在容器重啟後便會自動灰飛煙滅，不留任何安全隱患。

