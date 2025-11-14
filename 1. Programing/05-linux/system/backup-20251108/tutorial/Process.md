### ps
文檔說明: report a snapshot of the current processes.
```shell
# see every process on the system using `standard syntax`
ps -e
ps -ef # f 表示顯示更多資訊
ps -eF # 比 f 更多

# see every process on the system using `BSD syntax`
ps ax
ps axu
```
* standard sysntax 使用 - 前墜，而 BSD 沒有

### procs
```shell
procs # 直接就是 ps -ef
procs --tree # 使用樹模式
```