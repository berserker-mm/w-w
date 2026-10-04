# WARP to WireGuard

一个部署在 Cloudflare Workers 上的自托管 Web 工具，用于将 Cloudflare WARP / Zero Trust 设备注册转换为 WireGuard、Clash Meta / Mihomo / OpenClash、sing-box 配置和可扫码导入的二维码。

> 本项目不包含任何预设 Team Name、账户 ID、真实 Endpoint、私钥或 Token。所有组织信息和自定义 Endpoint 均由使用者自行输入。

## 功能

- Zero Trust 邮箱 OTP 登录：填写 Team Name 和邮箱，输入验证码后直接生成配置。
- 手动 JWT 模式：支持粘贴从 Cloudflare Access WARP 登录页提取的短期 JWT。
- 自动生成 X25519 / WireGuard 密钥对并注册新的 WARP 设备。
- 自动提取 Cloudflare `client_id`，转换为 Clash、sing-box 和 Shadowrocket 所需的 `reserved` 三字节数组。
- 输出格式：
  - WireGuard `.conf`
  - Clash Meta / Mihomo / OpenClash YAML 节点
  - sing-box WireGuard outbound JSON
  - Shadowrocket / WireGuard 扫码二维码（SVG）
- 支持用户填写自己的自定义 Endpoint；留空时使用 Cloudflare API 返回的 Endpoint。
- 二维码在 Worker 内生成，不向第三方二维码服务发送私钥或配置。
- 响应使用 `Cache-Control: no-store`，应用不使用数据库、KV 或持久化存储保存认证信息。

## 工作原理

1. 用户通过 Cloudflare Access OTP 登录，或提供临时 JWT。
2. Worker 使用 Web Crypto API 生成 X25519 密钥对。
3. Worker 调用 Cloudflare WARP 设备注册 API，提交公钥。
4. API 返回设备地址、Peer 公钥、Endpoint 和 Base64 编码的 `client_id`。
5. 工具将 `client_id` 解码为三个十进制字节，即客户端配置中的 `reserved`。
6. 页面生成不同客户端所需的配置和二维码。

`reserved` 转换示意：

```text
client_id (Base64) -> 3 raw bytes -> [decimal, decimal, decimal]
```

## 部署

### 前置条件

- Node.js 22 或更新版本
- Cloudflare 账户
- 已登录 Wrangler，或设置了拥有 Workers 权限的 `CLOUDFLARE_API_TOKEN`

### 本地运行

```bash
git clone https://github.com/zqs1qiwan/warp2wireguard.git
cd warp2wireguard
npm install
npm run dev
```

Wrangler 会输出本地预览地址，通常为 `http://localhost:8787`。

### 部署到 Workers

```bash
npm install
npm run deploy
```

仓库中的 `wrangler.jsonc` 不包含账户 ID、Zone 或自定义域名。Wrangler 会使用你当前登录的 Cloudflare 账户，并默认部署到 Workers.dev。

### 配置自定义域名

推荐在 Cloudflare Dashboard 中为 Worker 添加 Custom Domain。也可以自行在 `wrangler.jsonc` 中添加路由：

```jsonc
{
  "routes": [
    {
      "pattern": "warp.example.com/*",
      "zone_name": "example.com"
    }
  ]
}
```

不要将真实账户 ID、内部域名、私有 Endpoint 或其他敏感信息提交到公共仓库。

### 启用 OTP 登录

为避免新部署的公开接口被用于批量发送验证码，OTP API 默认关闭；手动 JWT 模式不受影响。仅在配置 Cloudflare Access、WAF Rate Limiting 或等效防滥用措施后启用 OTP，并限制允许使用的 Team：

```bash
npx wrangler secret put OTP_ENABLED
# 输入 true

npx wrangler secret put OTP_ALLOWED_TEAMS
# 输入允许的 Team Name；多个值使用逗号分隔，例如 team-a,team-b

npx wrangler secret put OTP_STATE_SECRET
# 输入至少 32 个随机字符；可使用密码管理器生成

npm run deploy
```

三个变量必须同时配置。`OTP_ALLOWED_TEAMS` 中不要填写完整域名，只填写 `your-team-name.cloudflareaccess.com` 的 `your-team-name` 部分。`OTP_STATE_SECRET` 用于 AES-GCM 加密浏览器和 Worker 之间短期保存的 OTP 会话，必须保密且至少 32 个字符。修改变量后应重新部署并测试。

## 使用方法

### 方法一：Zero Trust 登录

1. 在 Zero Trust Dashboard 中确认组织允许该邮箱注册设备。
2. 确认组织已配置 One-time PIN 身份提供商；如果只配置了第三方 SSO，请使用手动 JWT 模式。
3. 在页面填写 Team Name。Team Name 是 `your-team-name.cloudflareaccess.com` 中的 `your-team-name`。
4. 填写邮箱并发送验证码。
5. 输入邮件验证码，点击“验证并生成配置”。

### 方法二：手动 JWT

1. 浏览器访问：

   ```text
   https://your-team-name.cloudflareaccess.com/warp
   ```

2. 完成登录，在出现打开 Cloudflare WARP 的提示时可以取消。
3. 打开浏览器开发者工具 Console。
4. 执行：

   ```js
   console.log(document.querySelector('meta[http-equiv="refresh"]').content.split('=').pop())
   ```

5. 复制输出的 `eyJ...` JWT 并立即粘贴到工具中。该 JWT 通常只有约 60 秒有效期。

## 自定义 Endpoint

默认情况下，工具使用 Cloudflare WARP API 返回的 Endpoint。若你的组织有自己的合法 Endpoint，可以在生成前自行填写，例如：

```text
203.0.113.1:2408
```

`203.0.113.0/24` 是 RFC 5737 文档示例网段，上述地址仅用于说明，不是可连接的 WARP Endpoint。

## 安全与隐私

- 项目源码不包含作者或部署者的 Team Name、账户 ID、邮箱、真实 Endpoint、私钥、JWT 或刷新 Token。
- JWT、OTP、Cloudflare Access 会话 Cookie 和生成的私钥会经过你部署的 Worker 内存，因此建议自行部署，不要将敏感组织凭据提交给不受信任的公共实例。
- Worker 不配置数据库、KV、R2、Analytics Engine 等持久化绑定。
- 敏感 API 响应明确使用 `Cache-Control: no-store`。
- OTP 回调目标由后端根据经过格式校验的 Team Name 固定生成，客户端不能指定任意回调 URL。
- Cloudflare Access OTP Cookie 使用部署者的 `OTP_STATE_SECRET` 加密和认证；浏览器只接收十分钟有效的密文会话。
- 二维码由 Worker 内置库生成，不调用第三方二维码 API。
- OTP 默认关闭并受 `OTP_ALLOWED_TEAMS` 限制。公开启用前必须配置 Cloudflare Access、WAF Rate Limiting、Turnstile 或等效防滥用措施。
- 生成的配置包含私钥。不要分享配置、二维码、截图或浏览器响应内容。

## 已知限制

- Zero Trust 登录模式依赖 Cloudflare Access One-time PIN；第三方 SSO 流程无法在本站表单中直接完成。
- Cloudflare 可能调整未公开的 WARP 注册接口或 Access 登录页面结构，届时需要更新实现。
- 原生 WireGuard 通常不支持 Cloudflare 使用的 `reserved` 字段。请使用支持该字段的 Clash Meta / Mihomo、sing-box、Shadowrocket 等客户端。
- 同一份 WireGuard 私钥不应同时在多个设备上使用。每个设备应单独生成配置。

## 开发

```bash
npm install
npm run dev
npm test
npm run deploy
```

主要文件：

```text
src/index.js      Worker API、页面、配置生成逻辑
wrangler.jsonc    通用 Workers 配置，不包含个人账户或域名
package.json      项目依赖和脚本
```

## 致谢

- [rany2/warp.sh](https://github.com/rany2/warp.sh) 提供了 WARP 注册流程和 `client_id` / reserved 转换方面的重要参考。
- [qrcode-generator](https://github.com/kazuhikoarase/qrcode-generator) 用于在 Worker 内生成二维码。

## 免责声明

本项目是非官方工具，与 Cloudflare 无隶属或认可关系。使用者应遵守 Cloudflare 服务条款、所在地区法律法规及其组织安全政策。作者不对账户限制、配置失效、服务中断或其他损失承担责任。

## License

[MIT](LICENSE)

---

## English

WARP to WireGuard is a self-hosted Cloudflare Workers application that registers Cloudflare WARP / Zero Trust devices and generates WireGuard, Clash Meta / Mihomo / OpenClash, sing-box configurations, and QR codes.

### Features

- Zero Trust email OTP flow or manual short-lived JWT input.
- X25519 key generation using the Web Crypto API.
- Automatic WARP device registration and `client_id` to `reserved` conversion.
- WireGuard, Clash Meta, sing-box, and QR code output.
- Optional user-provided custom Endpoint.
- Server-side QR generation with no third-party QR API.
- No database or persistent storage bindings.

### Quick start

```bash
git clone https://github.com/zqs1qiwan/warp2wireguard.git
cd warp2wireguard
npm install
npm run dev
```

Deploy to Cloudflare Workers:

```bash
npm run deploy
```

OTP endpoints are disabled by default. Before enabling them, protect the deployment with Cloudflare Access, WAF Rate Limiting, Turnstile, or equivalent abuse controls, then set `OTP_ENABLED=true`, a comma-separated `OTP_ALLOWED_TEAMS`, and a random `OTP_STATE_SECRET` of at least 32 characters. The state secret encrypts short-lived OTP session data with AES-GCM. Manual JWT mode remains available without these variables.

The repository does not include an account ID, custom domain, organization name, private Endpoint, key, or token. Configure your own Cloudflare account and domain after cloning.

### Security

Credentials and generated private keys pass through the Worker instance you deploy. Self-host the application, protect public deployments with Cloudflare Access or rate limiting, and never share generated configuration files or QR codes.

### Disclaimer

This is an unofficial project and is not affiliated with or endorsed by Cloudflare. You are responsible for complying with Cloudflare's terms, local laws, and your organization's policies.

Licensed under the [MIT License](LICENSE).
