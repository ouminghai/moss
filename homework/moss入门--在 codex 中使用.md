# moss入门--在 codex 中使用

本文根据本次实操过程整理，目标是让新手能理解 Moss 的基本工作流，并能把 Moss 接入 Codex 使用。

## 1. Moss 入门指南

### 1.1 Moss 是什么

Moss 是一个面向 Monad 链上操作的工具框架。它不会替用户签名，也不会发送交易，而是帮助 agent 或开发者完成以下事情：

- 发现当前支持哪些链上能力。
- 加载某个能力的调用说明。
- 根据用户意图构建未签名的 Plan。
- 在真实链上状态中模拟 Plan，并检查实际效果是否符合预期。

也就是说，Moss 的核心价值不是“直接交易”，而是“在交易交给钱包前，先验证它到底会做什么”。

### 1.2 标准流程：discover → load → action → simulate

Moss 的 MCP server 暴露四个核心工具：

1. `discover`

   查找当前支持的能力。例如想换币，可以按 `verb: swap`、`category: dex` 搜索。

   示例结果：

   ```json
   {
     "protocol": "kuru",
     "method": "swap",
     "kind": "capability",
     "verb": "swap",
     "category": "dex",
     "tags": ["clob", "orderbook"]
   }
   ```

2. `load`

   加载某个能力的调用说明，包括参数、风险标签、意图模板等。

   例如 Kuru swap 需要：

   - `tokenIn`：输入代币，如 `MON`
   - `tokenOut`：输出代币，如 `USDC`
   - `amount`：人类可读金额，如 `"1"`，不要提前换算成 wei
   - `slippage`：滑点，单位是 bps，`100` 表示 1%

3. `action`

   根据参数构建 Plan。Plan 不是已发送交易，而是一组未签名交易和可验证预期。

   Plan 中比较重要的字段：

   - `intent`：这笔操作声称要做什么
   - `txs`：未签名交易
   - `expects`：最多流出多少、至少流入多少、是否授权
   - `declaredRisk`：声明风险，例如 `fundOut`、`approval`、`priceImpact`
   - `planHash`：Plan 完整性校验

4. `simulate`

   在 Monad 真实链上状态中模拟 Plan，提取实际资产流动、授权、收款方和协议回执。

   重点规则：

   - 只要有 warning，就停止，不要把交易交给钱包签名。
   - 没有 warning，也要确认模拟结果符合用户真实意图。
   - Moss 模拟不会消耗真实资金，也不需要私钥。

### 1.3 本地运行 Moss 示例

安装依赖并构建：

```bash
pnpm install
pnpm build
```

先跳过主网 e2e，确认离线测试没问题：

```bash
MOSS_SKIP_E2E=1 pnpm test
```

运行官方入门示例，把 1.5 MON 包装成 WMON：

```bash
MOSS_RPC_URL=https://rpc.monad.xyz pnpm --filter @themoss/example-simple-flow wrap
```

本次实操中，该命令已经成功跑通，结果显示：

- `discover` 找到了 `wmon.wrap`
- `load` 加载了 wrap 参数说明
- `action` 构建了 Plan
- `simulate` 返回 `warnings: []`

这说明 Moss 的主网模拟链路可以正常工作。

### 1.4 查询 1 MON 能换多少 USDC 的正确方式

在 Moss 中不要直接猜价格，而是让工具按完整流程检查：

1. `discover({ verb: "swap", category: "dex" })`
2. `load([{ protocol: "kuru", method: "swap" }])`
3. `action("kuru", "swap", account, { tokenIn: "MON", tokenOut: "USDC", amount: "1" })`
4. `simulate([plan])`

最后看 `simulate` 的：

- `effects.assetsOut`
- `effects.assetsIn`
- `observations`
- `warnings`

如果 `warnings` 为空，并且 `assetsIn` 显示 USDC 流入，就可以用这个模拟结果回答“1 MON 能换多少 USDC”。

## 2. 把 Moss 接入 Codex 使用的教程

### 2.1 Codex 需要的 MCP 配置

在项目根目录使用 `.mcp.json` 配置 Moss：

```json
{
  "mcpServers": {
    "moss": {
      "command": "node",
      "args": ["packages/mcp-server/dist/cli.js"],
      "env": {
        "MOSS_RPC_URL": "https://rpc.monad.xyz",
        "MOSS_CHAIN_ID": "143"
      }
    }
  }
}
```

这份配置的含义：

- `command`: 用 Node 启动 Moss MCP server。
- `args`: 指向已经构建好的 Moss MCP CLI。
- `MOSS_RPC_URL`: 使用 Monad 主网 RPC。
- `MOSS_CHAIN_ID`: Monad 主网 chain id，值为 `143`。

### 2.2 为什么不要默认用 127.0.0.1:8545

本次实操中，一开始 `.mcp.json` 指向：

```json
{ "MOSS_RPC_URL": "http://127.0.0.1:8545" }
```

这会要求本机已经启动 Monad 版 anvil fork。当前机器没有 `anvil`，所以 Moss 在 `action` 阶段读取 Kuru 市场参数时失败：

```text
URL: http://127.0.0.1:8545/
Details: fetch failed
```

如果只是想在 Codex 中做主网模拟和报价，应该使用：

```text
https://rpc.monad.xyz
```

只有在运行 `examples/agent-swap` 那种本地 fork 交易示例时，才需要 `127.0.0.1:8545`。

### 2.3 构建 Moss MCP server

Codex 启动 MCP server 前，需要确保 `packages/mcp-server/dist/cli.js` 存在。

执行：

```bash
pnpm -r --filter @themoss/mcp-server build
```

检查是否能启动：

```bash
node packages/mcp-server/dist/cli.js
```

正常时会看到类似输出：

```text
moss-mcp: 12 capabilities/queries across 4 protocols on chain 143 (https://rpc.monad.xyz)
```

这说明 Moss MCP server 已经能连接 Monad 主网 RPC。

### 2.4 在 Codex 中使用 Moss

配置完成后，重启当前 Codex 任务，或新开一个 Codex 任务进入本仓库。

然后可以直接向 Codex 提问：

```text
在 Monad 上 1 MON 能换多少 USDC？请使用 Moss，按 discover → load → action → simulate 流程检查。
```

Codex 应该会调用 Moss 工具：

- `mcp__moss.discover`
- `mcp__moss.load`
- `mcp__moss.action`
- `mcp__moss.simulate`

### 2.5 重要排错经验

#### 问题 1：Moss 仍然访问 127.0.0.1:8545

现象：

```text
URL: http://127.0.0.1:8545/
Details: fetch failed
```

原因：

Codex 当前任务里的 MCP 进程是在修改 `.mcp.json` 之前启动的。MCP server 不会在当前任务中自动热重载配置。

解决：

- 新开一个 Codex 任务。
- 或关闭再重新打开当前任务。
- 确认 `.mcp.json` 已经指向 `https://rpc.monad.xyz`。

#### 问题 2：MCP server 启动失败

可能原因：

- 还没有执行 `pnpm build`
- `packages/mcp-server/dist/cli.js` 不存在
- Node 版本低于 22

解决：

```bash
node --version
pnpm install
pnpm build
```

#### 问题 3：simulate 失败或 RPC 不支持 debug_traceCall

Moss 的模拟依赖 `debug_traceCall`。如果 RPC 不支持 debug namespace，simulate 会失败。

推荐使用：

```text
https://rpc.monad.xyz
```

文档中也提到可用的 RPC 包括：

- `https://rpc.monad.xyz`
- `https://rpc4.monad.xyz`
- `https://rpc-mainnet.monadinfra.com`
- `https://monad-rpc.huginn.tech`

### 2.6 最小成功检查清单

完成接入后，按这个顺序检查：

1. `.mcp.json` 指向 `https://rpc.monad.xyz`
2. `pnpm build` 成功
3. `node packages/mcp-server/dist/cli.js` 能显示 chain 143
4. 新开或重启 Codex 任务
5. 在 Codex 中能看到 `mcp__moss` 工具
6. 用 Moss 跑 `discover → load → action → simulate`
7. `simulate` 返回 `warnings: []`

只要这些都满足，就说明 Moss 已经可以在 Codex 中正常使用。

