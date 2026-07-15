# Moss 开源项目阅读与介绍

## 一、项目基本信息

Moss 是一个面向 AI Agent 的 Monad 链上交互能力层。它把复杂的 DApp 和协议操作封装成统一的工具流程：`discover -> load -> action -> simulate`。Agent 不需要自己记合约地址、读 ABI、拼 calldata、处理 decimals 或组装清理交易，而是通过 Moss 发现能力、读取参数说明、构建未签名交易计划，并在交给用户或钱包之前先模拟验证。

我阅读的项目是 GitHub 上的 `nishuzumi/moss`。截至 2026-07-15，项目处于 Alpha 阶段，使用 MIT License，主分支是 `main`，GitHub Discussions 未启用。项目 README 中明确提醒：Moss 不是签名器，也不会发送交易；它只负责构建和验证，最终签名仍由用户钱包完成。

## 二、Moss 是什么？它解决了什么问题？

Moss 的核心目标是降低 AI Agent 操作链上协议时的出错风险。一个看似简单的 swap，背后可能包含路由器地址、token decimals、原生币 wrap/unwrap、滑点、授权、退款和 sweep 等细节。如果 Agent 直接根据 ABI 自己拼交易，只要某个细节错了，就可能造成资产损失。

Moss 的做法是把这些细节放进经过维护的协议适配器中。协议适配器把链上协议暴露成 Agent 能理解的能力，例如 `swap`、`wrap`、`transfer` 等。Agent 只传人类可读的参数，例如 `"1"`、`"MON"`、`"USDC"`，Moss 负责解析 token、换算精度、构建交易和声明预期资产流。

更重要的是，Moss 在签名前增加了一道模拟验证关卡。每个写操作会生成一个 Plan，Plan 中包含它声明允许发生的事情，例如最多支出多少资产、至少收到多少资产、是否需要授权。`simulate` 会在最新链上状态中回放这些未签名交易，并把真实效果和声明进行对账。如果出现未声明的资产流出、过量授权、最小收入不满足、交易 revert、Plan 被篡改等 warning，Agent 必须停止。

## 三、为什么 AI Agent 需要 Moss？

AI Agent 适合理解自然语言和组织任务，但不适合凭记忆处理链上交易的所有底层细节。链上操作是高风险场景，错误的合约地址、错误的 decimals、错误的授权额度，都可能直接影响用户资产。

Moss 给 Agent 提供了三层保护：

1. 能力发现：Agent 先通过 `discover` 找到项目已经支持的协议能力，而不是即兴拼接任意合约调用。
2. 参数约束：Agent 通过 `load` 读取 intent、参数语义和风险标签，知道应该传什么、风险在哪里。
3. 签名前验证：Agent 用 `action` 得到未签名 Plan 后，必须调用 `simulate`。只有零 warning，并且模拟效果和用户原始意图一致，才可以把摘要展示给用户。

这种设计把 Agent 的职责变得更清楚：Agent 负责理解用户意图和做最终意图对齐；Moss 负责协议知识、交易构建和机械化安全检查。

## 四、如何使用 Moss 构建 AI Agent？

Moss 提供 MCP server 和 TypeScript library 两种使用方式。对 AI Agent 来说，MCP 是最直接的入口。MCP server 只暴露四个工具：

- `discover`：按 verb、category 或 protocol 发现能力和查询。
- `load`：读取某个能力的调用说明，包括 intent 模板、参数语义、风险标签。
- `action`：执行查询，或构建写操作的未签名 Plan。
- `simulate`：模拟一个或多个 Plan，并返回真实 effects、observations、gas 和 warnings。

一个 Agent 的标准工作流可以这样理解：

1. 先把用户的话转成结构化意图：用户想做什么、付出什么、希望得到什么。
2. 调用 `discover` 找能力，例如用户要 swap，就找 `verb: "swap"`、`category: "dex"`。
3. 调用 `load` 查看参数要求，确认 token、amount、slippage 等该如何传。
4. 调用 `action`，用用户账户地址和人类可读参数构建 Plan。
5. 调用 `simulate`，检查是否有 warning。
6. 如果有 warning，停止并向用户说明；如果没有 warning，还要对比 effects 是否符合用户原始意图。
7. 最后只展示清晰摘要，例如付出多少、收到多少、授权给谁、模拟 gas 等，签名仍由用户钱包决定。

例如我实际使用 Moss 查询过“Monad 上 1 MON 能换多少 USDC”。流程是发现 Kuru 的 `swap` 能力，加载参数说明，用 `action` 构建 `Swap 1 MON into USDC` 的 Plan，再用 `simulate` 回放。模拟返回的 observation 是在 Kuru 上用 1 MON 换得约 0.022631 USDC，并且 warnings 为空。这说明 Moss 不只是文档概念，它能直接把 Agent 的自然语言任务落到可验证的链上计划上。

## 五、项目结构观察

Moss 是一个 pnpm monorepo，目录边界很清楚：

- `packages/core`：纯核心机制，包括 registry、manifest、plan、token、decorator 等，不包含链上地址和 ABI。
- `packages/simulator`：模拟和验证引擎，基于 `debug_traceCall` 提取真实 effects，并与 Plan 的 expects 对账。
- `packages/erc`：ERC-20、ERC-721、WETH9 等通用接口层，提供标准 ABI 和通用 token/NFT 操作。
- `packages/system`：Monad 实例层，包括 MON、WMON、USDC、AUSD 等 token 数据，以及 WMON 适配器。
- `packages/protocols/kuru`：真实协议适配器示例，支持 Kuru 的市价 swap、quote、markets。
- `packages/protocols/_template`：新协议适配器模板。
- `packages/mcp-server`：把 Moss 暴露为 MCP server，提供 `discover/load/action/simulate` 四个工具。
- `examples/simple-flow`：最小运行示例，展示 wrap 和 Kuru swap。
- `examples/agent-swap`：更完整的 Agent swap 示例，在本地 Monad 主网 fork 上演示真实成交流程。
- `docs`：入门、MCP 工具、Agent 使用守则、协议接入指南和 ADR 设计决策。

我觉得这个结构体现了 maintainer 的一个重要原则：底层核心不碰具体链上数据，协议和实例信息放在更上层 package 中。这样可以减少依赖混乱，也方便社区贡献新的协议适配器。

## 六、README 与 Docs 阅读记录

README 主要回答三个问题：Moss 是什么、为什么需要它、怎么开始使用。它强调 Moss 的安全边界：永不签名、永不发送，只构建和验证。README 还列出了当前支持的协议能力，包括 WMON、erc20、erc721 和 Kuru。

Docs 的分工也很明确：

- `docs/getting-started.md` 和中文版适合新手先跑通项目。
- `docs/mcp-tools.md` 是四个 MCP 工具的契约说明。
- `docs/agent-skill.md` 是 Agent 使用 Moss 的安全规则，特别强调 mandatory simulate 和 warning halt rule。
- `docs/protocol-onboarding.md` 面向贡献者，讲如何从模板开始写协议适配器。
- `docs/adr/` 记录架构决策，比如为什么用 `debug_traceCall` 做模拟、为什么 token catalog 不能依赖链上 symbol、为什么一个协议一个 package。

这些文档不只是教程，还在维护项目的工程边界。贡献者可以从文档中知道什么该做、什么不能做，以及 PR 审查会看哪些证据。

## 七、Issues、Pull Requests、Discussions 观察

GitHub Issues 显示项目当前有不少开放任务。很多 Issue 都围绕协议适配器展开，例如 PancakeSwap、Uniswap v4、Clober、Aave、Morpho、Euler、Pendle、FastLane 等。这说明 Moss 的核心扩展路径是“不断增加协议适配器”，让 Agent 可调用的能力目录变大。

我比较感兴趣的是 Issue #6：`Adapter: PancakeSwap swap`。它被标记为 `good first issue`、`adapter`、`dex`、`difficulty:starter`。Issue 描述很具体：需要实现一个 AMM router adapter，提供 `swap` capability 和 `quote` query，先确认 Monad 上部署的是 PancakeSwap 哪个版本，再记录在 package header 中。它还直接指向 `docs/protocol-onboarding.md`、`packages/protocols/_template`、WMON 参考适配器和 Kuru 适配器。这个 Issue 很适合新人，因为 maintainer 已经把背景、起步路径、硬性规则和 Definition of Done 写清楚了。

Pull Requests 方面，开放 PR 包括 PancakeSwap V2/V3 适配器、ERC-1155 接口与 effects reconciliation、TypeScript 6 迁移、ABI 获取脚本、文档 FAQ、繁体中文文档等。可以看出项目维护方向有三类：扩大协议覆盖、增强接口层和模拟能力、改善贡献者体验。

Discussions 当前未启用，所以项目讨论主要发生在 Issues 和 PR 中。

## 八、Maintainer 如何组织和管理项目

这个项目的管理方式给我的印象是“规则前置”。Maintainer 不只是等 PR 来了再口头解释，而是提前把规则写进 README、CONTRIBUTING、PR template、Issue template、CI 和 ADR 中。

具体表现包括：

- 用 Issue labels 标注任务类型和难度，例如 `good first issue`、`adapter`、`difficulty:starter`、`needs-design`。
- 用 PR template 要求说明 What & why、变更类型、测试证据和 simulate effects summary。
- 用 CI 强制 lint、build、typecheck、test。
- 对协议适配器设置明确 Definition of Done：ABI 来源、地址验证、能力声明、expects、observations、discover/load 测试、主网模拟零 warning。
- 用 ADR 保存设计决策，避免后续贡献者反复争论同一类架构问题。

这种管理方式对开源项目很重要，因为它降低了沟通成本，也让新人知道从哪里开始、做到什么程度才算完成。

## 九、我的发现

我最大的发现是：Moss 的价值不只是“帮 Agent 调链上协议”，而是把链上交互变成一套可发现、可解释、可模拟、可审查的能力系统。它没有让 Agent 变成“更大胆的交易执行者”，反而给 Agent 加上了边界：不能乱拼交易，不能跳过模拟，不能忽略 warning，不能代替用户签名。

从开源项目阅读角度看，Moss 也展示了一个维护者如何把项目意图落到工程结构中：README 负责讲清楚定位，docs 负责教会使用和贡献，Issues 负责拆任务，PR 模板负责收集证据，CI 负责守住质量底线，ADR 负责记录长期设计取舍。

如果我要继续深入这个项目，我会优先研究 PancakeSwap adapter 相关 Issue 和 PR。它既贴近 Moss 的核心扩展方向，又有清晰的新手路径，适合通过实现一个真实协议适配器来理解 Moss 的完整贡献流程。

## 参考

- GitHub Repository: https://github.com/nishuzumi/moss
- README: https://github.com/nishuzumi/moss/blob/main/README.md
- MCP tools: https://github.com/nishuzumi/moss/blob/main/docs/mcp-tools.md
- Agent skill guide: https://github.com/nishuzumi/moss/blob/main/docs/agent-skill.md
- Protocol onboarding: https://github.com/nishuzumi/moss/blob/main/docs/protocol-onboarding.md
- Issue #6: https://github.com/nishuzumi/moss/issues/6
- Issue #28: https://github.com/nishuzumi/moss/issues/28
