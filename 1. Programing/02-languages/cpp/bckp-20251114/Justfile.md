ref: https://just.systems/man/zh/chapter_27.html

### 變量
```shell
tmpdir := `mktemp -d` # `...` 中的會運行後賦值
version := "0.2.7"
tardir  := tmpdir / "awesomesauce-" + version # 路徑可以使用 / 拼接
tarball := tardir + ".tar.gz" # 字串可以使用 + 拼接

publish:
  rm -f {{tarball}} # {{}} 中使用變量
  mkdir {{tardir}}
  cp README.md *.c {{tardir}}
  tar zcvf {{tarball}} {{tardir}}
  scp {{tarball}} me@server.com:release/
  rm -rf {{tarball}} {{tardir}}
```


### 錯誤忽略
默認上一條運行失敗就不會進行下一條，你可以在前面加上 `-` 表示錯誤也繼續。
```shell
foo:
  -cat foo # 錯誤也繼續
  echo 'Done!'
```


### 內制函數
#### 系統信息
* `arch()` — 指令集结构。
	* 可能的值是："aarch64", "arm", "asmjs", "hexagon", "mips", "msp430", "powerpc", "powerpc64", "s390x", "sparc", "wasm32", "x86", "x86_64", 和 "xcore"。
* `os()` — 操作系统
	* 可能的值是: "android", "bitrig", "dragonfly", "emscripten", "freebsd", "haiku", "ios", "linux", "macos", "netbsd", "openbsd", "solaris", 和 "windows"。
* `os_family()` — 操作系统系列
	* 可能的值是："unix" 和 "windows"。
#### 環境變數
* `env_var(key)` - 取得 key 環境變量的值
```shell
home_dir := env_var('HOME')

test:
  echo "{{home_dir}}"
```

#### 目錄
* `justfile()` - 取得 justfile 路徑
* `justfile_directory()`: 取得 justfile 目錄路徑

#### 字符串處裡
- `replace(s, from, to)` - 将 `s` 中的所有 `from` 替换为 `to`。
- `replace_regex(s, regex, replacement)` - 将 `s` 中所有的 `regex` 替换为 `replacement`。正则表达式由 [Rust `regex` 包](https://docs.rs/regex/latest/regex/) 提供。参见 [语法文档](https://docs.rs/regex/latest/regex/#syntax) 以了解使用示例。
- `trim(s)` - 去掉 `s` 的首尾空格。
- `trim_end(s)` - 去掉 `s` 的尾部空格。
- `trim_end_match(s, pat)` - 删除与 `pat` 匹配的 `s` 的后缀。
- `trim_end_matches(s, pat)` - 反复删除与 `pat` 匹配的 `s` 的后缀。
- `trim_start(s)` - 去掉 `s` 的首部空格。
- `trim_start_match(s, pat)` - 删除与 `pat` 匹配的 `s` 的前缀。
- `trim_start_matches(s, pat)` - 反复删除与 `pat` 匹配的 `s` 的前缀。

### 條件表達
##### if / else
```shell
# 等於
foo := if "2" == "2" { "Good!" } else { "1984" }
# 不等於
foo := if "hello" != "goodbye" { "xyz" } else { "abc" }
# 正則表達式匹配
foo := if "hello" =~ 'hel+o' { "match" } else { "mismatch" }
# 多條件
foo := if "hello" == "goodbye" {
  "xyz"
} else if "a" == "a" {
  "abc"
} else {
  "123"
}
```