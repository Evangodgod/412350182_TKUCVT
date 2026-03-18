# VM vs Container 技術比較

| 維度 | Virtual Machine (VM) | Container (Docker) |
|---|---|---|
| **架構** | 包含完整的 Guest OS | 共用 Host OS Kernel |
| **隔離級別** | 硬體級隔離 (Hypervisor) | 作業系統級隔離 (Namespace/Cgroups) |
| **啟動速度** | 分鐘級 (需載入核心) | 秒級 (僅啟動程序) |
| **資源消耗** | 高 (固定配置 CPU/RAM) | 極低 (按需分配) |

## 實務心得
在本週實驗中，我觀察到：
- **Snapshot** 在 VM 層級非常強大，即使我改壞了系統設定（如 `docker.list`），也能瞬間恢復。
- **Docker** 容器雖然啟動快，但若沒有在 VM 層級做好保護，一旦 Host OS 毀損，容器也會隨之消失。
