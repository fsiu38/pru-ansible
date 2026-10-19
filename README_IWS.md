# IBM Workload Scheduler (IWS / TWS) — Fault-Tolerant Agent (FTA) Internals

> 技術筆記：FTA 內部 Processes、Job 執行流程、Message Queues、Troubleshooting，以及 OS Patch 前後嘅安全操作考慮。  
> 參考 IBM Workload Automation 10.2.x 官方文件；**實際安裝版本及 OS 未確認**，所有指令必須喺相應版本及測試環境驗證。  
> 本文係架構／操作參考，**唔係已驗證可直接於 Production 執行嘅 Runbook**。

## 1. FTA 係乜？

FTA（Fault-Tolerant Agent）係 IBM Workload Scheduler 嘅排程執行節點。FTA 可以喺本地持有 Production Plan（`Symphony`），按 Dependency 執行及追蹤 Job，並同 Domain Manager / Master Domain Manager 交換訊息。

幾個必須分清楚嘅狀態：

- **Linked / Unlinked**：Workstation 同上層之間嘅通訊連線狀態。
- **Scheduler Processes Running / Stopped**：FTA 本地管理 Processes 有冇運行。
- **Jobs Running / Drained**：有冇正在執行嘅工作；`limit cpu 0` 唔等於已 Drain。
- **Job Status Synchronized**：本地同上層嘅 Job Status 有冇成功同步。

呢四項唔可以只靠一個 `ps` 或單一 `conman` 狀態推斷。

## 2. FTA 概念架構

```text
             Master Domain Manager / Domain Manager
                        ⇅ Network / Link
  ┌──────────────────────── FTA ────────────────────────┐
  │ netman ── writer ── mailman    [network / messaging]│
  │                        ⇅                            │
  │             Intercom.msg / Mailbox.msg              │
  │                        ⇅                            │
  │                batchman ⇄ Symphony                  │
  │                        │                            │
  │                    Courier.msg                      │
  │                        │                            │
  │                      jobman                         │
  │                        │                            │
  │             job monitor (jobmon)                     │
  │                        │                            │
  │             jobmanrc / .jobmanrc                    │
  │                        │                            │
  │                 OS Job / Script                     │
  │                                                     │
  │ monman: event monitoring（另一條功能路線）          │
  └─────────────────────────────────────────────────────┘
```

**注意：** 上圖只係方便理解嘅概念圖，唔代表所有 Processes 直接以圖中箭咀通訊。真正 Job Status 會經 Message Files 傳遞（見第 4 節）。

## 3. FTA Processes 詳解

| Process / 元件 | 主要職責 | 分辨重點 |
|---|---|---|
| `netman` | Network listener，建立／處理 Workstation 連接 | 網絡入口，唔係 Job Scheduler |
| `writer` | 將收到嘅遠端訊息傳畀本地 `mailman` | 接收訊息橋樑 |
| `mailman` | 將訊息路由到本地或遠端 Workstation | **唔係 Email Sender** |
| `batchman` | 讀取／更新 `Symphony`、處理 Dependencies、選擇可執行 Job | 本地 Scheduling Controller |
| `jobman` | 接收 Job Launch Request、啟動及追蹤 Job | Job Execution Controller |
| **job monitor（jobmon）** | 為單個 Job 設定環境、執行 Command / Script、回報結果 | 唔等於另一個必定常駐嘅 Daemon |
| `monman` | Event Monitoring / Event Rules | 同 `mailman` 唔同 |
| `Symphony` | 本地 Production Plan 資料 | 係檔案，**唔係 Process** |

### jobman vs jobmon

- `batchman` 決定工作可唔可以執行。
- `jobman` 收到 Launch Message 後，建立 Job Monitor。
- **Job Monitor（常稱 jobmon）** 準備 Job Environment，再運行該 Job。
- Job 完成後，結果經 `jobman` 回報。
- IBM 文件指出：喺 **UNIX / Linux** 上，Job Monitor 亦可能以 **`jobman`** Process 形式出現；**Windows** 文件列出 `JOBMON.exe` 同 `joblnch.exe`。所以多個 `jobman` 並唔必然代表異常。

### mailman vs monman vs 「mailmon」

- `mailman`：Scheduling / Job Status 嘅訊息路由。
- `monman`：Event Monitoring。
- **`mailmon` 唔係上述 IBM 官方標準 FTA Process 清單中嘅名稱**。若實際 OS 見到 `mailmon`，要先確認係內部自訂程式、Wrapper、別名定其他產品；唔應直接當成 `mailman`。

### FTA vs Dynamic Agent (DA)

FTA 主要係 `netman / writer / mailman / batchman / jobman`。Dynamic Agent 主要係 `agent / JobManager / TaskLauncher`，特定網絡配置會用 `JobManagerGW`。**FTA Stop/Start 唔應假設已涵蓋 DA。**

IBM 官方：[IBM Workload Scheduler components](https://www.ibm.com/docs/en/workload-automation/10.2.7?topic=overview-workload-scheduler-components)；[Workstation processes](https://www.ibm.com/docs/en/workload-automation/10.2.4?topic=commands-workload-scheduler-workstation-processes)。

## 4. 一個 Job 由 READY 到完成嘅訊息流程

```text
Symphony (Production Plan)
   │
   ▼
batchman  ── Dependency OK / Job Eligible
   │
   ▼
Courier.msg            [batchman 寫；jobman 讀]
   │
   ▼
jobman
   │
   ├─ 建立 Job Monitor
   │    ├─ 設定環境變數
   │    ├─ 執行 TWS_home/jobmanrc
   │    ├─ 視設定執行 USER_HOME/.jobmanrc (UNIX)
   │    └─ 執行 Job Command / Script
   │
   └─ Job Start / Completion Status
                  │
                  ▼
              Mailbox.msg       [mailman 讀]
                  │
                  ▼
                mailman
              /         \
             ▼           ▼
        Domain Manager  Intercom.msg
                            │
                            ▼
                         batchman
                            │
                            ▼
                   Update local Symphony
                   / select next job
```

呢個流程同時解釋咗點解：
- **Job 成功執行 ≠ MDM 即刻收到成功狀態**；
- `mailman` 或 Message Queue 有問題時，可能見到狀態延遲或不同步；
- `batchman` 負責將本地 Job 狀態更新到 `Symphony`。

### 關鍵 Message Files

| File | Writer → Reader | 用途 |
|---|---|---|
| `Courier.msg` | `batchman` → `jobman` | 請求啟動 Job |
| `Mailbox.msg` | `jobman`、其他來源 → `mailman` | Job Status 同其他訊息 |
| `Intercom.msg` | `mailman` → `batchman` | 將狀態／指令交返本地 Production Controller |
| `NetReq.msg` | 本地指令來源 → `netman` | Start / Stop / Link / Unlink 等本地請求 |
| `Monbox.msg`、`Moncmd.msg` | 多個元件 → `monman` | Event Monitoring 指令／訊息 |

**唔好隨意刪除、清空或者手動修改 `*.msg` 同 `Symphony`**；如果真係涉及 Message Queue / Plan 修復，必須跟指定產品版本嘅 IBM Recovery Procedure，先 Backup 再處理。

IBM 官方：[Workstation inter-process communication](https://www.ibm.com/docs/en/workload-automation/10.2.7?topic=commands-workstation-inter-process-communication)；[Network communication / Job processing](https://www.ibm.com/docs/en/workload-automation/10.2.6?topic=commands-workload-scheduler-network-communication)；[Job environment](https://www.ibm.com/docs/en/workload-automation/10.2.7?topic=environment-job-overview)。

## 5. Troubleshooting 對照表

| 症狀 | 優先檢查 | 原因假設，未經診斷唔可直接斷定 |
|---|---|---|
| FTA 不能 Link 上層 | `netman`、`writer`、`mailman`、網絡、DNS、TLS、Link 設定 | Network / Message Delivery 問題 |
| Linked 但 Job 唔起 | `batchman`、`jobman`、Dependencies、Limit、Priority | Scheduler 未放行或 Launch 路線失敗 |
| Job 開始後立即 FAIL | Job Monitor、`jobmanrc`、`.jobmanrc`、OS 帳戶、權限 | Environment / Command 問題 |
| OS Job 已完成但 Status 未更新 | `jobman`、`Mailbox.msg`、`mailman`、`Intercom.msg` | 回報／同步延遲 |
| Event Rule 無反應 | `monman`、Event Rule / Monitoring Configuration | Event Monitoring 問題 |

Linux / UNIX **只讀**初步檢查範例：

```bash
ps -ef | grep -E '[n]etman|[m]ailman|[b]atchman|[j]obman|[w]riter|[m]onman'
```

呢個只係 Process 名稱搜尋：可能包含命令參數匹配、Wrapper 或漏報；**唔可以單靠 Process 數量判斷 FTA Healthy**。應結合 `conman` 讀取 Workstation / Job 狀態、IWS Logs 同實際網絡連線檢查。確切 `conman` 語法需按安裝版本核實。

## 6. OS Patch / Ansible Tower：最重要嘅安全概念

### limit cpu 0 唔等於冇 Job

IBM `limit cpu` 文件指出：設定 0 後，特定 `hi` / `go` Priority 嘅 Jobs / Job Streams **仍可能執行**；已經 Run 緊嘅 Jobs 亦唔會因為將 Limit 設為 0 就自動完成。

**所以 Before Maintenance 要核實 Running Jobs 已經 Drain 完（或有正式批核嘅處理方案），而唔係淨係檢查 Limit=0。**

### unlink 唔等於 stop

`unlink` 係**關閉 Workstations 之間嘅通訊 Link**，唔係停止本地 Scheduling Processes，亦唔係 Kill 正在運行嘅 Jobs。重新 Link 後，積存訊息可能需要同步。

### stop 同 shutdown 唔同

- `conman stop`：停止 IBM Workload Scheduler Production Processes。
- 官方文件指出停止 `netman` 要用 `shutdown`（`conman shut`）。
- `shutdown` 係影響執行 `conman` 嘅**本地 Workstation**，而且官方要求先 `unlink`。
- 命令作用範圍、操作權限、服務依賴，必須按 FTA 版本、OS、執行位置確認；**唔好直接將上述命令順序複製到 Production**。

IBM 官方：[limit cpu](https://www.ibm.com/docs/en/workload-automation/10.2.7?topic=commands-limit-cpu)；[unlink](https://www.ibm.com/docs/en/workload-automation/10.2.7?topic=commands-unlink)；[stop](https://www.ibm.com/docs/en/workload-automation/10.2.7?topic=commands-stop)；[shutdown](https://www.ibm.com/docs/en/workload-automation/10.2.7?topic=commands-shutdown)。

### 建議 IWS Before 驗證點

```text
Preflight: IWS version / target hosts / connections / permissions
    ↓
Snapshot: FTA WS Limits + Link / Agent / Running Job states
    ↓
FTA WS Limit Control (Before)
    ↓
Verify no new work + confirm Running Jobs drained
    ↓
FTA Unlink
    ↓
Stop IWSFTA
    ↓
Verify intended FTA Processes stopped (incl. netman if required)
    ↓
OS Before → OS Patch
```

### 建議 IWS After 驗證點

```text
Start IWSFTA
    ↓
Verify relevant FTA Processes / local scheduler health
    ↓
FTA Link
    ↓
Verify communication and Job Status synchronization
    ↓
FTA / DA Process Checks (separately)
    ↓
Restore original FTA WS Limits
    ↓
RMSTDLIST / Post-check
    ↓
Confirm service recovered before Splunk Unsuppress
```

**兩條流程都係設計建議，唔代表現有 Roles 已經包含這些驗證。** 如 Start / Link / Process Check 失敗，要明確回傳 Tower Job Failed，唔可以用 `debug` + `meta: end_play` 令錯誤被當作成功。

## 7. 本 Repository 目前狀態

本專案目標 Workflow：

```text
backup.yml
  → wait_until_time.yml
  → splunk_suppress.yml
  → IWS_before.yml
  → before.yml
  → os_patch.yml
  → after.yml
  → IWS_after.yml
  → splunk_unsuppress.yml
```

- `IWS_before.yml` 現有 3 個階段：FTA WS Limit Before → FTA Unlink → Stop IWSFTA。
- `IWS_after.yml` 現有 6 個階段：Start IWSFTA → FTA Link → FTA Process Check → DA Process Check → FTA WS Limit After → RMSTDLIST。
- IWS Playbooks 依賴 `Workstation_control/*` Roles；目前**唔喺本 Repository**，所以無法審核 Role 內部嘅 `conman` 指令、Retries、Timeouts、Drain 邏輯或 Idempotency。
- 根據已見到嘅 YAML，`IWS_after.yml` 多處 `rescue` 用 `debug` + `meta: end_play`；呢個模式**未能保證 Tower 會將故障標為 Failed**，需要改善。
- 本文唔修改任何現有 Playbook、Tower Workflow 或 Runtime 設定。

## 8. IBM 官方參考資料

1. [Workload Scheduler components (10.2.7)](https://www.ibm.com/docs/en/workload-automation/10.2.7?topic=overview-workload-scheduler-components)
2. [Workstation processes (10.2.4)](https://www.ibm.com/docs/en/workload-automation/10.2.4?topic=commands-workload-scheduler-workstation-processes)
3. [Job environment overview (10.2.7)](https://www.ibm.com/docs/en/workload-automation/10.2.7?topic=environment-job-overview)
4. [Inter-process communication (10.2.7)](https://www.ibm.com/docs/en/workload-automation/10.2.7?topic=commands-workstation-inter-process-communication)
5. [Network communication (10.2.6)](https://www.ibm.com/docs/en/workload-automation/10.2.6?topic=commands-workload-scheduler-network-communication)
6. [Limit CPU (10.2.7)](https://www.ibm.com/docs/en/workload-automation/10.2.7?topic=commands-limit-cpu)
7. [Unlink (10.2.7)](https://www.ibm.com/docs/en/workload-automation/10.2.7?topic=commands-unlink)
8. [Stop (10.2.7)](https://www.ibm.com/docs/en/workload-automation/10.2.7?topic=commands-stop)
9. [Shutdown (10.2.7)](https://www.ibm.com/docs/en/workload-automation/10.2.7?topic=commands-shutdown)

---

*用途：IWS / FTA 技術教育、Troubleshooting 同 OS Patch 自動化設計參考。正式操作以公司已批核 Runbook、實際 IWS 版本同 IBM 官方文件為準。*
