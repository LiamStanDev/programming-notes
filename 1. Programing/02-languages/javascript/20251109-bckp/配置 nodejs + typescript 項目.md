#### 全局應用
```shell
pnpm install -g typescript # 可以使用 tsc 工具
pnpm install -g ts-node # 可以使用 ts-node 來運行程序，不需要 tsc -> js -> node
pnpm install -g eslint # 代碼檢查器
```

#### 項目配置
```shell
pnpm init # 會建立 package.json 文件
tsc --init # 建立 tsconfig.json 文件
npx eslint --init # 建立 .eslintrc.js
```

#### 項目運行
```shell
ts-node ./api_server.ts
```