# W02｜VMware 網路模式與雙 VM 排錯

## 網路配置

| VM | 網卡 | 模式 | IP | 用途 |
|---|---|---|---|---|
| dev-a | NIC 1 | NAT | 192.168.152.128 | 上網 |
| dev-a | NIC 2 | Host-only | 192.168.200.128 | 內網互連 |
| server-b | NIC 1 | Host-only | 192.168.200.129 | 內網互連 |

## 連線驗證紀錄

- [x] dev-a NAT 可上網：evan@dev-a:~$ ping -c 4 google.com
PING google.com (142.250.196.206) 56(84) bytes of data.
64 bytes from nctsaa-ac-in-f14.1e100.net (142.250.196.206): icmp_seq=1 ttl=128 time=3.18 ms
64 bytes from nctsaa-ac-in-f14.1e100.net (142.250.196.206): icmp_seq=2 ttl=128 time=3.42 ms
64 bytes from nctsaa-ac-in-f14.1e100.net (142.250.196.206): icmp_seq=3 ttl=128 time=3.63 ms
64 bytes from nctsaa-ac-in-f14.1e100.net (142.250.196.206): icmp_seq=4 ttl=128 time=2.93 ms

--- google.com ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 2.928/3.288/3.627/0.261 ms
- [x] 雙向互 ping 成功：evan@dev-a:~$ ping -c 4 192.168.200.129
PING 192.168.200.129 (192.168.200.129) 56(84) bytes of data.
64 bytes from 192.168.200.129: icmp_seq=1 ttl=64 time=0.753 ms
64 bytes from 192.168.200.129: icmp_seq=2 ttl=64 time=0.974 ms
64 bytes from 192.168.200.129: icmp_seq=3 ttl=64 time=0.980 ms
64 bytes from 192.168.200.129: icmp_seq=4 ttl=64 time=0.395 ms

--- 192.168.200.129 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3058ms
rtt min/avg/max/mdev = 0.395/0.775/0.980/0.237 ms

evan@server-b:~$ ping -c 4 192.168.200.128
PING 192.168.200.128 (192.168.200.128) 56(84) bytes of data.
64 bytes from 192.168.200.128: icmp_seq=1 ttl=64 time=1.23 ms
64 bytes from 192.168.200.128: icmp_seq=2 ttl=64 time=0.796 ms
64 bytes from 192.168.200.128: icmp_seq=3 ttl=64 time=0.435 ms
64 bytes from 192.168.200.128: icmp_seq=4 ttl=64 time=0.586 ms

--- 192.168.200.128 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3023ms
rtt min/avg/max/mdev = 0.435/0.761/1.230/0.299 ms
- [x] SSH 連線成功：evan@dev-a:~$ # 格式：ssh 正確帳號@IP
ssh evan@192.168.200.129
evan@192.168.200.129's password: 
Welcome to Ubuntu 24.04.4 LTS (GNU/Linux 6.17.0-19-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

扩展安全维护（ESM）Applications 未启用。

37 更新可以立即应用。
这些更新中有 23 个是标准安全更新。
要查看这些附加更新，请运行：apt list --upgradable

启用 ESM Apps 来获取未来的额外安全更新
请参见 https://ubuntu.com/esm 或者运行: sudo pro status

Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings

Last login: Thu Mar 26 03:13:56 2026 from 192.168.200.128
evan@server-b:~$ hostname
server-b
- [x] SCP 傳檔成功：`cat /tmp/test-from-dev.txt` 在 server-b 上的輸出
- [x] server-b 不能上網：`ping 8.8.8.8` 失敗輸出

## 故障演練一：介面停用

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| server-b 介面狀態 | UP | DOWN | （填入） |
| dev-a ping server-b | 成功 | 失敗 | （填入） |
| dev-a SSH server-b | 成功 | 失敗 | （填入） |

## 故障演練二：SSH 服務停止

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| ss -tlnp grep :22 | 有監聽 | 無監聽 | （填入） |
| dev-a ping server-b | 成功 | 成功 | （填入） |
| dev-a SSH server-b | 成功 | Connection refused | （填入） |

## 排錯順序
（寫出你的 L2 → L3 → L4 排錯步驟與每層使用的命令）

## 網路拓樸圖
（嵌入或連結 network-diagram.png）

## 排錯紀錄
- 症狀：
- 診斷：（你首先查了什麼？用了哪個命令？）
- 修正：（做了什麼改動？）
- 驗證：（如何確認修正有效？）

## 設計決策
（說明本週至少 1 個技術選擇與取捨，例如：為什麼 server-b 只設 Host-only 不給 NAT？）
