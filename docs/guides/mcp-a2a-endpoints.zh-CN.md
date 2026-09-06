# MCP / A2A 端点配置教程

> 三步走：**本地跑通 → 暴露到公网 https → 注册进 AgentTrust**。

> 💡 **如果觉得这个太难的话，请直接打开 Claude Code 跟他说「给你自己配一个 mcp 的公网网关」，让它帮你配吧。**
> 配完建议回来对照「四个不要」检查一遍——尤其是**别让它关掉你的鉴权**。

## 〇、硬规则：注册表单只接受这一种端点（不满足直接拦截）

| 规则 | 怎么校验 | 不满足的后果 |
| --- | --- | --- |
| 协议必须是 `https://` | URL 的 `protocol === "https:"` | 表单直接报"端点不合法"，无法提交 |
| 主机名是公网可达的域名 | 主机名不在内网黑名单 + 主机名含 `.` | 同上 |
| 端点注册后**永久不可改** | 合约 `AgentInfo.endpoint` 没有 setter | 填错只能注销重新注册（押金按规则退） |

**完整内网黑名单**（与前端 `isPublicEndpoint` 校验对齐，注册页会直接拦下，**不要试图绕**）：

| 类型 | 命中示例 |
| --- | --- |
| 字面写法 | `localhost`、`::1`、`0.0.0.0` |
| IPv4 环回 | `127.x.x.x`（任意） |
| IPv4 私网 | `10.x.x.x`、`192.168.x.x`、`172.16.x.x` ~ `172.31.x.x` |
| 特殊主机名后缀 | `*.localhost`、`*.local`、`*.internal`、`*.home.arpa` |

> **为什么 `0.0.0.0` 也在黑名单里**：它是"监听所有网卡"的绑定地址，不是任何客户端能连到的门牌号（Linux 会映射到 `127.0.0.1`、Windows/macOS 直接失败），注册后等于永久废掉的身份。服务照旧绑 `0.0.0.0:8123` 完全没问题——只是注册时**必须填对外可达的域名**。

## 〇之二、注册 ≠ 开放：端点请照常开鉴权

这是最容易搞错的一点，**弄错了会把你的服务裸奔到公网**。

AgentTrust 注册时**只把端点地址写进链上目录**：

- **不会**主动探测你的端点（注册表单只做 https + 非内网两项**格式校验**，不发任何网络请求）；
- **不会**要求你的端点匿名可访问；
- **不会**替任何调用方做鉴权或转发。

| 环节 | 谁负责 |
| --- | --- |
| 把地址登记进目录，让别人能找到你 | AgentTrust（链上） |
| 谁能调、要不要 token、限不限流 | **你的 MCP 服务自己** |

所以：

- ✅ **正确**：端点照常要求 `Authorization: Bearer <token>`，把 `https://mcp.你的域名/mcp` 填进注册表单即可。
- ❌ **错误**：为了让"注册校验通过"而关掉鉴权、开放匿名调用。注册**根本不校验这个**——关掉只会让任何人都能白嫖你的算力和模型额度。

> 有人把下面的自检命令误解成"注册必须先通过匿名探测"，于是拆掉了自家端点的鉴权。
> 自检是**你自己确认服务活着**的手段，AgentTrust 不会跑它。
> 不带 token 时自检返回 `401 Unauthorized` **才是正常的**——那正好证明鉴权生效了。

### 注册页上那两条提示是什么意思

填端点时，表单下方可能出现两行提示。它们都只是**提醒**，不会阻止你提交：

| 提示 | 含义 | 你要做什么 |
| --- | --- | --- |
| *Registering only publishes the address… keep your endpoint's own auth (for example a Bearer token).* | 提示你：注册只是登记地址，**不必**为了让别人（或平台）探测而关掉鉴权 | **什么都不用做**。端点照常挂 Bearer token 就行 |
| *Warning: <主机名> looks like a temporary tunnel address…* | 提示你：这个域名是临时隧道，`*.trycloudflare.com` 之类每次重启都会变，而链上端点永久不可改 | 只是试水 → 可以继续；要当长期身份 → 先绑自有域名 |

看到第一条**不等于**你的配置有问题，它对所有填写的端点都会显示。

### `/health` 和 `/mcp` 不是一回事

很多 MCP 服务（比如 `claude-agent-mcp-server`）会额外开一个 `/health` 状态页。它的典型响应长这样：

```json
{ "ok": true, "name": "claude-agent-mcp-server", "version": "1.0.0", "mcp": "/mcp", "auth": "bearer" }
```

- `/health` 是**门口的营业状态牌**：公开的、不需要 token、只用来确认进程活着；
- `/mcp` 才是**真正的服务窗口**：MCP 协议入口，要带 `Authorization: Bearer <token>`。

📌 **注册时一律填 `/mcp`（或你服务端自己的 MCP 路径），不要填 `/health`。**
上面那个响应里的 `"mcp": "/mcp"` 就是服务自己告诉你的正确答案。
把 `/health` 填进去也不会被拦（格式是合法的 https 公网地址），但别人顺着链上找到的会是一个只会回 `ok` 的状态页，调不了任何工具。

希望别人能调用时，让他们向你索取 token，或者接入付费/授权网关。
AgentTrust 只负责让别人**找到**你，不负责**放行**。

### 从零配一个公网 HTTPS 网关（本机 + 自有域名，推荐）

> 💡 **嫌麻烦？直接让 Claude Code 帮你配。**
> 打开 Claude Code 跟它说一句：**「给你自己配一个 mcp 的公网网关」**，它会全程代劳。
> 配完记得回来对照文末「四个不要」检查一遍——尤其是**别让它关掉你的鉴权**。

先看清楚流量走向，这是最容易搞混的一步：

| 环节 | 在哪 | 说明 |
| --- | --- | --- |
| 你的 MCP 服务 | **你自己的电脑** `127.0.0.1:8123` | 不是跑在云服务器上 |
| 隧道进程 | 你自己的电脑（cloudflared） | 主动往外连，不用开入站端口 |
| DNS | 阿里云 / 腾讯云 / Cloudflare 任一 | 只加一条 `mcp` 的 CNAME |
| 你现有的网站 | 云服务器上的 A 记录 | **一个字都不要动** |

所以：**别去云服务器上给 MCP 配 Nginx**——MCP 在你电脑上，那台机器根本不参与。

#### 第 1 步 · 确认本机 MCP 在跑

浏览器打开 `http://127.0.0.1:8123/health`，能返回 JSON 就说明本地服务活着。

#### 第 2 步 · 建 Cloudflare Zero Trust 隧道（推荐走控制台，少踩坑）

1. 打开 https://one.dash.cloudflare.com/ （没有账号就免费注册）；
2. **Networks → Tunnels → Create a tunnel → Cloudflared**，名字随便起，比如 `claude-agent-mcp`；
3. 选 **Windows**，复制页面给的安装/运行命令（本质是 `cloudflared.exe service install <TOKEN>`），
   在**本机的 cmd** 里执行——装成服务后会开机自启；
4. **Public Hostname** 加一条：

   | 字段 | 填什么 |
   | --- | --- |
   | Subdomain | `mcp` |
   | Domain | 你的域名。列表里没有就选 *Add a domain from outside Cloudflare* / CNAME setup，**不要**改 NS |
   | Path | 留空 |
   | Service type | `HTTP` |
   | URL | `http://127.0.0.1:8123` |

5. 保存后概览页会给出 **CNAME 目标**，形如 `<UUID>.cfargotunnel.com`——复制下来。

> ⚠️ **不要点「把 NS 改成 Cloudflare」**。域名 DNS 还在阿里云，一改现有站点的解析全乱。
> 走 CNAME setup（部分接入），只签 `mcp.你的域名` 这一条就够了。

#### 第 3 步 · 去 DNS 服务商加一条 CNAME

以阿里云为例：控制台 → 云解析 DNS → 你的域名 → 解析设置 → 添加记录。

| 字段 | 填什么 |
| --- | --- |
| 记录类型 | `CNAME` |
| 主机记录 | `mcp` |
| 解析请求来源 | 默认 |
| 记录值 | `<UUID>.cfargotunnel.com`（第 2 步拿到的） |
| TTL | 10 分钟 |

**不要改**：`@`、`www`、`NS`。

加完在本机验证：

```bash
nslookup mcp.你的域名
# 应看到 CNAME 指向 *.cfargotunnel.com
```

#### 第 4 步 · 验证，然后注册

```bash
# 1. /health 是公开状态页 —— 返回 JSON 就说明隧道通了
curl -i https://mcp.你的域名/health

# 2. /mcp 才是服务入口：不带 token 应当返回 401
#    这恰恰说明鉴权在工作，不是配错了
curl -i -X POST https://mcp.你的域名/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"probe","version":"0.0.1"}}}'

# 3. 带上自己的 token 再跑一次，这次应该返回带 serverInfo 的 JSON
#    （在上一条的 -H 里加： "Authorization: Bearer <TOKEN>"）
```

填进 AgentTrust 注册页的是：

```
✅ https://mcp.你的域名/mcp
```

**Token 不要写进链上。** 链上只登记地址，凭据私下给需要连你的 agent。

#### 先试水：不想绑域名就用 quick tunnel

```bash
# 装 cloudflared
winget install --id Cloudflare.cloudflared      # Windows
brew install cloudflared                        # macOS
# Linux：到 https://github.com/cloudflare/cloudflared/releases 下二进制

cloudflared tunnel --url http://127.0.0.1:8123
# 输出类似 https://some-random-word-1234.trycloudflare.com —— 它就是你的临时公网端点
```

> ⚠️ **别拿这个地址当长期身份**：quick tunnel 域名每次启动都会变。
> 端点写进链上后**永久不可改**（合约 `AgentInfo.endpoint` 没有 setter），
> 隧道一重启，链上身份就指向一个死地址，只能注销重来。
> 注册页会对 `*.trycloudflare.com`、`*.ngrok.io`、`*.loca.lt` 这类临时隧道域名亮红字警告——
> 看到警告先想清楚：我只是试水，还是要拿它当长期身份？后者请回到第 2 步绑自有域名。

#### 四个不要（这四条踩了最疼）

| ❌ 不要 | 为什么 |
| --- | --- |
| 为了"能被注册探测"关掉 Bearer 鉴权 | AgentTrust 注册时**一个网络请求都不发**，从来不探测你的服务。关掉鉴权纯属自废武功，只会让任何人都能白嫖你的算力和模型额度 |
| 开 `MCP_PUBLIC_ASK=1` 这类匿名开放开关 | 等于把你的 Claude 额度送给全网——谁都能拿它问问题，账单是你的 |
| 拿 quick tunnel 的临时域名当长期身份 | 域名每次重启都变，而链上端点永久不可改 |
| 改 `@`/`www`/NS，或去云服务器上给 MCP 配 Nginx | MCP 在你自己电脑上，云服务器那条 A 记录完全不参与。动了它只会弄坏现有站点 |

---

## 一、端点是什么

端点就是一个普通的 `https://` 网址，只不过它响应的不是网页，而是 MCP / A2A 协议请求。

| 类型 | 作用 |
| --- | --- |
| MCP 端点 `https://你的域名/mcp` | 工具服务地址，别的程序按这个地址调用你的工具 |
| A2A 端点 `https://你的域名/a2a` | 智能体名片 + 洽谈室（A2A = Agent-to-Agent），说明你是谁、会什么 |

没有地址别人找不到你；地址写错（比如填了内网地址）同样找不到。

## 二、本地跑通

Python 用 FastMCP 最省事：

```python
# server.py
from fastmcp import FastMCP

mcp = FastMCP("MyAgent")

@mcp.tool
def get_price(symbol: str) -> str:
    """查询代币价格（示例工具）"""
    return f"{symbol} = 100 USD"

mcp.run()  # 默认 stdio，先在本地验证工具逻辑
```

```bash
pip install fastmcp
python server.py
```

Node.js 用官方 SDK：`npm i @modelcontextprotocol/sdk`，思路相同。先本地跑通，再上公网。

## 三、接进你手头的智能体（三选一）

目标只有一个：让 `https://你的域名/mcp` 变成别人连得上、你自己验证过的 MCP 服务。

### 路线 A：Claude Code（Anthropic 官方 CLI）

```bash
npm install -g @anthropic-ai/claude-code   # 需要 Node.js 18+
claude                                     # 在任意项目目录启动会话
claude mcp add --transport http my-agent https://agent.example.com/mcp
claude mcp list                            # 查看已接入的 server
```

也可写进项目根目录的 `.mcp.json`，随仓库共享：

```json
{
  "mcpServers": {
    "my-agent": { "type": "http", "url": "https://agent.example.com/mcp" }
  }
}
```

验证：会话里输入 `/mcp`，能看到 `my-agent` 已连接、`get_price` 已列出，再实际调一次工具即端到端打通。

> Claude Code 是交互式 CLI，不是常驻 HTTP 服务。想让它成为你的端点，用它来开发并运行自己的 MCP server（路线 C），不要把它本身暴露到公网。

### 路线 B：Codex CLI（OpenAI）

```bash
npm install -g @openai/codex
codex
```

编辑 `~/.codex/config.toml`：

```toml
[mcp_servers.my-agent]
url = "https://agent.example.com/mcp"
```

验证：重启 `codex` 后让它调一次工具；`codex mcp --help` 看你这个版本支持哪些管理子命令。

> ⚠️ Codex 对 MCP 传输类型的支持随版本变化较快，装最新版；连不上先确认你的 server 走的是 streamable HTTP。老版本只支持本地 stdio，可先 `command = "python", args = ["server.py"]` 过渡。

### 路线 C：自建智能体（Python / Node.js）

Python：把上一节的 `mcp.run()` 换成公网模式即可。

```python
mcp.run(transport="http", host="0.0.0.0", port=8000)   # 旧版写法：transport="streamable-http"
```

Node.js / TypeScript：

```bash
npm install @modelcontextprotocol/sdk zod
```

```ts
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { z } from "zod";

const server = new McpServer({ name: "MyAgent", version: "0.1.0" });
server.tool("get_price", { symbol: z.string() }, async ({ symbol }) => ({
  content: [{ type: "text", text: `${symbol} = 100 USD` }],
}));
// 在你的 HTTP 层（Express/Hono）挂载 streamable HTTP 传输后启动
```

**加 A2A（可选）**：在同域名下暴露名片文件 `https://你的域名/.well-known/agent.json`，写明名称、能力与端点。

**调试**：`npx @modelcontextprotocol/inspector` 打开官方 Inspector，填入 URL 连一次，工具列表与调用结果全部可视化。

## 四、暴露到公网

把服务部署到有公网 IP 的机器，前面挂 HTTPS（Caddy 两行配置自动签证书）：

```
# Caddyfile
agent.example.com {
    reverse_proxy 127.0.0.1:8000
}
```

没有服务器可部署到 Railway / Fly.io / 云函数等支持长连接的平台，它们直接给 https 域名。

| 验收项 | 为什么 |
| --- | --- |
| `https://` 开头 | 明文 http 在公网等于裸奔，注册页也只接受 https |
| 公网可访问 | 填 `localhost`、`127.0.0.1`、`192.168.x.x`，别的智能体永远连不上 |
| 域名稳定 | 端点**注册后永久不可改**，别用临时地址 |

## 五、自检：确认别人真连得上

```bash
# 1. 地址通不通、证书对不对（返回 4xx 也算通了，说明服务在应答）
curl -i https://agent.example.com/mcp

# 2. MCP 协议握手（streamable HTTP 要求 POST + JSON-RPC）
curl -i -X POST https://agent.example.com/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"probe","version":"0.0.1"}}}'
```

第 2 条返回带 `serverInfo` 的 JSON 即成功。A2A 同理：访问 `https://你的域名/.well-known/agent.json`，能看到名片 JSON 即成功。

## 六、注册到 AgentTrust

1. 登录 agenttrust.site → **智能体注册**；
2. 「MCP/A2A 端点」填自检通过的 `https://agent.example.com/mcp`；
3. 三条必知规则：
   - **端点注册后永久不可改**（链上 `AgentInfo` 没有 setter），填错只能重新注册新身份；
   - **端点 ≠ 身份证**，任何人都能注册同一个网址，它不证明归属；
   - 真正的全局唯一标识是 **ATID**（注册时生成的链上 NFT 编号）和 **(来源平台, 外部智能体 ID) 绑定组合**。注册后在「外部智能体身份」区域做 L1 绑定声明，再逐步升级 L2–L4 强证明。

## 七、常见坑速查

| 症状 | 原因 | 解法 |
| --- | --- | --- |
| curl `Connection refused` / 超时 | 没监听公网或防火墙没开端口 | 检查 `0.0.0.0` 监听与安全组 |
| 浏览器能开但 curl 报证书错误 | 证书域名不匹配 | 用 Caddy 自动签，别手动折腾 |
| 本地正常，公网 502 | 反代指向的端口不对 | 核对 Caddyfile 的 `reverse_proxy` 端口 |
| 注册后想换地址 | 端点不可改 | 换新身份重新注册（押金按注销规则退） |
| 填了内网 IP | 注册页直接拦截 | 换公网 https 域名 |
| Claude Code 里 `/mcp` 看不到 server | 没注册或 scope 不对 | `claude mcp add` 重加，注意 `--scope`（user 全局 / project 当前项目） |
| Claude Code 报 `Transport error` | 端点不是 streamable HTTP 或反代截断了 POST | 回到第五节用 curl 自检握手 |
| Codex 连不上 HTTP server | 版本较老不支持 `url` 形式 | 升级 Codex，或本地 stdio 过渡 |
| `nslookup mcp.你的域名` 没解析到 `*.cfargotunnel.com` | CNAME 没填或还没生效 | 核对记录值，等 TTL（10 分钟）后再试 |
| 隧道显示 disconnected | 本机 cloudflared 服务没在跑 | Windows 服务里看状态；或重跑 `cloudflared.exe service install <TOKEN>` |
| 一觉起来 Claude 额度掉了很多 | 开了 `MCP_PUBLIC_ASK=1` 之类的匿名开关 | 立刻关掉，`/mcp` 恢复要求 Bearer |
| 有 AI 跟你说「注册页探测不带 token，得关鉴权」 | 错误前提——注册**一个网络请求都不发** | 别照做，把鉴权开回来（见「四个不要」） |
| 注册后别人说调不到工具 | 填的是 `/health` 而不是 `/mcp` | `/health` 只是状态页，改填 `/mcp` |
| 改完 NS 后现有网站打不开了 | 整站接入了 Cloudflare 但源站没配好 | NS 改回去；只走 CNAME setup 签 `mcp` 子域 |

---

*本教程对应 AgentTrust 前端校验规则（只拦「填了但不合法」的端点）。有疑问欢迎提 issue：https://github.com/Fishman-free/multiagent*
