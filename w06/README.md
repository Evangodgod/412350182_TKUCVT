# W06｜Docker Image 與 Dockerfile

## 映像組成
- Layers 是什麼：（用自己的話寫）
- Config 是什麼：（用自己的話寫）
- Manifest 是什麼：（用自己的話寫）

## python:3.12-slim inspect 摘錄
- Config.Cmd：
```json
      "python3"
```
- Config.Env：
```json
    "PATH=/usr/local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
    "LANG=C.UTF-8",
    "GPG_KEY=7169605F62C751356D054A26A821E680E5FA6305",
    "PYTHON_VERSION=3.12.13",
    "PYTHON_SHA256=c08bc65a81971c1dd5783182826503369466c7e67374d1646519adf05207b684"
```
- Config.WorkingDir：/app
- RootFS.Layers 數量：4 層

## Layer 快取實驗
| 情境 | build 時間 |
|---|---|
| v1 首次 build | 16.746s |
| v1 改 app.py 後 rebuild | 16.009s |
| v2 首次 build | 18.200s |
| v2 改 app.py 後 rebuild | 3.301s |

觀察（用自己的話寫）：為什麼 v2 的 rebuild 這麼快？

## CMD vs ENTRYPOINT 實驗

### 實測結果對照表

| 寫法 | `docker run <img>` 輸出 | `docker run <img> extra1 extra2` 輸出 |
|---|---|---|
| **CMD shell form**<br>`argtest:shell` | `argv = ['show_args.py', 'default1', 'default2']`<br>**`PID = 7`** | **直接被忽略/失效**<br>（無任何 show_args.py 輸出） |
| **CMD exec form**<br>`argtest:exec` | `argv = ['show_args.py', 'default1', 'default2']`<br>**`PID = 1`** | **系統崩潰報錯**：<br>`exec: "extra1": executable file not found in $PATH` |
| **ENTRYPOINT + CMD**<br>`argtest:entry` | `argv = ['show_args.py', 'default1', 'default2']`<br>**`PID = 1`** | **完美附加參數**：<br>`argv = ['show_args.py', 'extra1', 'extra2']`<br>**`PID = 1`** |

---

### 💡 核心結論與底層原理深入解讀

#### 1. 為什麼 `argtest:shell` 的 PID 是 7，而其他兩個是 1？
* **Shell Form (`CMD python ...`)**：Docker 底層其實是暗中啟動了一個 Shell 終端機，實際執行的命令是 `/bin/sh -c "python show_args.py ..."`。此時 **PID 1 是 `/bin/sh`**，而 Python 程式只是 sh 衍生出來的子進程（所以 PID 變成了 7）。
* **Exec Form (`CMD ["python", ...]`)**：Docker 不會透過 shell，而是直接呼叫內核執行 `python`。因此 **Python 程式本身就是 PID 1**。
* **為什麼生產環境（Production）一定要 PID 1？** 因為當我執行 `docker stop` 時，Docker 只會把關閉訊號（SIGTERM）發給 PID 1。如果是 Shell Form，`/bin/sh` 會直接無視這個訊號，導致應用程式無法優雅地關閉（Graceful Shutdown），每次都要硬生生卡 10 秒被 Docker 暴力強制殺死（SIGKILL）。



#### 2. 為什麼帶參數時，`argtest:shell` 會完全失效？
當輸入 `docker run argtest:shell extra1 extra2` 時，後面的 `extra1 extra2` 會把整個 `CMD` 覆蓋掉。Docker 嘗試去執行一條叫做 `extra1` 的 Linux 指令，但因為 Shell Form 內部的包裝混亂，導致參數直接被吞掉，完全沒有觸發 `show_args.py`。

#### 3. 為什麼帶參數時，`argtest:exec` 會噴出 `executable file not found` 崩潰？
因為 Exec Form 的 `CMD` 是**「整條覆蓋」**。當你後面接了 `extra1 extra2`，原本的 `["python", "show_args.py", ...]` 就徹底消失了，Docker 會直接把 `extra1` 當作要執行的實體主程式。因為你的 Linux 系統中根本沒有一個叫 `extra1` 的軟體，所以直接噴出找不到執行檔的 OCI 運行錯誤。

#### 4. 為什麼 `ENTRYPOINT + CMD` 是黃金組合？
在 `argtest:entry` 中，`ENTRYPOINT` 固定了主程式是 `python show_args.py`，而 `CMD` 裡面的 `default1 default2` 僅僅是「預設參數」。
* 當不帶參數時，它組合出：`python show_args.py default1 default2`。
* 當帶入 `extra1 extra2` 時，**它只會精準覆蓋掉 CMD 那段預設參數**，完美組合出：`python show_args.py extra1 extra2`！

這就是為什麼在開發專業的 Docker 映像檔時，**必定會使用 Exec Form 的 `ENTRYPOINT` 定義核心程式，再用 `CMD` 定義預設參數**，這樣容器才能兼具「安全（PID 1）」與「高度靈活的參數外接彈性」！

## Multi-stage 大小對照

| Image | SIZE |
|---|---|
| python:3.12（builder base） | 1.02 GB |
| python:3.12-slim（runtime base） | 43.2 MB |
| myapp:v2（單階段） | 197 MB |
| myapp:multi（多階段） | 184 MB |

### 解釋：builder stage 的 layer 去哪了？

在 `Dockerfile.multi` 中，我們用了兩個 `FROM` 指令，這在 Docker 裡代表兩個完全獨立的「平行宇宙（沙盒階段）」：

1. **第一階段（builder）**：我們用了高達 1.02 GB 的 `python:3.12` 重型 Image，裡面塞滿了 gcc 編譯器、Header 開發工具。在這裡跑 `pip install` 時，產生的所有編譯垃圾、下載暫存快取，通通都**固化在 builder 的這些 Layer 裡面**。
2. **第二階段（runtime）**：切換到只有 43.2 MB 的 `python:3.12-slim` 乾淨空屋。接著透過 `COPY --from=builder` 指令，像搬家工人一樣，**只把在第一階段真正編譯成功的「純淨 Flask 套件成品」給單獨搬過來**。

**結論：**
builder stage 的那些重型 Layer，在建置（`docker build`）結束後，**完全沒有被打包進最終的 `myapp:multi` 映像檔中**。它們會被留在 Docker 本機的快取資料庫裡（變成匿名快取碎片），這就是為什麼最終的產物可以從大胖子直接暴瘦成只有 184 MB 的精實規格


### .dockerignore 實驗數據對照表

| 實驗階段 | Build Context 傳輸大小 | 建置總耗時 (real) | 狀態與現象說明 |
|---|---|---|---|
| **du -sh .** | 150 MB | 512 KB | **加入忽略檔前**：專案目錄高達 150MB（因為塞了巨型垃圾檔）。<br>**加入忽略檔後**：專案目錄本體不變，但 Docker 成功過濾垃圾。 |
| **build context 傳輸大小** | **157.33 MB** | **457 B** | **加入忽略檔前**：垃圾檔被迫一起打包上傳，容量大爆炸。<br>**加入忽略檔後**：垃圾被完美阻擋，傳輸量降到剩下不到 1KB！ |
| **build 時間** | **0m33.002s** | **0m23.569s** | **加入忽略檔前**：因為要花時間壓縮與上傳 157MB 垃圾，明顯變慢。<br>**加入忽略檔後**：省去無效檔案的傳輸時間，建置速度大幅提升！ |
---

### 💡 核心結論與底層原理深入解讀

#### 1. 什麼是 Build Context？為什麼沒寫忽略檔會變慢？
執行 `docker build` 時，Docker 的架構其實是分為客戶端（Docker CLI）與伺服器端（Docker Daemon 引擎）。
當指令啟動時，第一件事就是把當前目錄下的「所有檔案」原封不動地打包，並網路上傳給 Docker Daemon（這個打包上傳的資料包就叫 **Build Context**）。

在步驟 2 中，因為寫了 `COPY . .` 且沒有配置過濾規則，導致 150MB 的無效大檔案硬生生地被塞進資料包中。即使這個垃圾檔案在程式執行時完全用不到，客戶端依然得乖乖花費大量的 CPU 與時間去壓縮、傳輸這 157 MB 的資料，嚴重拖慢了整體的建置效率。



#### 2. `.dockerignore` 的防護罩機制
在專案根目錄建立了 `.dockerignore` 檔案，並在裡面宣告了 `dummy_garbage.bin`、`.git`、`*.log`。
這張過濾網的運作順序是在**最一開始、打包 Build Context 之前**就生效。Docker CLI 在掃描目錄時，一看到符合規則的檔案，就會直接跳過、不將其納入打包清單中。

這就是為什麼步驟 4 的傳輸容量會從 157.33 MB 直接暴跌剩下 **457 B** 的驚人差距,這項技術在實際企業開發中極其重要，能完美避免如 `node_modules/`、`.git/` 歷史紀錄或本地測試日誌等垃圾檔案弄髒映像檔，是保持 Docker 映像檔精實與高速建置的必備好習慣

## 排錯紀錄
- 症狀：
- 診斷：
- 修正：
- 驗證：

## 設計決策
（說明本週至少 1 個技術選擇與取捨，例如：為什麼 runtime 選 `python:3.12-slim` 而不是 `alpine`？）
