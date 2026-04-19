# W04｜Linux 系統基礎：檔案系統、權限、程序與服務管理

## FHS 路徑表

| FHS 路徑 | FHS 定義 | Docker 用途 |
|---|---|---|
| /etc/docker/ | 系統級設定檔 | 存放 daemon.json 等服務設定 |
| /var/lib/docker/ | 程式變動持久資料 | 存放映像層、容器狀態、Volume 資料 |
| /usr/bin/docker | 使用者執行檔 | 存放 Docker CLI 命令列工具 |
| /run/docker.sock | 執行期暫存 (Socket) | Docker Daemon 與 CLI 通訊的 Unix Socket |

## Docker 系統資訊

- Storage Driver：overlayfs
- Docker Root Dir：/var/lib/docker
- 拉取映像前 /var/lib/docker/ 大小：236K
- 拉取映像後 /var/lib/docker/ 大小：240K

## 權限結構

### Docker Socket 權限解讀
![app](./images/dockersock.png)

### 使用者群組
![app](./images/defaultid.png)

### 安全意涵
Docker Group 成員等同於 Root 權限，是因為 Docker Daemon 本身是以 Root 身份運行，並擁有對 Host 作業系統的完全控制權。透過加入 `docker` 群組，使用者能直接存取 `/run/docker.sock`，進而指揮 Daemon 執行任何動作。我在實驗中透過 `docker run -v /etc/shadow:/host-shadow:ro` 掛載系統密碼檔並成功讀取，證明了這種權限設定在生產環境中極具風險，因為這允許使用者輕易突破主機與容器的邊界。

## 程序與服務管理

### systemctl status docker
![app](./images/status.png)

### journalctl 日誌分析
![app](./images/journal.png)
- 日誌分析摘要：
    - 觀察到的事件：記錄了 image pulled (拉取 nginx 映像檔)、網路介面建立 sbJoin (容器啟動)、以及 received task-delete (容器執行完畢後被刪除)。
    - 結論：日誌顯示 Docker Daemon 運作正常，且能正確響應 docker run --rm 指令的生命週期管理。當前使用的 Daemon PID 為 1198。

### CLI vs Daemon 差異
CLI 是使用者操作的「遙控器」（/usr/bin/docker），負責發送指令；Daemon 是在背景持續運行的「電視機本體」（dockerd），負責真正執行指令。`docker --version` 正常只能證明 CLI 程式檔存在，不能代表 Daemon 已啟動或能正常通訊，因此無法代表 Docker 功能正常。

## 環境變數

- $PATH：![app](./images/path.png)
- which docker：![app](./images/which.png)
- 容器內外環境變數差異觀察：![app](./images/matchinout.png)
- 簡述：容器內擁有獨立的環境空間，其變數定義與 Host 完全隔離，體現了容器的「隔離性」原則。

## 故障場景一：停止 Docker Daemon

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| systemctl status docker | active | inactive | active |
| docker --version | 正常 | 正常 | 正常 |
| docker ps | 正常 | Cannot connect | 正常 |
| ps aux grep dockerd | 有 process | 無 process | 有 process |

![app](./images/p25before.png)
![app](./images/p25now.png)
![app](./images/p25after.png)

## 故障場景二：破壞 Socket 權限

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| ls -la docker.sock 權限 | srw-rw---- | srw------- | srw-rw---- |
| docker ps（不加 sudo） | 正常 | permission denied | 正常 |
| sudo docker ps | 正常 | 正常 | 正常 |
| systemctl status docker | active | active | active |

![app](./images/p30before.png)
![app](./images/p30now.png)
![app](./images/p30after.png)

## 錯誤訊息比較

| 錯誤訊息 | 根因 | 診斷方向 |
|---|---|---|
| Cannot connect to the Docker daemon | Daemon 服務未啟動 | systemctl status docker |
| permission denied…docker.sock | Socket 權限不足 | ls -l /run/docker.sock 與 id |

說明：`Cannot connect` 代表 Daemon 根本不在線上，指令找不到通訊對象；`permission denied` 代表 Daemon 有在運作，但因為使用者沒在 `docker` 群組裡，被 Linux 的權限機制擋在了門外。

## 排錯紀錄
- 症狀：`docker ps` 噴出 `permission denied` 錯誤。
- 診斷：執行 `id` 確認當前使用者不在 `docker` 群組中，且 `ls -l /run/docker.sock` 確認該檔案權限需要 `docker` 群組成員身份。
- 修正：執行 `sudo usermod -aG docker $USER` 並執行 `exit` 重新登入以更新權限。
- 驗證：重新登入後執行 `docker ps`，成功列出容器資訊，且無需加 `sudo`。

## 設計決策
教學環境選擇使用 `usermod -aG docker` 是為了提高操作效率，減少重複輸入 `sudo` 的負擔。然而，這犧牲了「最小權限原則」，若使用者帳號被滲透，攻擊者能藉此取得 Host 的 Root 權限。在正式生產環境，應嚴格限制該群組成員，或優先採用 Rootless Docker 模式來強化資安防護。
