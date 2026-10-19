所有 Ansible Playbook、Role 同 Tower Workflow 必須遵循最佳實踐；如因技術或環境限制無法做到，必須喺 README 清楚記錄原因。

# Ansible Tower OS Patch Workflow（Ansible 2.9）

本 Repository 用嚟整理 OS Patch 維護流程。目標係用 Ansible Tower Workflow 串連 9 個 Job Template；目前只完成部分 Playbook，其餘係 **故意會失敗嘅佔位檔案**，唔可以直接作正式維護使用。

## Tower Privilege Escalation 約定（重要）

**本專案假設 Ansible Tower Job Template 已啟用 Privilege Escalation（sudo → root）**，並由 Tower 管理登入憑證及提權設定。呢個係專案部署前提，**唔代表已核實 Tower 實際設定**；部署前必須確認。Repository 不需要另建 `ansible.cfg` 去重複設定。

- Playbook 預設繼承 Tower 提權設定；**不要重複加入** `become: true`、`become_method: sudo`、`become_user: root`。
- **必須以原登入帳戶執行**嘅 IWS/MDM/Process Check Play，保留 `become: false`，明確覆蓋 Tower 提權；不可一律刪除。
- 如將來 Tower 設定改變，必須重新審核 Playbook 權限；唔好假設刪除顯式提權仍會以 root 執行。
- 新 Playbook / Role 亦遵守同一約定，避免分散重複嘅權限設定。

## 預定 Workflow 次序

| # | Playbook | 工作 | 目前狀態 |
|---|---|---|---|
| 01 | `backup.yml` | Patch 前 Backup | **未實作** |
| 02 | `wait_until_time.yml` | 等到指定 Maintenance Start Time | **未實作** |
| 03 | `splunk_suppress.yml` | 暫時 Suppress Splunk 告警 | **未實作** |
| 04 | `IWS_before.yml` | IWS／FTA 前作業 | 已有流程；依賴外部 Roles |
| 05 | `before.yml` | OS Patch 前記錄 Kernel、檢查 Process | 已有程式 |
| 06 | `os_patch.yml` | OS Patch | **未實作** |
| 07 | `after.yml` | OS Patch 後比較 Kernel、檢查 Process | 已有程式 |
| 08 | `IWS_after.yml` | IWS／FTA 後作業 | 已有流程；依賴外部 Roles |
| 09 | `splunk_unsuppress.yml` | 恢復 Splunk 告警 | **未實作** |

正常成功路線（Success Path）：

```text
Backup
  -> Wait Until Time
  -> Splunk Suppress
  -> IWS Before
  -> OS Before
  -> OS Patch
  -> OS After
  -> IWS After
  -> Splunk Unsuppress
```

此表係目標設計，**未代表 9 個 Job 已經喺 Tower 建立、接線或者通過測試**。

## 已有 Playbook 嘅實際行為

### IWS_before.yml

依次執行：
1. `Workstation_control/fta_ws_limit_control_before`：記錄／處理 FTA WS Limit
2. `Workstation_control/fta_unlink`：FTA Unlink
3. `Workstation_control/stop_iwsfta`：停止 IWSFTA

使用 `mdm_dev`、`fta_dev` 同 `fta_ws` 等 Host / Group；失敗時相應 `rescue` 使用 `fail`。

### before.yml

對選定嘅 `all` Host 執行 `uname -a`，把結果存到目標主機 `~/tmp_os_before.txt`，再執行 `check_process` Role。**唔好將預期已停止嘅 IWS Process 加入此階段嘅檢查清單。**

### after.yml

從同一目標主機讀取 `~/tmp_os_before.txt`，再次執行 `uname -a`，顯示 Patch 前後差異，再執行 `check_process` Role。**Kernel 有冇變只會顯示，唔會單憑差異判斷 Patch 成功／失敗。**

### IWS_after.yml

依次執行：
1. `Workstation_control/start_iwsfta`：啟動 IWSFTA
2. `Workstation_control/fta_link`：FTA Link
3. `Workstation_control/fta_process_check`：IWSFTA Process Check
4. `Workstation_control/da_process_check`：IWSDA Process Check
5. `Workstation_control/fta_ws_limit_control_after`：恢復 FTA WS Limit
6. `Workstation_control/after_fta_check`：RMSTDLIST 檢查

**已知風險：** 多個 `rescue` 使用 `debug` + `meta: end_play`，未必令 Tower Job 標記為 Failed；正式使用前必須處理及測試失敗傳遞。

## 未實作步驟（安全佔位）

`backup.yml`、`wait_until_time.yml`、`splunk_suppress.yml`、`os_patch.yml`、`splunk_unsuppress.yml` 目前全部使用 `hosts: localhost`，並透過 `fail` 明確終止；**唔會真的 Backup、等待、呼叫 Splunk，亦唔會執行 OS Patch**。

尚待定義：
- Backup：備份對象、方式、儲存位置、成功判定同還原方式。
- Wait Until Time：時間輸入、時區、過期時間處理；日後可考慮 Tower Survey 傳值，但目前未設定。
- Splunk Suppress / Unsuppress：實際 API、驗證方式、對應告警範圍、維護窗口、授權同重試策略。
- OS Patch：實際 Patch Job 行為、目標 Host、重開機／驗證同回復策略。

正式啟用 Workflow 前，必須先完成以上功能及測試；唔可以靠移除 `fail` 當作完成。

## Inventory、Roles 同執行環境

- 目標環境：Ansible 2.9 / Ansible Tower；新增或修改任務時要留意版本相容性。
- `ansible.cfg` 指向 `inventory/hosts.yml` 同 `roles/`。
- `inventory/hosts.yml` 內 `192.0.2.x` 係示範 IP，**唔係正式機器**。
- 現有 Inventory 未提供 IWS Playbook 所需嘅 `mdm_dev`、`fta_dev`、`da_dev`、`fta_ws` 等 Group；Tower 必須配置實際 Inventory。
- Repo 內目前只有 `roles/check_process`；IWS 所用 `Workstation_control/*` Roles **唔喺此 Repo**，需要喺 Tower 執行環境／其他 Project 可正確引用，否則 Job 唔會正常運作。
- `roles/check_process` 透過 `common_processes`、`extra_processes` 建立檢查名單，以 `pgrep -x` 查 Process；主機必須有 `pgrep`。
- `before.yml` 同 `after.yml` 以 `hosts: all` 執行；喺 Tower 必須用正確 Inventory／Limit 避免誤觸其他 Host。
- Kernel 記錄會寫喺**目標主機執行帳戶**嘅 Home（唔係 Tower Controller）。Before / After 要對應同一批主機及可存取同一路徑嘅帳戶；Before 會覆寫檔案，**不可同一主機同時跑多個會互相覆寫嘅 Workflow**。

## Tower Workflow／失敗處理原則

- 規劃為 **9 個 Job Template**，正常路線以 Success 線串接；正式執行前需核對每個 Job 嘅 Project、Inventory、Credentials、Limit、Variables，同埋權限。
- Backup 未成功、未到維護窗口或者 Suppress 失敗，都唔應繼續做 IWS 停止／OS Patch。
- **Splunk 成功 Suppress 後，只靠 Success 線唔保證會 Unsuppress**：中途任何 Job 失敗，都可能令告警持續被 Suppress。
- Failure／Recovery Path **尚未實作**。必須設計經核對嘅人工或自動恢復步驟，確定 IWS／系統狀態安全後先解除告警；**唔好無條件將所有 Failure 線接去 Unsuppress**。
- 未確認 Backup、Patch／Rollback、Splunk API 同 IWS 啟停邏輯前，禁止當作 Production-ready Workflow。

## 驗證狀況

未喺真實 Tower／目標主機執行整體測試；亦未就呢次 README 更新宣稱已做 Syntax Check。

實作完成後，至少要做 Ansible 2.9 Syntax Check、Tower Inventory／Role 解析、Survey 參數驗證，同埋 Success／Failure／恢復路線測試。

只更新本文檔唔會改動任何 Playbook 或 Tower 設定。
