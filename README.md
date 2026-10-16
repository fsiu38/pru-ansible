# Ansible Tower OS patch workflow（Ansible 2.9）

## Workflow 次序

`IWS_before.yml` → `before.yml` → 你現有嘅 OS patch Job → `after.yml` → `IWS_after.yml`

Tower 建立四個 Job Template，分別選以上 playbook，再用 Workflow 嘅 Success 線串起。OS patch Job 唔包含喺呢個 repo。

## Inventory 同程序檢查

`inventory/hosts.yml` 嘅 IP 係文件示範地址，請換成真實地址，並設定 Tower SSH credentials。
`common_processes` 同 `extra_processes` 定義要檢查嘅程序；role 用 `pgrep -x`，要求 Linux 主機有 pgrep。

注意：`before.yml` 係 IWS 停止之後先執行，所以唔好將 IWS 程序加入呢兩份 process list，否則會因預期停止而失敗。

所有 Ansible Playbook、Role 同 Tower Workflow 必須遵循最佳實踐；如因技術或環境限制無法做到，必須喺 README 清楚記錄原因。
