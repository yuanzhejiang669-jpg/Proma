# Proma 修复：Agent 模式第三方 Anthropic 渠道配置污染问题长期说明

> 更新时间：`2026-04-18`
>
> 这份文档用于把“Proma 发行版 Agent 模式接第三方 Anthropic 格式渠道时，为什么会出现 selected model 报错、真正根因在哪里、以后升级后该怎么复查”长期固定在仓库里，方便每次更新后快速回看。

## 一句话结论

这次问题**不是 SDK 版本兼容错误**，也**不是模型名本身一定写错**。

真正根因是：

- 发行版运行时文件 `main.cjs` 仍然允许 Claude Agent SDK 同时读取 `user + project` 两级配置；
- 用户级 `C:\Users\yzjiang\.proma\sdk-config\settings.json` 中残留了旧的 `ANTHROPIC_*` 配置；
- 这些旧配置污染了当前 Agent 会话真正选择的渠道、Base URL、模型和认证参数；
- 最终在 UI 上表现为：

```text
There's an issue with the selected model (...)
It may not exist or you may not have access to it.
```

因此，这次问题的本质是：**配置来源被污染**，不是“模型一定不存在”。

## 两个关键文件

| 角色 | 绝对路径 | 作用 | 是否是根因文件 |
|---|---|---|---|
| 程序逻辑文件 | `C:\Users\yzjiang\AppData\Local\Programs\Proma\resources\app\dist\main.cjs` | 决定 Agent 运行时从哪里读取 Claude Agent SDK 配置 | **是** |
| 污染数据文件 | `C:\Users\yzjiang\.proma\sdk-config\settings.json` | 存放旧的 `ANTHROPIC_*` 残留配置，一旦被读取就会污染当前会话 | 否，它是污染源 |

最准确的说法是：

- `main.cjs` 是**根因逻辑文件**；
- `settings.json` 是**污染源数据文件**。

## 现象

在以下条件下更容易复现：

- 使用 Proma **发行版**；
- 进入 **Agent 模式**；
- 第三方渠道格式选择 **Anthropic**；
- 使用类似 `http://127.0.0.1:8317/v1` 这样的第三方接口；
- 模型填写为类似 `gpt-5.4(xhigh)` 这样的第三方映射模型。

典型报错为：

```text
There's an issue with the selected model (gpt-5.4(xhigh)).
It may not exist or you may not have access to it.
Run --model to pick a different model.
```

## 容易被误判成什么

这个问题很容易被误判为：

1. 第三方渠道服务挂了；
2. UI 对 `/v1` 到 `/v1/messages` 的映射坏了；
3. `gpt-5.4(xhigh)` 这个模型名本身不合法；
4. Claude Agent SDK 版本升级后出现了接口兼容问题；
5. Chat 模式能用，因此 Agent 理论上也不该出问题。

这些都不是本次问题的核心结论。

## 为什么判断它不是 SDK 版本兼容问题

当前已有排查结论显示：

- 第三方 Anthropic Messages API 链路本身并没有被证明彻底不可用；
- 报错与“当前会话实际拿到的最终配置”密切相关；
- 一旦把设置来源从 `['user', 'project']` 收窄为 `['project']`，同类报错就能恢复；
- 这说明问题主要出在**配置叠加顺序和来源范围**，而不是 SDK 与第三方服务的协议完全不兼容。

所以更准确的结论是：

**这是 Proma 在 Agent 集成链路中的配置来源设计问题，而不是一个简单的 SDK 版本兼容问题。**

## 真正的根因链条

问题链条如下：

1. 发行版 `main.cjs` 中的 Agent 运行时逻辑仍使用：

```js
settingSources: ["user", "project"]
```

2. 这意味着 Claude Agent SDK 会同时读取：
   - 用户级配置；
   - 项目级配置。

3. 用户级配置文件 `C:\Users\yzjiang\.proma\sdk-config\settings.json` 中如果残留了旧的 `ANTHROPIC_*`，例如：
   - `ANTHROPIC_BASE_URL`
   - `ANTHROPIC_API_KEY`
   - `ANTHROPIC_CUSTOM_MODEL_OPTION`
   - `ANTHROPIC_DEFAULT_OPUS_MODEL`
   - `ANTHROPIC_DEFAULT_SONNET_MODEL`
   - `ANTHROPIC_DEFAULT_HAIKU_MODEL`

4. 那么当前界面里实际选中的 Agent 渠道配置，就可能被这些旧值覆盖。

5. 最终 SDK 收到的是一组被污染后的 `base URL / model / auth` 组合，于是落入“selected model 不存在或无权限”的错误分支。

## 为什么 `main.cjs` 会被视为真正的问题文件

因为它不是普通数据文件，而是**发行版实际运行的主进程 bundle**。

也就是说：

- 它决定 Agent 启动时怎么构造 SDK 配置；
- 它决定是否允许读取用户级配置；
- 它决定 `settings.json` 里的污染项有没有机会进入当前会话。

因此：

- 如果 `main.cjs` 还是 `['user', 'project']`，污染就有机会再次发生；
- 如果 `main.cjs` 改成 `['project']`，即使用户级 `settings.json` 还残留旧值，这条 Agent 运行链路也不会再读进去。

## 实际修复方式

这次问题采用的是**最小补丁**方案。

### 修改前

```js
settingSources: ["user", "project"]
```

### 修改后

```js
settingSources: ["project"]
```

### 补丁目标

只做一件事：

- **阻断用户级 SDK 配置继续污染当前 Agent 会话**。

### 补丁文件

```text
C:\Users\yzjiang\AppData\Local\Programs\Proma\resources\app\dist\main.cjs
```

### 配套清理

为降低后续误触发概率，还建议同时检查并清理：

```text
C:\Users\yzjiang\.proma\sdk-config\settings.json
```

里面残留的旧 `ANTHROPIC_*` 配置。

## 修复前后，用户体感有什么变化

大多数情况下，**用户几乎不会感知到 UI 层面的差异**。

最主要的变化只有一个：

- 修复前：Agent 模式可能会莫名其妙读错配置，表现成 selected model 报错；
- 修复后：Agent 模式不再被用户级旧配置污染。

换句话说：

- 这不是 UI 改版；
- 不是功能增加；
- 不是渠道结构重构；
- 而是把一个错误的配置读取范围收窄回去。

唯一可能出现的行为差异是：

- 如果有人**原本故意依赖用户级全局 `ANTHROPIC_*` 配置来影响这条 Agent 链路**，那么改成只读 `project` 之后，这种“全局覆盖当前会话”的行为就不会再生效。

但从 Proma 当前的 Agent 设计来看，当前会话中显式选择的渠道、Base URL、模型，本来就应该比用户级旧残留更权威。

## 为什么开发者一开始会用 `user + project`

`user + project` 这种配置来源设计，本身在很多 CLI / SDK 系统里都很常见，因为它符合“分层配置”的习惯：

- `user`：全局默认设置；
- `project`：当前项目或当前会话的局部覆盖。

这种设计的原始目的通常是：

1. 少重复配置；
2. 允许用户保留全局 API Key / 默认端点；
3. 再由具体项目做局部覆盖。

### 但为什么在 Proma 这里反而出问题

因为 Proma 的 Agent 模式并不是一个“让用户手工运行普通 CLI 项目”的环境。

在 Proma 里：

- 当前渠道是 UI 显式选择的；
- Base URL 是当前会话明确传入的；
- 模型也是当前会话明确选中的；
- 这些当前运行时参数本来就应该是最高优先级。

这时再把用户级旧配置读进来，反而会造成：

- 当前选择被旧值覆盖；
- 会话参数来源变得不透明；
- 用户看到的报错与真实根因脱节。

所以更准确的结论不是“`user + project` 在所有系统里都错”，而是：

**`user + project` 这个通用分层思路，放到 Proma 的 Agent 运行链路里，不适合作为默认行为。**

## 为什么升级或重装后可能复发

因为当前补丁命中的文件位于发行版安装目录：

```text
C:\Users\yzjiang\AppData\Local\Programs\Proma\resources\app\dist\main.cjs
```

这类文件在以下场景中很容易被覆盖：

- 重装 Proma；
- 升级发行版；
- 覆盖安装；
- 官方新版替换 `resources\app\dist\main.cjs`。

因此，当前这类补丁如果没有被正式并入发行版本体，就会在后续安装过程中再次丢失。

## 每次更新后建议的最小检查清单

以后每次升级、重装或覆盖安装后，建议按以下顺序快速复查：

1. 确认当前问题是不是仍然表现为 Agent 模式 selected model 报错；
2. 打开：

```text
C:\Users\yzjiang\AppData\Local\Programs\Proma\resources\app\dist\main.cjs
```

3. 搜索 `settingSources`；
4. 如果仍然是：

```js
settingSources: ["user", "project"]
```

则说明发行版安装目录中的旧逻辑又回来了；
5. 再检查：

```text
C:\Users\yzjiang\.proma\sdk-config\settings.json
```

是否残留旧的 `ANTHROPIC_*`；
6. 如果确认仍是同类问题，再按既有最小补丁思路处理；
7. 重启 Proma 后再做实际 Agent 功能复测。

## 在源码仓库里应该关注什么

如果不是检查发行版安装目录，而是在源码仓库中排查本问题，优先关注：

```text
apps/electron/src/main/lib/adapters/claude-agent-adapter.ts
```

以及所有与下面这些关键词有关的逻辑：

- `settingSources`
- Claude Agent SDK 初始化
- Anthropic 配置来源
- `ANTHROPIC_*` 环境变量构建
- Agent 运行时配置叠加顺序

## 最后结论

以后如果再次看到下面这种现象：

- Chat 模式正常；
- Agent 模式接第三方 Anthropic 渠道失败；
- 页面报 selected model 错误；

请优先想到：

**这很可能不是模型本身的问题，而是 Agent 运行时重新读取了用户级污染配置。**

最应该优先检查的两个位置仍然是：

1. `C:\Users\yzjiang\AppData\Local\Programs\Proma\resources\app\dist\main.cjs`
2. `C:\Users\yzjiang\.proma\sdk-config\settings.json`

而对这个问题最关键的一句话总结仍然是：

**`main.cjs` 里如果允许读取 `user + project`，就可能把用户级旧 `ANTHROPIC_*` 污染重新带回当前 Agent 会话；把它收窄为只读 `project`，就是这次问题的核心修复点。**
