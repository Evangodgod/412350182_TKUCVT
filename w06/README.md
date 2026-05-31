# W06｜Docker Image 與 Dockerfile

## 映像組成

- **Layers 是什麼**：
  就像是一片一片疊起來的**唯讀樂高積木**。在 Dockerfile 裡每執行一個指令（如 `RUN`、`COPY`），Docker 就會把產生的檔案變更打包成一層 Layer。這些 Layer 是唯讀且不可變的，底層一模一樣的積木可以被不同的映像檔同時共享，省下重複下載和佔用硬碟的時間。

- **Config 是什麼**：
  就是這個映像檔的**設定說明書**。它紀錄了這個映像檔的元數據（Metadata），包含環境變數（Env）、預設要執行的命令（Cmd/Entrypoint）、工作目錄（Workdir）、開出的通訊埠（Expose），以及這顆映像檔是由哪幾層 Layers 依序疊起來的歷史紀錄。

- **Manifest 是什麼**：
  就是映像檔的**清單索引（或者是出貨清單）**。當我們執行 `docker pull` 時，Docker 會最先下載這個 Manifest。它裡面明確記載了這款映像檔支援哪些 CPU 架構（例如 amd64、arm64），並指引 Docker 去哪裡把對應的 Config 檔案和那一整疊的 Layers 檔案給正確下載下來。

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

### 觀察：為什麼 v2 的 rebuild 這麼快？

因為 `v2` 利用了 **「Docker 快取機制（Cache Hit）」**。

在 `v1` 的版本中，把程式碼 `COPY app/ .` 放在最上面，導致每次只要改一個字，那層 Layer 的快取就失效了，連帶逼迫它底下的 `RUN pip install` 每次都要花 16 秒重新下載 Flask 套件。

而在 `v2` 中，把順序對調了,不常變動的 `requirements.txt` 提到上面先單獨執行 `pip install`。當 rebuild 時，Docker 發現相依清單根本沒變，就直接**秒過（顯示 CACHED）**，只有最下面複製程式碼的那一層花了不到 1 秒重新打包。這就是錯誤的指令排序與正確排序之間的巨大時間差距

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

- **症狀**：
  在最終執行 `git push origin main` 準備交作業時，進度條跑到一半突然卡住，接著終端機噴出 `error: 无法倒回 rpc post 数据`、`RPC 失败。curl 65 Recv failure` 以及 `fatal: 远端意外挂断了` 的中斷報錯。

- **診斷**：
  因為在 Part F 的實驗中，我們為了測試 `.dockerignore` 的威力，故意在專案目錄下用 `dd` 指令建立了一個高達 **150 MB** 的巨型垃圾檔案 `dummy_garbage.bin`。
  後來在執行 `git add w06/` 時，不小心把這個 150MB 的大魔王一起加進了 Git 的追蹤記憶中。GitHub 透過 HTTPS 限制了單次傳輸的緩衝區大小，當網路稍微延遲且檔案過大時，伺服器就會直接暴力斷開連線。

- **修正**：
  不需要去動網路快取設定，直接治本！執行以下 Git 撤回指令，把 150MB 的大垃圾從 Git 的暫存記憶中抹除（但保留在本機，不影響實驗紀錄）：
  ```bash
  git rm --cached w06/dummy_garbage.bin
  git commit --amend --no-edit

## 設計決策
**技術取捨**：為什麼 runtime 階段選擇 python:3.12-slim 而不是 alpine？

在 W06 的多階段建置（Multi-stage Build）中，最終的生產環境（Runtime）我們面臨了基礎映像檔（Base Image）的技術抉擇。雖然 alpine 映像檔以極度輕量（通常只有 5MB 左右）聞名，但本週我們依然決定選擇約 43MB 的 python:3.12-slim，原因與取捨如下：

    避免 C 語言編譯相依性的地獄（glibc vs musl）：
    python:3.12-slim 底層是基於 Debian Linux，使用的是標準的 glibc 函式庫；而 alpine 為了極致瘦身，使用的是輕量版的 musl libc。許多 Python 的高效能科學計算或網頁套件（例如未來可能用到的 NumPy, Pandas, 甚至部分密碼學套件），在編譯時都是針對 glibc 設計的。如果在 Alpine 上安裝，經常會因為找不到相依性而直接安裝失敗，迫使工程師要在 Dockerfile 裡手動裝上一整套 make, gcc 等重型工具，最後做出來的映像檔反而比 slim 還要胖。

    維運安全性與穩定性取捨：
    雖然 Alpine 體積小，但在生產環境中，部分底層 C 函式庫的行為差異可能會引發難以追查的記憶體錯誤（Segmentation Fault）或效能滑落。

**最終決策**：
我們選擇犧牲大約 30 多 MB 的磁碟空間換取極高的相容性與部署穩定度。透過 python:3.12-slim 搭配 Multi-stage build，最終做出來的 myapp:multi 體積（184 MB）已經非常精實，且能保證未來不管安裝任何複雜的 Python 擴充套件，都不會踩到 Alpine 的編譯地獄。

### 🌐 網路選取之 Dockerfile 原始碼 (Node.js 多階段建置)

```dockerfile
# 第一階段：編譯環境 (Builder Stage)
FROM node:20-alpine AS builder
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# 第二階段：最終執行環境 (Runtime Stage)
FROM node:20-alpine AS runner
WORKDIR /usr/src/app
ENV NODE_ENV=production
COPY --from=builder /usr/src/app/dist ./dist
COPY --from=builder /usr/src/app/node_modules ./node_modules
COPY --from=builder /usr/src/app/package.json ./package.json
USER node
EXPOSE 3000
CMD ["node", "dist/main.js"]
```

## 核心優點分析

    極致瘦身 (Multi-stage)：利用雙階段建置，在 builder 階段編譯完後，只將核心成品 dist 與 node_modules 搬移到 runner 階段，成功丟棄數百 MB 的編譯垃圾。

    快取優化 (Cache Hit)：先 COPY package*.json 再執行 npm install。只要相依套件沒變，改原始碼 rebuild 時就能直接秒過快取，大幅縮短建置時間。

    安全防護 (Non-root USER)：結尾使用 USER node 將容器權限從預設的最高權限 root 降級。即使容器被黑客攻破，也無法輕易控制宿主機系統，符合企業級安全規範。

## 實戰排錯紀錄 (Troubleshooting)

在 VMware 本地虛擬機實際對此 Dockerfile 進行 docker build 驗證時，踩到了以下兩個經典小坑並成功修復：

   **坑1**：npm ci 報錯中斷

        原因：原版程式碼使用 RUN npm ci，這要求目錄下必須有 package-lock.json 鎖定檔。因為是手動建立的極簡測試專案，缺少此檔案導致中斷。

        修正：將指令改為更通用的 RUN npm install。

   **坑2**：node_modules not found 報錯

        原因：手動建立的測試 package.json 內容中，dependencies 為空 {}。npm 偵測到無套件需下載，故完全沒有生成 node_modules 資料夾，導致第二階段 COPY --from 找不到目標而崩潰。

        修正：在 package.json 中手動加入一個標準網頁依賴套件 "express": "^4.19.2"，使資料夾順利生成。

**最終結果**：成功排除以上故障，並在終端機順利印出 Bonus Node App Running! 執行結果，驗證開源 Dockerfile 設計完全可行。
