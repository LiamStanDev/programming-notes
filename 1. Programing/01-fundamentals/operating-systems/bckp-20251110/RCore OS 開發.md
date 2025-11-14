
## Ch3: 多道程序與分時多任務
### 怎麽取得 Qemu 中設備資訊？

#### 什麼是 Device Tree？

Device Tree 是一種硬體抽象描述格式，它描述：

* 有哪些硬體（CPU、RAM、UART、網卡、GPIO、I2C、SPI、PCI 等等）
* 這些硬體的記憶體映射（memory-mapped I/O 的地址）
* 硬體之間的連接（像是某個 SPI 控制器下面連著哪個 sensor）
* 中斷控制器與編號

Device Tree 通常使用 .dts (Device Tree Source) 檔案來撰寫，然後編譯成 .dtb（Device Tree Blob，binary 格式）供 kernel 使用。

#### 取得 Qemu Device Tree 並解析
```bash
# 取得
qemu-system-riscv64 -machine virt,dumpdtb=/tmp/dump.dtb
# 反編譯
dtc -o /tmp/dump.dts /tmp/dump.dtb
# 讀取
vim /tmp/dump.dts
```