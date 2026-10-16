# Ansible Tower OS patch workflow（Ansible 2.9）

## Workflow 次序

`IWS_before.yml` → `before.yml` → 你現有嘅 OS patch Job → `after.yml` → `IWS_after.yml`

Tower 建立四個 Job Template，分別選以上 playbook，再用 Workflow 嘅 Success 線串起。OS patch Job 唔包含喺呢個 repo。
原本 `site.yml` 已拆成 `before.yml` 同 `after.yml`。

## 各步做啲咩

- `IWS_before.yml`：檢查 IWS 正常運行，將 stdout 同 return code 以 JSON 寫入目標主機 `~/tmp_iws_before.txt`，停止，再等停止完成。
- `before.yml`：執行 `uname -a`，寫入目標主機 `~/tmp_os_before.txt`，檢查設定嘅程序。
- `after.yml`：讀取 kernel 記錄，執行 `uname -a` 並顯示差異，再檢查程序。Kernel 改變唔會令 Job 失敗。
- `IWS_after.yml`：讀取之前狀態，啟動 IWS，最多重試 12 次、每次相隔 5 秒，確認 return code 同之前一樣為 0。完整輸出亦會顯示對比；PID／時間唔同唔當失敗。

## 必須設定：IWS 指令

未知道你真正嘅 IWS 指令，所以唔會亂估。以下只係 **systemd service 示範**，請喺 Tower inventory/group/host variables 換成實際指令：

```yaml
iws_status_argv: [systemctl, is-active, YOUR_IWS_SERVICE]
iws_stop_argv: [systemctl, stop, YOUR_IWS_SERVICE]
iws_start_argv: [systemctl, start, YOUR_IWS_SERVICE]
iws_stopped_rc: 3
```

指令用 argv list，唔支援 shell 管道／重定向。Status 指令必須以 rc 0 表示運行中，停止時回傳 `iws_stopped_rc`（預設 3）；其他 rc 當錯誤。自訂 IWS script 如果唔符合呢個約定，需要 wrapper 或調整檢查邏輯。停止同啟動指令失敗會中止 Job。

四個 Job 必須用同一批目標主機、相同 remote/become 帳戶；`~` 係執行帳戶嘅 home，唔係 Tower controller。需要權限時喺 Template 啟用 privilege escalation。每次 before 會覆寫記錄，唔好對同一主機同時跑多個 maintenance workflow。記錄缺失時 after 會失敗，唔會盲目啟動。

## Inventory 同程序檢查

`inventory/hosts.yml` 嘅 IP 係文件示範地址，請換成真實地址，並設定 Tower SSH credentials。
`common_processes` 同 `extra_processes` 定義要檢查嘅程序；role 用 `pgrep -x`，要求 Linux 主機有 pgrep。

注意：`before.yml` 係 IWS 停止之後先執行，所以唔好將 IWS 程序加入呢兩份 process list，否則會因預期停止而失敗。

## 驗證

```bash
ansible-playbook --syntax-check IWS_before.yml
ansible-playbook --syntax-check before.yml
ansible-playbook --syntax-check after.yml
ansible-playbook --syntax-check IWS_after.yml
```

未喺真實 Tower／目標主機執行測試。Workflow 中途失敗時，Success 線唔會自動執行 IWS_after；維護人員需按實際 patch 狀態決定恢復，唔好無條件接 Failure 線去啟動。
