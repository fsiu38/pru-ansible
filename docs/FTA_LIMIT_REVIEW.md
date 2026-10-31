# FTA Workstation Limit — Before / After 原碼研究筆記

> 根據 2026-10-31 兩張 VS Code 相片人工抄錄，尚未同公司原始檔逐字核對。
> 只作理解／Code Review，不是完成測試或可直接部署的修正版；Role 原有操作邏輯未重構。

## 核心用途

Before：記錄每個 FTA Workstation 原本嘅 Limit，改成 0，再等執行中的 Job Stream 完成。
After：讀取 Before 儲存嘅 Limit，將 Workstation Limit 恢復。

## Before 12 個 Task

1. 查工作站狀態，儲存在 fta_status_before。
2. 用 stdout.split()[4] 取出 Limit（即第 5 個欄位）。
3. 將工作站名稱和 Limit 合併入 fta_limit_before 字典。
4. 用 set_stats 將該字典作為 Workflow artifact 儲存。
5. Debug 顯示原本 Limit。
6. 用 conman lc 指令把 Limit 改為 0。
7. 顯示 lc 指令輸出。
8. 再次查工作站狀態。
9. 顯示工作站狀態。
10. 用 conman ss 指令每隔 60 秒查執行中的 Job Stream，最多重試 10 次。
11. Debug 顯示最後查詢結果。
12. 若最後仍有 Job Stream，執行 fail。

## After 7 個 Task

1. 查現在工作站狀態。
2. Debug 顯示狀態。
3. 用 conman lc 指令，把 fta_limit_before[fta_target_ws] 記錄的 Limit 設返。
4. Debug 顯示 lc 輸出。
5. Pause 5 秒。
6. 再查工作站狀態。
7. Debug 顯示完整結果。

## 三個 Conman 命令

- sc：查看指定 Workstation 狀態。
- lc：更改指定 Workstation 嘅 Limit。
- ss：查看指定 Workstation 嘅 Job Stream 狀態。

## 重要變數

- fta_target_ws：由既有 IWS_before.yml／IWS_after.yml 用 groups['fta_ws'] loop 傳入。
- tws_env_load 和 conman_cmd：原 repo group_vars/mdm_dev.yaml 定義。
- fta_limit_before_value：Before 查詢的 Limit。
- fta_limit_before：Workstation 名稱映射到原本 Limit 的字典；After 必須收到。

## 值得慢慢檢查，但這個 branch 沒有修正的問題

- 相片抄錄未獲公司原始程式碼核對；尤其 shell 指令符號要逐字確認。
- split()[4] 只是假設 sc 輸出第 5 個欄位永遠是 Limit。
- set_stats 跨 Tower Job 傳值，需要同一個 Workflow 並確認 artifacts 真正傳到 After。
- 未用 assert 驗證 Limit 改為 0／恢復後係原值。
- shell 管線可能掩蓋 conman 指令的錯誤，|& 需要 Bash。
- wait_jobstream 的 failed_when: false、until、retries 配合可能出現錯誤傳遞疑問。
- 千祈唔好直接喺 production 執行呢兩個 Role；佢哋會真的改排程 Limit。
- 這個 repo 是 public；不要再加入未授權公開的公司內部資料或秘密。

## 既有入口

- IWS_before.yml：include_role Workstation_control/fta_ws_limit_control_before。
- IWS_after.yml：include_role Workstation_control/fta_ws_limit_control_after。
- 本次沒有改動上述 Playbook、Inventory 或 Tower 設定。
