* Arch Wiki: https://wiki.archlinuxcn.org/zh-tw/Rime
* Rime 定製指南: https://github.com/rime/home/wiki/CustomizationGuide
* https://github.com/rime/home/wiki/UserGuide#%E6%B3%A8%E9%9F%B3

```yaml
# ~/.local/share/fcitx5/rime/default.custom.yaml

patch:
  menu:
    page_size: 7
  # 使用 [] 換頁
  "key_binder/bindings":
    - { when: paging, accept: bracketleft, send: Page_Up }
    - { when: has_menu, accept: bracketright, send: Page_Down }
```