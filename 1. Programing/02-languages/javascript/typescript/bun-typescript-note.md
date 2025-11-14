# Bun.js 標準庫 API 完整整理

## 目錄
1. Bun.js 簡介
2. Bun 全域物件與 API
3. 標準庫 API 詳解
   - 3.1 HTTP 伺服器（Bun.serve）
   - 3.2 檔案系統（fs）
   - 3.3 網路請求（fetch）
   - 3.4 WebSocket
   - 3.5 Process 與環境變數
   - 3.6 Buffer
   - 3.7 其他常用 API
4. 常見實戰範例
5. 參考資源

---

## 1. Bun.js 簡介
Bun.js 是一個現代 JavaScript 執行環境，內建 Bundler、Transpiler、Task Runner 及 npm client，主打高效能與 Node.js 相容性，並提供豐富的標準庫 API。

---

## 2. Bun 全域物件與 API
Bun 提供全域 `Bun` 物件，內含多種高效能 API，並盡量實作標準 Web API（如 fetch、URL、WebSocket）。

---

## 3. 標準庫 API 詳解

### 3.1 HTTP 伺服器（Bun.serve）
Bun 內建高效能 HTTP 伺服器，支援 TypeScript。
```typescript
import { serve } from "bun";
serve({
  fetch(req) {
    return new Response("Hello Bun!");
  },
  port: 3000,
});
```
- 支援路由、靜態檔案、Cookie 操作
- `Request`/`Response` 皆為標準 Web API

### 3.2 檔案系統（fs）
Bun 提供同步與非同步檔案操作 API，與 Node.js 類似。
```typescript
// 讀取檔案
const data = await Bun.file("test.txt").text();
// 寫入檔案
await Bun.write("output.txt", "內容");
```
- 支援 text、arrayBuffer、stream 等格式
- 亦可使用 Node.js 的 fs 模組

### 3.3 網路請求（fetch）
Bun 原生支援標準 fetch API。
```typescript
const res = await fetch("https://api.example.com");
const json = await res.json();
```
- 與瀏覽器 fetch 行為一致
- 支援 Request/Response 物件

### 3.4 WebSocket
Bun 支援 WebSocket API，適合即時通訊應用。
```typescript
const ws = new WebSocket("ws://localhost:3000");
ws.onopen = () => ws.send("Hello!");
ws.onmessage = (event) => console.log(event.data);
```

### 3.5 Process 與環境變數
- `process.env`：存取環境變數
- `process.argv`：取得啟動參數
- `process.exit()`：結束程序
- 兼容 Node.js process API

### 3.6 Buffer
Bun 支援 Node.js 的 Buffer API，用於二進位資料處理。
```typescript
const buf = Buffer.from("hello");
console.log(buf.toString("hex"));
```

### 3.7 其他常用 API
- `Bun.spawn`：啟動子程序
- `Bun.sleep`：非同步延遲
- `Bun.hash`：雜湊運算
- `Bun.inspect`：物件格式化輸出
- `Bun.readableStreamToArrayBuffer`：Stream 轉換

---

## 4. 常見實戰範例

### 建立靜態檔案伺服器
```typescript
import { serve } from "bun";
serve({
  fetch(req) {
    return new Response(Bun.file("index.html"));
  },
  port: 8080,
});
```

### 使用 Bun.spawn 執行外部命令
```typescript
const proc = Bun.spawn(["ls", "-la"]);
await proc.exited;
console.log(await new Response(proc.stdout).text());
```

### 使用 Buffer 處理檔案
```typescript
const buf = await Bun.file("image.png").arrayBuffer();
const base64 = Buffer.from(buf).toString("base64");
```

---

## 5. Node.js 使用者遷移注意事項

### 5.1 與 Node.js 的主要差異與注意事項

- **模組系統**：Bun 支援 ES Modules 與 CommonJS，但部分 Node.js 模組解析行為可能不同，建議優先使用 ES Modules。
- **API 相容性**：Bun 力求相容 Node.js 內建 API（如 fs、process、Buffer），但部分 API 尚未完全覆蓋，或行為略有差異，需參考官方相容性列表。
- **全域物件**：Bun 提供全域 Bun 物件，Node.js 沒有此物件。
- **標準 Web API**：Bun 原生支援 fetch、Request、Response、WebSocket 等 Web 標準 API，Node.js 需額外安裝 polyfill。
- **HTTP 伺服器**：Bun 主要使用 Bun.serve，API 與 Node.js 的 http.createServer 不同，請參考 Bun 官方文件調整用法。
- **套件管理**：Bun 內建 bun install/bun add，與 npm/yarn/pnpm 指令略有不同。
- **執行 TypeScript**：Bun 可直接執行 .ts 檔案，Node.js 需透過 ts-node 或轉譯。
- **測試工具**：Bun 內建測試框架，語法與 Jest 類似但不完全相同。
- **部分 Node.js API 尚未支援**：如 cluster、child_process、worker_threads 等，請查閱官方支援狀態。
- **路徑處理**：部分 path 處理細節與 Node.js 有差異，需測試驗證。

### 5.2 參考官方相容性列表
- [Bun Node.js 相容性](https://bun.sh/docs/runtime/nodejs-apis)

---

## 6. 參考資源
- [Bun 官方 API 參考](https://bun.sh/reference)
- [Bun HTTP 伺服器](https://bun.sh/docs/api/http)
- [Bun Web APIs](https://bun.sh/docs/runtime/web-apis)
- [Bun 中文文檔 API](https://www.bunjs.cn/docs/runtime/bun-apis)
