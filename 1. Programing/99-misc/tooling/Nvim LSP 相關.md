##### vim.lsp.commands
用來註冊非 Lsp Specification 的 Handler 如
```lua
-- 這是 rust analyzer 提供的
vim.lsp.commands['rust-analyzer.debugSingle'] = function(command)
	local overrides = require('rustaceanvim.overrides')
	-- ....
	rt_dap.start(args)
end
```

##### vim.lsp.config
```lua
```

##### vim.lsp.enable
```lua
```

##### vim.lsp.get_clients

##### Client:request
向 server 發送請求
```lua
local client = assert(vim.lsp.get_clients()[1])
client:request('textDocument/definition')
```