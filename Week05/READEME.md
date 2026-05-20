# W05｜把容器拆開來看：Namespace / Cgroups / Union FS / OCI

## Docker 環境

- Storage Driver：`overlayfs`
- Cgroup Version：`2`
- Cgroup Driver：`systemd`
- Default Runtime： `io.containerd.runc.v2 runc`

## Namespace 觀察

### 六種 namespace 用途（用自己的話）
- PID：只會看到自己容器裡的 process
- NET：隔離網路環境，每個容器有：自己的 IP、routing table、網卡
- MNT：容器看到的是自己的檔案系統
- UTS：隔離 hostname，不同 container 可以有不同 hostname
- IPC：隔離程序間通訊包含：shared memory、semaphore、message queue
- USER：容器裡的root不一定是Host的root

### Host vs 容器 inode 對照
[namespace-table.md](https://hackmd.io/@JeC8TqmsRCywA0fGVF5WEA/ry82yYoyzl)

### 容器內 `ps aux` 輸出
只有看到3個process，因為在 container 中執行 ps aux 時，只能看到 container 內部的 process，包括 PID 1 的主程序 sleep 3600、互動 shell 與 ps 本身，無法看到 Host 的全部程序。

## Cgroups 實驗

### 容器內讀到的限制
- memory.max：268435456
- cpu.max：100000

### Host 端對照（用 `docker inspect -f '{{.HostConfig.CgroupParent}}'` 動態取得路徑）
- memory.max：268435456
- cpu.max：100000
- memory.current（執行時某一刻）：401408

### OOM 故障三階段
| 項目 | 故障前 | 故障中（memory=32m + dd 200m）| 回復後（memory=256m）|
|---|---|---|---|
| 容器 exit code | - | 137 | 0 |
| OOMKilled | - | true | false |
| dmesg 關鍵字 | 無 OOM | oom-killer、Memory cgroup out of memory、Killed process dd | 無 OOM |

## Image 分層

### `docker image inspect nginx:1.27-alpine` layer 數量
8個layers

### 兩個同源 image 共享 layer 的證據
本次實驗中觀察到兩者 layer hash 皆不同

### `docker diff` 輸出範例與解讀
| 符號 | 意思                       | 本次例子                                                 |
| -- | ------------------------ | ---------------------------------------------------- |
| A  | Added，新增檔案或目錄            | `A /tmp/hello.txt`、`A /etc/nginx/conf.d/custom.conf` |
| C  | Changed，內容或 metadata 被改變 | `C /etc/nginx/conf.d`、`C /tmp`                       |
| D  | Deleted，刪除檔案             | `D /etc/nginx/conf.d/default.conf`                   |


## OCI 呼叫鏈

| 元件 | 主要職責 |
|---|---|
| dockerd | Docker daemon，負責接收使用者的 docker CLI 指令，例如 `docker run`、`docker ps`，並管理 image、network、volume 等高階功能。 |
| containerd | 負責 container 生命周期管理，例如建立、啟動、停止 container，並管理 image pull 與 snapshot。 |
| containerd-shim | 作為 container 與 containerd 之間的中介程序，讓 container 在 containerd 重啟後仍可繼續運行，同時負責管理 container 的 stdin/stdout 與退出狀態。 |
| runc | 真正建立 container 的 OCI runtime，負責依照 OCI Runtime Spec 建立 namespace、cgroup、mount 等 Linux 隔離環境。 |

| `config.json` 欄位 | 功能 |
|---|---|
| `linux.namespaces` | 定義 namespace 隔離 |
| `linux.resources.memory` | 記憶體 cgroup 限制 |
| `linux.resources.cpu` | CPU cgroup 限制 |
| `mounts` | container 的掛載點設定 |
| `process` | container 啟動程序與環境變數 |
| `root.path` | container root filesystem 路徑 |


## 排錯紀錄
- 症狀：  在 container 中執行 `docker exec -it ns-demo sh` 後，發現 `ps aux` 顯示的 process 數量與 host 不同，且 pull alpine image 時出現 timeout，無法下載 image。
- 診斷：先使用 `ping 8.8.8.8` 測試外網連線，發現 app VM 無法連外；再檢查 VM 網卡設定，確認 app 僅配置 Host-only 網卡，因此無法存取外部網路與 DNS。
timeout。
- 修正：添加暫時性的NAT網卡

- 驗證：  增加 NAT 後，`ping 8.8.8.8` 可成功回應，且 Docker 可正常 pull image；完成 image 下載後，即使移除 NAT，container 仍可正常啟動與進行 namespace、filesystem 等實驗。

## 想一想（回答 3 題）
1. 容器裡的 PID 1 跟 host PID 1 是同一支 process 嗎？`kill -9 1`（在容器內）會發生什麼？
 不是同一支 process。Container 使用 PID namespace 建立獨立的 process 視圖，因此 container 內的 PID 1 通常是 container 的主程序，例如 sleep、nginx等，而 host 的 PID 1 則通常是 systemd/init。
2. 兩個容器都基於 `ubuntu:24.04`，磁碟空間是吃兩份還是共用？怎麼驗證？
共用，可以透過：
```
docker image inspect ubuntu:24.04
docker system df
```
或比較 image 的 .RootFS.Layers sha256 來觀察。若 layer sha256 相同，代表 Docker 共用相同 lay
3. 如果 host 的 kernel 爆漏洞，容器還能稱為「隔離」嗎？這個限制跟 VM 差在哪？
VM 擁有自己的 guest OS 與 guest kernel，與 host 之間透過 hypervisor 隔離，因此即使 guest kernel 被攻擊，也不一定能直接影響 host。相較之下，VM 的隔離性通常比 container 更強，但資源消耗與啟動成本也較高。
