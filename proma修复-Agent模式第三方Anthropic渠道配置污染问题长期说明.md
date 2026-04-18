# Proma 修复：Agent 模式第三方 Anthropic 渠道配置污染问题长期说明

> 更新时间：`2026-04-18`
>
> 这份文档用于长期记录一个容易被误判的问题：Proma 在 `Agent` 模式接第三方 `Anthropic` 格式渠道时，报错表面上像“模型不存在 / 无权限”，但真实问题往往出在 **配置来源污染**。
>
> **重要补充：这类问题不只发生在发行版，开源版源码链路也有同类风险。**

## 一句话结论

这次问题**不应被简单归因为 SDK 版本兼容错误**，也**不应先归因为模型名写错**。

更准确的结论是：

- **历史共性问题**：Agent 运行链路允许 Claude Agent SDK 同时读取 `user + project` 两级设置；
- **污染源**：用户级 SDK 配置目录中的旧 `ANTHROPIC_*` 设置，可能覆盖当前 Agent 会话真正选择的渠道、Base URL、模型和认证参数；
- **发行版表现**：如果实际运行的 `main.cjs` 仍是 `settingSources: ["user", "project"]`，这条污染链路就仍然存在；
- **开源版当前状态**：源码主线已经对 `process.env` 型 `ANTHROPIC_*` 污染做了多层加固，但 `settingSources` 这一层在当前 `main` 源码里仍然是放开的；
- 因此，**开源版和发行版都和这个问题有关，只是问题落点不完全一样**。

## 需要区分的三个关键位置

| 角色 | 绝对路径 | 当前意义 |
|---|---|---|
| 开源版源码问题入口 | `D:\Proma-Source\apps\electron\src\main\lib\adapters\claude-agent-adapter.ts` | 当前 `main` 分支里，`settingSources` 仍是 `['user', 'project']` |
| 发行版实际运行文件 | `C:\Users\yzjiang\AppData\Local\Programs\Proma\resources\app\dist\main.cjs` | 最终用户真正运行的是它；是否还会被 user settings 污染，要以这里的实际内容为准 |
| 用户级污染源文件 | `C:\Users\yzjiang\.proma\sdk-config\settings.json` | 如果残留旧的 `ANTHROPIC_*`，一旦被 SDK 读到，就可能污染当前 Agent 会话 |

## 为什么说开源版也有类似问题

因为当前开源版 `main` 分支源码里，`settingSources` 仍然允许读 `user + project`。

对应位置：

- `D:\Proma-Source\apps\electron\src\main\lib\adapters\claude-agent-adapter.ts:442`

当前源码实际值：

```ts
settingSources: ['user', 'project']
```

这意味着：

- 只要 SDK 用户级配置目录中还有旧的 `ANTHROPIC_*`；
- 当前 Agent 运行链路就仍然有机会把这些旧值读回来；
- 从而污染当前界面显式选择的 Agent 渠道配置。

所以这不是“只有发行版有问题、开源版完全没问题”。

更准确的说法是：

- **开源版源码层面**：当前仍保留 `user + project` 的读取范围；
- **发行版运行层面**：要看实际 bundle 是否也仍保留这个值；
- **两者共享同一类历史根因：用户级 SDK 配置可能覆盖当前 Agent 会话配置。**

## 为什么说发行版也有这个问题

因为最终用户运行的不是源码，而是打包后的主进程 bundle：

- `C:\Users\yzjiang\AppData\Local\Programs\Proma\resources\app\dist\main.cjs`

如果这个文件里仍然包含：

```js
settingSources: ["user", "project"]
```

那么发行版就仍然保留同一条污染链路。

也就是说：

- **开源版决定了问题会不会被继续带入构建产物；**
- **发行版决定了最终用户机器上是否真的还在复现。**

## 真实的根因链条

这类问题的根因链条如下：

1. Agent 集成链路允许 Claude Agent SDK 同时读取：
   - 用户级设置（`user`）
   - 项目级设置（`project`）

2. 用户级 SDK 配置目录中如果残留旧的 `ANTHROPIC_*`，例如：
   - `ANTHROPIC_BASE_URL`
   - `ANTHROPIC_API_KEY`
   - `ANTHROPIC_CUSTOM_MODEL_OPTION`
   - `ANTHROPIC_DEFAULT_OPUS_MODEL`
   - `ANTHROPIC_DEFAULT_SONNET_MODEL`
   - `ANTHROPIC_DEFAULT_HAIKU_MODEL`

3. 这些旧值一旦参与配置叠加，就可能覆盖当前会话真正选中的：
   - Base URL
   - API Key
   - 模型
   - 其他 Anthropic 相关运行参数

4. 最终 SDK 实际拿到的是一组被污染后的 `base URL / model / auth` 组合。

5. 用户在 UI 上看到的报错往往是：

```text
There's an issue with the selected model (...)
It may not exist or you may not have access to it.
```

但本质不是“模型一定不存在”，而是“当前会话配置被污染了”。

## 这为什么不应先归因为 SDK 版本兼容问题

因为从现有排查结果看：

- 第三方 Anthropic Messages API 链路并没有被证明整体不可用；
- 报错会随着配置来源的收窄而消失；
- 一旦把 `settingSources` 从 `['user', 'project']` 收窄为 `['project']`，同类报错就能恢复；
- 这更像是 **Proma 对 Claude Agent SDK 的配置接入策略** 问题，而不是纯粹的版本 API 不兼容。

所以更准确的结论是：

**这首先是配置来源设计问题，其次才需要结合 SDK 版本变化做额外加固。**

## 开源版当前并不是“完全没防护”

虽然开源版当前 `settingSources` 仍然是 `['user', 'project']`，但在 `@anthropic-ai/claude-agent-sdk 0.2.111` 下，源码已经针对 **环境变量污染** 做了多层加固。

### 1. 应用启动时先清理 `process.env` 中的 `ANTHROPIC_*`

文件：

- `D:\Proma-Source\apps\electron\src\main\index.ts`

作用：

- 防止本地 shell / 父进程环境中的旧 `ANTHROPIC_*` 一启动就泄漏进 Agent 运行链路。

### 2. `buildSdkEnv()` 再次过滤 `ANTHROPIC_*`

文件：

- `D:\Proma-Source\apps\electron\src\main\lib\agent-orchestrator.ts:423`

作用：

- 从 `process.env` 继承系统变量时，再次剔除所有 `ANTHROPIC_*`；
- 避免开发环境、本地终端或其他进程的 Anthropic 变量干扰当前渠道配置。

### 3. 显式注入当前会话真正需要的 Anthropic 变量

同文件关键位置：

- `D:\Proma-Source\apps\electron\src\main\lib\agent-orchestrator.ts:442`
- `D:\Proma-Source\apps\electron\src\main\lib\agent-orchestrator.ts:451`
- `D:\Proma-Source\apps\electron\src\main\lib\agent-orchestrator.ts:456`

当前策略是：

- 显式写入 `ANTHROPIC_API_KEY`
- 按需写入规范化后的 `ANTHROPIC_BASE_URL`
- 显式写入 `CLAUDE_CONFIG_DIR`

### 4. 针对 SDK 0.2.111 的 `options.env` “叠加语义”做空字符串覆盖

同文件关键位置：

- `D:\Proma-Source\apps\electron\src\main\lib\agent-orchestrator.ts:487`
- `D:\Proma-Source\apps\electron\src\main\lib\agent-orchestrator.ts:491`

背景是：

- SDK `0.2.111` 起，`options.env` 不再是“替换”，而是“叠加到 `process.env` 上”；
- 因此如果 shell 中还残留 `ANTHROPIC_MODEL`、`ANTHROPIC_CUSTOM_HEADERS` 等，单纯过滤还不够；
- 现在源码会把 `sdkEnv` 未显式管理的 `ANTHROPIC_*` 设为空字符串，强制覆盖叠加回流。

### 5. 启动 Agent 前再同步一次 `process.env`

同文件关键位置：

- `D:\Proma-Source\apps\electron\src\main\lib\agent-orchestrator.ts:829`
- `D:\Proma-Source\apps\electron\src\main\lib\agent-orchestrator.ts:830`
- `D:\Proma-Source\apps\electron\src\main\lib\agent-orchestrator.ts:831`
- `D:\Proma-Source\apps\electron\src\main\lib\agent-orchestrator.ts:838`

作用：

- 保证当前进程环境与最终传给 SDK 的 `sdkEnv` 尽量一致；
- 防止 in-process 代码路径绕开 `options.env` 直接读取旧值。

## 所以开源版当前的准确状态是什么

应该区分成两层：

| 层面 | 当前状态 |
|---|---|
| `process.env` 型 `ANTHROPIC_*` 污染 | **已有较强加固** |
| `settingSources` 导致的用户级 settings 污染 | **当前 `main` 仍未彻底切断** |

也就是说，开源版现在不是“完全裸奔”，但也不是“这个问题已经彻底不存在”。

## 历史上开源版其实也做过“只读 project”的修复

从既有排查记录看，开源版历史上确实做过把：

```ts
settingSources: ['user', 'project']
```

收窄为：

```ts
settingSources: ['project']
```

的修复，目的就是阻断用户级 settings 污染当前 Agent 会话。

但当前 `main` 源码没有保留这一收窄结果，而是仍然开放 `user + project`。

所以：

- **历史上开源版处理过这个问题；**
- **当前主线源码并不能被简单说成已经彻底解决。**

## 发行版与开源版的关系应该怎么表述

最准确的表述应该是：

### 开源版

- 问题入口主要看源码：
  - `D:\Proma-Source\apps\electron\src\main\lib\adapters\claude-agent-adapter.ts`
- 当前主线源码里，`settingSources` 仍允许读取 `user + project`；
- 同时源码又新增了针对 `process.env` 型 Anthropic 污染的多层加固。

### 发行版

- 问题入口主要看实际 bundle：
  - `C:\Users\yzjiang\AppData\Local\Programs\Proma\resources\app\dist\main.cjs`
- 如果这个 bundle 仍是 `user + project`，则最终用户环境里同类问题仍可能复现；
- 如果这个 bundle 已被补成 `project`，则“用户级 settings 覆盖当前 Agent 会话”这条链路会被切断。

### 用户级污染源

- 两者共用的污染源检查点仍然是：
  - `C:\Users\yzjiang\.proma\sdk-config\settings.json`

## 最小修复思路应该怎么理解

### 对发行版

如果确认实际运行的 bundle 仍然是：

```js
settingSources: ["user", "project"]
```

那么最小补丁仍然是收窄为：

```js
settingSources: ["project"]
```

补丁文件：

- `C:\Users\yzjiang\AppData\Local\Programs\Proma\resources\app\dist\main.cjs`

### 对开源版

如果目标是从源码层面彻底堵住“用户级 settings 污染当前 Agent 会话”这条链路，核心位置仍然是：

- `D:\Proma-Source\apps\electron\src\main\lib\adapters\claude-agent-adapter.ts`

同时必须保留当前已经存在的 env 加固逻辑，尤其是：

- `D:\Proma-Source\apps\electron\src\main\index.ts`
- `D:\Proma-Source\apps\electron\src\main\lib\agent-orchestrator.ts`

原因是：

- `settingSources` 修的是 **user settings 污染**；
- `buildSdkEnv()` / `process.env` 清理修的是 **环境变量叠加污染**；
- 这两层不是一回事，不能只留一层。

## 为什么开发者一开始会用 `user + project`

`user + project` 是一种很常见的分层配置设计：

- `user`：全局默认设置；
- `project`：当前项目或当前会话的局部覆盖。

它的出发点通常是：

1. 减少重复配置；
2. 允许保留全局 API Key / 默认端点；
3. 再由具体项目进行局部覆盖。

但放到 Proma 的 Agent 场景里就有一个问题：

- 当前渠道、Base URL、模型本来已经由 UI 显式选中了；
- 这些当前运行时参数本来应该是最权威的；
- 再把用户级旧 settings 混进来，只会让当前会话的配置来源变得不透明。

所以更准确的结论不是“`user + project` 天生错误”，而是：

**在 Proma 的 Agent 集成链路里，`user + project` 不适合作为默认行为。**

## 用户体感上会有什么差异

大多数情况下，修复前后**不会有明显 UI 差异**。

主要差异只有一个：

- 修复前：Agent 模式可能莫名其妙读错配置，表现成 selected model 报错；
- 修复后：Agent 模式不再被用户级旧配置或 shell 环境旧值污染。

也就是说：

- 不是 UI 改版；
- 不是功能增加；
- 不是渠道结构重构；
- 而是把错误的配置来源收窄或隔离回去。

## 以后每次更新后应该怎么查

### 如果你在查发行版

优先检查：

1. `C:\Users\yzjiang\AppData\Local\Programs\Proma\resources\app\dist\main.cjs`
2. `C:\Users\yzjiang\.proma\sdk-config\settings.json`

重点看：

- `main.cjs` 里是不是仍然有 `settingSources: ["user", "project"]`
- 用户级 SDK 配置中是否仍残留旧 `ANTHROPIC_*`

### 如果你在查开源版源码

优先检查：

1. `D:\Proma-Source\apps\electron\src\main\lib\adapters\claude-agent-adapter.ts`
2. `D:\Proma-Source\apps\electron\src\main\lib\agent-orchestrator.ts`
3. `D:\Proma-Source\apps\electron\src\main\index.ts`
4. `D:\Proma-Source\apps\electron\package.json`

重点看：

- `settingSources` 当前是否仍为 `['user', 'project']`
- `buildSdkEnv()` 是否还保留 `ANTHROPIC_*` 过滤与空字符串覆盖
- 当前 SDK 版本是否仍为 `0.2.111`

## 最后结论

以后如果再次看到下面这种现象：

- Chat 模式正常；
- Agent 模式接第三方 Anthropic 渠道失败；
- 页面报 selected model 错误；

请优先想到：

**这很可能不是模型本身的问题，而是 Agent 运行时把用户级旧配置或环境变量旧值重新带了回来。**

最应该优先检查的几个位置是：

1. `D:\Proma-Source\apps\electron\src\main\lib\adapters\claude-agent-adapter.ts`
2. `D:\Proma-Source\apps\electron\src\main\lib\agent-orchestrator.ts`
3. `C:\Users\yzjiang\AppData\Local\Programs\Proma\resources\app\dist\main.cjs`
4. `C:\Users\yzjiang\.proma\sdk-config\settings.json`

而对这个问题最关键的一句话总结仍然是：

**这不是只有发行版才有的问题；开源版源码和发行版运行产物都与它有关。开源版当前主要是“源码仍放开 user settings、但已加强 env 隔离”，发行版当前主要看“实际运行 bundle 是否仍保留 `user + project`”。**
