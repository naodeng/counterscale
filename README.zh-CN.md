<p align="right"><a href="README.md">English</a></p>

# Counterscale

![](/packages/server/public/counterscale-logo-300x300.webp)

![ci status](https://github.com/benvinegar/counterscale/actions/workflows/ci.yaml/badge.svg)
[![License](https://img.shields.io/github/license/benvinegar/counterscale)](https://github.com/benvinegar/counterscale/blob/master/LICENSE)
[![codecov](https://codecov.io/gh/benvinegar/counterscale/graph/badge.svg?token=NUHURNB682)](https://codecov.io/gh/benvinegar/counterscale)

Counterscale 是一款简单的网站分析追踪与仪表盘，可在 Cloudflare 上自托管部署。

设计目标是易于部署与维护，运营成本趋近于零——即便在高流量下（Cloudflare [免费版](https://developers.cloudflare.com/workers/platform/pricing/#workers)理论上可支持约 10 万次/天的请求）。

_Counterscale 由 [Modem——开发团队的自动分类 PM](https://modem.dev) 赞助。_

## 许可证

Counterscale 为免费开源软件，采用 MIT 许可证。详见：[LICENSE](LICENSE)。

## 限制说明

Counterscale 主要基于 Cloudflare Workers 与 [Workers Analytics Engine](https://developers.cloudflare.com/analytics/analytics-engine/)。截至 2025 年 2 月，Workers Analytics Engine 的**最长保留期为 90 天**，因此 Counterscale 仅能展示最近 90 天的记录数据。我们同时支持将数据长期存储在 R2 桶中（使用 Apache Arrow 文件），该长期存储默认开启，可通过 CLI 关闭。

## 安装

### 环境要求

* macOS 或 Linux
* Node v20 及以上
* 有效的 [Cloudflare](https://cloudflare.com) 账号（免费或付费均可）

### Cloudflare 准备

若尚未拥有账号，请[在此创建 Cloudflare 账号](https://dash.cloudflare.com/sign-up)并完成邮箱验证。

1. 登录 Cloudflare 控制台，若尚未配置，请先设置 [Cloudflare Workers 子域名](https://developers.cloudflare.com/workers/configuration/routing/workers-dev/)
2. 为账号启用 [Cloudflare Analytics Engine](https://developers.cloudflare.com/analytics/analytics-engine/) 测试版：进入「存储与数据库 > Analytics Engine」并点击「启用」按钮（[截图](./docs/enable-analytics-engine.png)）。随后弹出的「创建数据集」窗口可忽略并关闭。
    - 说明：若首次使用 Workers，需先创建一个 Worker 才能启用 Analytics Engine。进入「Workers 与 Pages > 概览」，点击「创建 Worker」按钮（[截图](./docs/create-worker.png)）创建一个「Hello World」Worker（名称可任意，之后可删除）。
3. 创建 [Cloudflare API 令牌](https://developers.cloudflare.com/fundamentals/api/get-started/create-token/)。该令牌至少需要具备 `Account.Account Analytics` 权限（[截图](./docs/api-token.png)）。
    - _警告：请保持该页面打开或将 API 令牌妥善保存（如密码管理器），关闭窗口后将无法再次查看该令牌，需重新创建。_

### 部署 Counterscale

首先登录 Cloudflare 并授权 Cloudflare CLI（Wrangler）：

```bash
npx wrangler login
```

然后运行 Counterscale 安装程序：

```bash
npx @counterscale/cli@latest install
```

按提示操作。会要求输入之前创建的 Cloudflare API 令牌，并询问是否用密码保护仪表盘：

- 选择 **是**（建议用于公开部署）：需设置一个密码，访问分析仪表盘时需输入该密码。
- 选择 **否**：仪表盘将公开可访问，无需认证。

脚本完成后，服务端应用即已部署。可访问 `https://{部署时输出的子域名}.workers.dev` 进行验证。

注意：_首次部署 Counterscale 时，Worker 子域名可能需要几分钟才能生效。_

### 开始记录网站流量

可通过两种方式加载追踪代码：

#### 1. 脚本加载器（CDN）

部署后，Counterscale 会在部署地址提供 `tracker.js`：

```
https://{部署时输出的子域名}.workers.dev/tracker.js
```

将以下代码片段复制到网站 HTML 中即可开始上报流量：

```html
<script
    id="counterscale-script"
    data-site-id="your-unique-site-id"
    src="https://{部署时输出的子域名}.workers.dev/tracker.js"
    defer
></script>
```

#### 2. 包/模块方式

Counterscale 追踪器以 npm 包形式发布：

```bash
npm install @counterscale/tracker
```

使用站点 ID 和部署的上报端点 URL 初始化：

```typescript
import * as Counterscale from "@counterscale/tracker";

Counterscale.init({
    siteId: "your-unique-site-id",
    reporterUrl: "https://{部署时输出的子域名}.workers.dev/collect",
});
```

**可用方法**
| 方法 | 参数 | 返回类型 | 说明 |
|--------|------------|-------------|-------------|
| `init(opts)` | `ClientOpts` | `void` | 使用站点配置初始化 Counterscale 客户端；若不存在则创建全局实例。 |
| `isInitialized()` | 无 | `boolean` | 检查客户端是否已初始化。 |
| `getInitializedClient()` | 无 | `Client \| undefined` | 返回已初始化的客户端实例，未初始化则返回 undefined。 |
| `trackPageview(opts?)` | `TrackPageviewOpts?` | `void` | 记录一次页面浏览。需先初始化客户端；未提供时自动检测 URL 与 referrer。 |
| `cleanup()` | 无 | `void` | 清理客户端实例并移除事件监听，将全局客户端设为 undefined。 |

#### 3. 服务端模块

若希望在服务端而非浏览器中上报分析数据，可使用 `/server` 模块：

```bash
npm install @counterscale/tracker
```

```typescript
import * as Counterscale from "@counterscale/tracker/server";

// 初始化追踪器
Counterscale.init({
    siteId: "your-unique-site-id",
    reporterUrl:
        "https://{部署时输出的子域名}.workers.dev/collect",
    reportOnLocalhost: false, // 可选，默认 false
    timeout: 2000, // 可选，默认 1000ms
});

// 记录一次页面浏览
await Counterscale.trackPageview({
    url: "https://example.com/page", // 或相对路径：'/page'
    hostname: "example.com", // 使用相对 URL 时必填
    referrer: "https://google.com",
    utmSource: "social",
    utmMedium: "twitter",
});
```

**服务端模块方法**
| 方法 | 参数 | 返回类型 | 说明 |
|--------|------------|-------------|-------------|
| `init(opts)` | `ServerClientOpts` | `void` | 初始化服务端追踪器。 |
| `isInitialized()` | 无 | `boolean` | 检查追踪器是否已初始化。 |
| `getInitializedClient()` | 无 | `ServerClient \| undefined` | 返回已初始化的服务端客户端实例。 |
| `trackPageview(opts)` | `TrackPageviewOpts` | `Promise<void>` | 记录一次页面浏览。需显式传入 URL 和 hostname。 |
| `cleanup()` | 无 | `void` | 清理服务端客户端实例。 |

服务端模块面向后端应用，与客户端版本区别如下：

- 无依赖 DOM 的功能（自动追踪、浏览器埋点）
- 使用 fetch API 而非 XMLHttpRequest
- 需显式传入 URL 和 hostname
- 采用「发出即忘」策略，追踪错误不会抛出异常

## 升级

大多数版本升级只需重新执行 CLI 安装命令：

```bash
npx @counterscale/cli@latest install

# 或指定版本
# npx @counterscale/cli@VERSION install
```

无需重新输入 API 密钥，数据会保留。

Counterscale 使用[语义化版本](https://semver.org/)。升级主版本（如 2.x、3.x、4.x）时可能有额外步骤，请参考[发布说明](https://github.com/benvinegar/counterscale/releases)。

## 故障排除

若网站无法立即访问（例如「安全连接失败」），可能是 Cloudflare 尚未激活你的子域名（yoursubdomain.workers.dev）。该过程可能需要一分钟；可在 Cloudflare 控制台（Workers & Pages → counterscale）中查看新建 Worker 的状态。

## 高级用法

### 手动记录页面浏览

初始化 Counterscale 追踪器时，将 `autoTrackPageviews` 设为 `false`，然后在需要记录时手动调用 `Counterscale.trackPageview()`。

```typescript
import * as Counterscale from "@counterscale/tracker";

Counterscale.init({
    siteId: "your-unique-site-id",
    reporterUrl: "https://{部署时输出的子域名}.workers.dev/collect",
    autoTrackPageviews: false, // <- 不要忘记此项
});

// ... 在发生页面浏览时
Counterscale.trackPageview();
```

### 自定义域名

部署地址可改为使用你拥有的自定义域名。详见[此处](https://developers.cloudflare.com/workers/configuration/routing/custom-domains/)。

## CLI 命令

Counterscale 提供命令行工具（CLI）用于安装、配置和管理部署。

### 可用命令

#### `install`

安装并将 Counterscale 部署到 Cloudflare 的主命令。

```bash
npx @counterscale/cli@latest install
```

选项：

- `--advanced` - 启用高级模式，可自定义 Worker 名称与分析数据集
- `--verbose` - 输出更多日志

#### `auth`

管理 Counterscale 部署的认证设置。

```bash
npx @counterscale/cli@latest auth [子命令]
```

子命令：

- `enable` - 为部署启用认证
- `disable` - 关闭认证
- `roll` - 更新/轮换认证密码

##### 示例

启用认证：

```bash
npx @counterscale/cli@latest auth enable
```

关闭认证：

```bash
npx @counterscale/cli@latest auth disable
```

更新密码：

```bash
npx @counterscale/cli@latest auth roll
```

#### `storage`

管理 Counterscale 部署的长期存储设置。

```bash
npx @counterscale/cli@latest storage [子命令]
```

子命令：

- `enable` - 为部署启用存储
- `disable` - 关闭存储

##### 示例

启用存储：

```bash
npx @counterscale/cli@latest storage enable
```

关闭存储：

```bash
npx @counterscale/cli@latest storage disable
```

## 开发

如何参与开发请参阅 [Contributing](CONTRIBUTING.md)。

## 说明

### 数据库

当前仅有一个「数据库」：Cloudflare Analytics Engine 数据集，通过 HTTP 与 Cloudflare API 通信。

目前没有本地「测试」数据库。因此在本地开发时：

- 写入不会生效（不会记录任何请求）
- 读取来自生产环境的 Analytics Engine 数据集（本地开发看到的是生产数据）

### 采样

Cloudflare Analytics Engine 使用采样以在高流量下控制 ingestion/查询成本（与多数分析工具类似，可参考 [Google Analytics 关于采样](https://support.google.com/analytics/answer/2637192?hl=en#zippy=%2Cin-this-article)）。更多关于 [CF AE 采样的说明](https://developers.cloudflare.com/analytics/analytics-engine/sampling/)可在此查看。
