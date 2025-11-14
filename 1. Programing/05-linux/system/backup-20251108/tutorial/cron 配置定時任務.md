用 `cron` 来安排每日快照也是一个很好的选择。`cron` 是一个在 Unix-like 系统上用来定期执行任务的工具。以下是如何使用 `cron` 来每天自动创建 Btrfs 快照的步骤：

### 1. 编辑 `crontab` 文件：
- 使用 `crontab -e` 命令打开当前用户的 `crontab` 文件。如果你是第一次使用，它可能会让你选择一个文本编辑器。

### 2. 添加 `cron` 任务：
- 在打开的 `crontab` 文件中，添加一行来定义你的定时任务。以下是一个例子：
```
0 2 * * * /usr/bin/btrfs subvolume snapshot /path/to/subvolume /path/to/snapshot/location/$(date +\%Y\%m\%d\%H\%M\%S)
```

* `0 2 * * *` 表示这个任务将在每天的凌晨2点执行。你可以根据需要调整这个时间。
- `/usr/bin/btrfs subvolume snapshot /path/to/subvolume /path/to/snapshot/location/$(date +\%Y\%m\%d\%H\%M\%S)` 是执行的命令。请确保替换 `/path/to/subvolume` 和 `/path/to/snapshot/location/` 为你的实际路径。`$(date +\%Y\%m\%d\%H\%M\%S)` 会用当前的日期和时间来命名快照，确保每个快照都有唯一的名字。

### 3. 保存并退出编辑器：

- 保存你的更改并退出编辑器。`cron` 会自动加载这些更改。

### 4. (可选) 检查 `crontab`：
- 使用 `crontab -l` 命令查看当前用户的 `crontab`，确保你的任务已被正确添加。
    

### 注意事项：
- 确保你有执行快照命令的权限，或者使用 `sudo crontab -e` 来编辑 root 用户的 `crontab`，这样任务就会以 root 权限运行。
- 对于更复杂的脚本，建议先写一个 shell 脚本来执行相关命令，然后在 `crontab` 中调用这个脚本。
- 如果你的系统上安装了 `anacron`，它可以帮助确保即使在预定时间系统关闭的情况下，错过的任务也能在系统下次启动时运行。