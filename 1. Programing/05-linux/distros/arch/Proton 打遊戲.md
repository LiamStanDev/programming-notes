### 啟動參數

| 啟動參數                       | 說明                                                               |
| -------------------------- | ---------------------------------------------------------------- |
| `PROTON_LOG=1`             | 產生 proton log, 位於 `$HOME/steam-<appid>.log`                      |
| `PROTON_USE_NTSYNC=1`      | Proton 使用新的、高效能的 Windows 內核同步機制：`ntsync`，取代舊的 `esync`/`fsync` 機制 |
| `PROTON_NO_ESYNC=1`        | 禁用 `esync`                                                       |
| `gamemoderun`              | 將 CPU governor 設為 `performance`, 提高響應與優先權                        |
| `MANGOHUD=1`               | 顯示完整 FPS/GPU/CPU 資訊                                              |
| `STEAMDECK=1`              | SteamDeck 欺騙用避免遊戲防作弊                                             |
| `PROTON_HIDE_NVIDIA_GPU=1` |                                                                  |

### 更新驅動之後沒有聲音
這問題發生在 wireplumber cache 沒有更新導致
```shell
rm -r ~/.local/state/wireplumber
```
可以看到裡面有 `stream-properties` 我猜就是它的問題

ref: https://bbs.archlinux.org/viewtopic.php?id=299212

