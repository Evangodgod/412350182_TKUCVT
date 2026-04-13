# W03｜多 VM 架構：分層管理與最小暴露設計

## 網路配置

| VM | 角色 | 網卡 | 模式 | IP | 開放埠與來源 |
|---|---|---|---|---|---|
| bastion | 跳板機 | NIC 1 | NAT | 192.168.152.128 | SSH from any |
| bastion | 跳板機 | NIC 2 | Host-only | 192.168.200.128 | — |
| app | 應用層 | NIC 1 | Host-only | 192.168.200.129 | SSH from 192.168.56.0/24 |
| db | 資料層 | NIC 1 | Host-only | 192.168.200.130 | SSH from app + bastion |

## SSH 金鑰認證

- 金鑰類型: ed25519
- 公鑰部署到：~/.ssh/authorized_keys
- 免密碼登入驗證：
  - bastion → app：
  - bastion → db：
  - ![免密碼登入驗證](./images/ok.png)

## 防火牆規則

### app 的 ufw status
（貼上 `sudo ufw status verbose` 輸出）

### db 的 ufw status
（貼上 `sudo ufw status verbose` 輸出）

### 防火牆確實在擋的證據
![防火牆確實在擋](./images/fail8080.png)

## ProxyJump 跳板連線
- 指令：evan@bastion:~$ mkdir -p ~/.ssh
cat >> ~/.ssh/config << 'EOF'
Host bastion
    HostName 192.168.200.128       
    User evan          

Host app
    HostName 192.168.200.129   
    User evan      
    ProxyJump bastion

Host db
    HostName 192.168.200.130  
    User evan     
    ProxyJump bastion
EOF

chmod 600 ~/.ssh/config

- 驗證輸出：
![驗證輸出](./images/hostnamecheck.png)
- SCP 傳檔驗證：![scp](./images/scp.png)

## 故障場景一：防火牆全封鎖

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| app ufw status | active + rules | deny all | （填入） |
| bastion ping app | 成功 | （填入） | （填入） |
| bastion SSH app | 成功 | **timed out** | （填入） |

## 故障場景二：SSH 服務停止

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| ss -tlnp grep :22 | 有監聽 | 無監聽 | （填入） |
| bastion ping app | 成功 | 成功 | （填入） |
| bastion SSH app | 成功 | **refused** | （填入） |

## timeout vs refused 差異
（用自己的話說明兩種錯誤的差異、各自指向什麼排錯方向）

## 網路拓樸圖
（嵌入或連結 network-diagram）

## 排錯紀錄
- 症狀：
- 診斷：（你首先查了什麼？）
- 修正：（做了什麼改動？）
- 驗證：（如何確認修正有效？）

## 設計決策
（說明本週至少 1 個技術選擇與取捨，例如：為什麼 db 允許 bastion 直連而不是只允許從 app 跳？）
