問題出在 gnome-keyring 啓動時異常，需要進行以下操作
```bash
systemctl --user status gnome-keyring # 檢查是否有錯誤訊息
systemctl --user restart gnome-keyring # 重啓之後就沒問題了
```

ref: https://github.com/zed-industries/zed/issues/14028
