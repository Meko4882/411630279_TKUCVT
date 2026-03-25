## 至少四個維度的 VM vs Container 對照
| 維度     | 虛擬機器（VM）                       | 容器（Container）                       |
| -------- | ------------------------------------ | --------------------------------------- |
| 隔離層   | 完整 Guest OS + Hypervisor           | 共用 Host OS kernel                     |
| 啟動速度 | 數分鐘（要開整個 OS）                | 數秒（只啟動程序）                      |
| 資源佔用 | 重（每台需獨立 OS，通常 1+ GB）      | 輕（單機可跑數百個，MB 等級）           |
| 封裝內容 | 完整 OS + 應用程式 + 設定            | 應用程式 + 函式庫 + 相依項              |
| 映像大小 | 數 GB                                | 數十 MB ~ 數百 MB                       |
| 核心技術 | Hypervisor（VMware / KVM / Hyper-V） | Container Engine（Docker / containerd） |
| 回復方式 | Snapshot 還原                        | 重新拉取映像 / 重新部署                 |

## 本課選擇「VM 裡跑 Docker」的理由（用自己的話寫）
* 因為vm是個獨立的環境，在裡面遇到系統問題也可以透過之前的快照去還原，也不會影響到電腦本機系統

## Hypervisor Type 1 vs Type 2 的差異與本課的選擇
| 類型 | Type 1（are-metal Hypervisor) | Type 2（Hosted Hypervisor）|
| ---- | ------------------- | ----------------------------- |
| 架構 | 直接運行在硬體上 | 運行在作業系統上 |
| 效能 | 較高 | 較低 |
| 安裝難度 | 較高 | 較簡單 |
| 常見工具 | VMware ESXi、Microsoft Hyper-V Server、Xen | VMware Workstation、VirtualBox、Parallels、VMware Fusion |
| 使用情境 | 企業資料中心、雲端基礎設施（AWS EC2 底層就是 Type 1）| 個人開發、教學、本機測試 |
