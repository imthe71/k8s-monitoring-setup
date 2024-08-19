# Kubernetes 效能監控儀表板說明

本文檔詳細介紹了兩個 Kubernetes 效能監控儀表板的配置，每個儀表板的面板內容，以及如何通過這些儀表板觀察 CPU Throttling 現象。

## 儀表板一：Node 效能監控

此儀表板旨在監控 Kubernetes 集群中各個節點的效能數據，幫助您了解系統資源的整體使用情況。

### 面板 1：CPU 使用率
- **指標**：`100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)`
- **內容說明**：顯示每個節點的 CPU 使用率，表現為百分比。此面板能幫助你識別 CPU 負載較高的節點，並進行相應的資源調整。

### 面板 2：記憶體使用率
- **指標**：`(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100`
- **內容說明**：顯示每個節點的記憶體使用率，表現為百分比。此面板能讓你了解記憶體資源的消耗情況，及時發現可能的瓶頸。

### 面板 3：磁碟 I/O
- **指標**：`rate(node_disk_reads_completed_total[5m])` 和 `rate(node_disk_writes_completed_total[5m])`
- **內容說明**：顯示每個節點的磁碟讀寫操作次數，單位為每秒操作數（ops）。此面板幫助你監控 I/O 密集型工作負載的影響。

### 面板 4：網路流量
- **指標**：`rate(node_network_receive_bytes_total[5m])` 和 `rate(node_network_transmit_bytes_total[5m])`
- **內容說明**：顯示每個節點的網路接收和發送數據流量，單位為每秒位數（bps）。此面板讓你可以監控網路使用情況，及時發現潛在的網路問題。

## 儀表板二：Kubernetes 叢集效能監控

此儀表板專注於 Kubernetes 叢集中各個 Pod 的資源使用情況，幫助你識別哪些工作負載消耗了最多的系統資源。

### 面板 1：Pod CPU 使用率
- **指標**：`sum(rate(container_cpu_usage_seconds_total{container!="POD",pod!=""}[5m])) by (pod)`
- **內容說明**：顯示每個 Pod 的 CPU 使用率，表現為百分比。這個面板能讓你識別出消耗最多 CPU 資源的 Pod，便於進行調整。

### 面板 2：Pod 記憶體使用率
- **指標**：`sum(container_memory_usage_bytes{container!="POD",pod!=""}) by (pod)`
- **內容說明**：顯示每個 Pod 的記憶體使用情況，表現為絕對值（如 MB 或 GB）。此面板幫助你監控各個 Pod 的記憶體消耗，避免記憶體不足的情況發生。

### 面板 3：Pod 數量變化
- **指標**：`count(kube_pod_info) by (namespace)`
- **內容說明**：顯示各命名空間中 Pod 的數量變化。這有助於你了解叢集中不同服務的部署情況和資源分配。

## 觀察 CPU Throttling 現象

CPU Throttling 是當一個 Pod 的 CPU 使用量超過了其配額時，Kubernetes 對該 Pod 的 CPU 使用進行限制，這可能會導致性能下降。以下是如何通過儀表板觀察和分析 CPU Throttling 現象的步驟。

### 如何配置 CPU Throttling 面板

1. **新增一個名為 "CPU Throttling" 的面板**：
   - **指標**：`sum(rate(container_cpu_cfs_throttled_seconds_total[5m])) by (pod)`
   - **內容說明**：該指標計算每個 Pod 在過去 5 分鐘內被限制 CPU 使用的秒數。當這個值增加時，表示該 Pod 正在遭受 CPU Throttling。

2. **觀察 Throttling 趨勢**：
   - 當你發現某個 Pod 的 Throttling 次數或持續時間顯著增加時，這表明該 Pod 的 CPU 資源需求超過了分配的配額。你可以根據這些數據來調整 Pod 的資源配額，或對工作負載進行優化。

3. **設置告警**：
   - 你可以設置一個告警條件，當某個 Pod 的 Throttling 次數達到一定閾值時，觸發告警，提醒你需要進行資源調整或分析應用程式的性能需求。
