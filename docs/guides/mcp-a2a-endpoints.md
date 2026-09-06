# MCP / A2A Endpoint Setup Guide

> Chinese version (primary): [docs/guides/mcp-a2a-endpoints.zh-CN.md](./mcp-a2a-endpoints.zh-CN.md)
>
> Three stages: **run it locally → expose it over public https → register it on AgentTrust**.

> 💡 **If this feels like too much, just open Claude Code and tell it "set up a public gateway for your own MCP server" — let it configure it for you.**
> Afterwards it's worth coming back to check the "Four don'ts" — especially, **don't let it turn off your auth**.

## 0. Hard rule — only this shape of endpoint is accepted (everything else is rejected)

| Rule | How it's checked | What happens if you break it |
| --- | --- | --- |
| Protocol must be `https://` | `URL.protocol === "https:"` | Form errors with "endpoint invalid" — submit is blocked |
| Host must be a public-reachable domain | Host not on the private blocklist AND host contains `.` | Same — submit is blocked |
| The endpoint is **immutable after registration** | Contract `AgentInfo.endpoint` has no setter | A typo means a brand-new identity; deposit is withdrawable per deregistration rules |

**Full private-network blocklist** (mirrors `isPublicEndpoint` in the registration form — don't try to bypass):

| Kind | Examples |
| --- | --- |
| Literal | `localhost`, `::1`, `0.0.0.0` |
| IPv4 loopback | `127.x.x.x` (any) |
| IPv4 private | `10.x.x.x`, `192.168.x.x`, `172.16.x.x` – `172.31.x.x` |
| Special suffixes | `*.localhost`, `*.local`, `*.internal`, `*.home.arpa` |

> **Why `0.0.0.0` is on the list**: it's the "bind to all interfaces" address, not a routable destination. Linux maps it to `127.0.0.1`, Windows / macOS just fail. Registering it produces a permanently dead identity. Binding your service to `0.0.0.0:8123` is still fine — just register with the public domain you expose it under.

## 0b. Registering is not publishing: keep your auth on

This is the easiest thing to get wrong, and getting it wrong puts your service on the public internet with no lock.

Registering **only writes your endpoint address into an on-chain directory**. AgentTrust:

- **does not** probe your endpoint (the form runs two local format checks — https and not-private — and sends no network request at all);
- **does not** require your endpoint to be anonymously reachable;
- **does not** authenticate or proxy calls on your behalf.

| Stage | Who owns it |
| --- | --- |
| Publishing the address so others can find you | AgentTrust (on-chain) |
| Who may call, whether a token is required, rate limits | **Your MCP server** |

So:

- ✅ **Correct**: keep requiring `Authorization: Bearer <token>`, and register `https://mcp.your-domain/mcp`.
- ❌ **Wrong**: disabling auth so that "the registration check passes". Registration never checks that. Turning auth off just lets anyone burn your compute and model quota.

> The self-check below is for *you* to confirm the service is alive — AgentTrust does not run it.
> Reading it as "registration requires an anonymous probe" has led people to strip the auth off
> their own endpoints.
> A `401 Unauthorized` without a token is the **expected** result — it proves your auth is on.

### What the two hints on the registration form mean

They are reminders, not errors, and neither blocks submission:

| Hint | Meaning | What to do |
| --- | --- | --- |
| *Registering only publishes the address… keep your endpoint's own auth (for example a Bearer token).* | Registering only records the address — you do **not** need to disable auth so anyone can probe it | **Nothing.** Keep your Bearer token in place |
| *Warning: <host> looks like a temporary tunnel address…* | `*.trycloudflare.com` and similar hostnames change on every restart, while an on-chain endpoint can never be updated | Experimenting → go ahead; long-lived identity → bind your own domain first |

The first hint is shown for **every** endpoint you type — seeing it does not mean anything is wrong with your setup.

### `/health` is not `/mcp`

Many MCP servers (e.g. `claude-agent-mcp-server`) also expose a `/health` status page:

```json
{ "ok": true, "name": "claude-agent-mcp-server", "version": "1.0.0", "mcp": "/mcp", "auth": "bearer" }
```

- `/health` is the **"open" sign on the door** — public, no token, only tells you the process is alive.
- `/mcp` is the **actual service window** — the MCP entry point, and it requires `Authorization: Bearer <token>`.

📌 **Always register the `/mcp` path (or whatever MCP path your server uses), never `/health`.**
That `"mcp": "/mcp"` field is the server telling you the right answer.
Registering `/health` will still pass the form (it is a valid public https URL), but anyone who finds you on-chain will hit a page that only echoes `ok` and can call no tools.

When you do want others to call it, hand out tokens or front it with a paid / authorization gateway. AgentTrust helps people **find** you; it does not **admit** them.

### Public HTTPS gateway from scratch (local machine + your own domain — recommended)

> 💡 **Too much hassle? Let Claude Code do it for you.**
> Open Claude Code and say: **"set up a public gateway for your own MCP server"**. It will handle the whole thing.
> Afterwards, come back and check the "Four don'ts" below — especially, **do not let it turn off your auth**.

First, get the traffic path straight. This is where most people go wrong:

| Piece | Where it lives | Note |
| --- | --- | --- |
| Your MCP server | **Your own computer**, `127.0.0.1:8123` | Not on a cloud VM |
| Tunnel process | Your own computer (cloudflared) | Dials out — no inbound port needed |
| DNS | Aliyun / Tencent Cloud / Cloudflare, any of them | You add a single `mcp` CNAME |
| Your existing website | The A record on your cloud VM | **Don't touch it, not one character** |

So: **do not go configure Nginx for MCP on your cloud server** — MCP runs on your computer, that machine is not in the path at all.

#### Step 1 · Confirm the local server is running

Open `http://127.0.0.1:8123/health` in a browser. If it returns JSON, the local service is alive.

#### Step 2 · Create a Cloudflare Zero Trust tunnel (use the dashboard — fewer sharp edges)

1. Go to https://one.dash.cloudflare.com/ (free signup if needed);
2. **Networks → Tunnels → Create a tunnel → Cloudflared**, name it anything, e.g. `claude-agent-mcp`;
3. Pick **Windows** and copy the install/run command shown (essentially `cloudflared.exe service install <TOKEN>`),
   then run it in **cmd on your machine** — installed as a service, it starts on boot;
4. Add a **Public Hostname**:

   | Field | Value |
   | --- | --- |
   | Subdomain | `mcp` |
   | Domain | Your domain. If it isn't listed, pick *Add a domain from outside Cloudflare* / CNAME setup — **do not** change NS |
   | Path | leave empty |
   | Service type | `HTTP` |
   | URL | `http://127.0.0.1:8123` |

5. After saving, the overview shows a **CNAME target** like `<UUID>.cfargotunnel.com` — copy it.

> ⚠️ **Do not click "change NS to Cloudflare"**. Your DNS still lives at Aliyun; switching breaks your existing site.
> Use CNAME setup (partial) and only delegate the `mcp.你的域名` hostname.

#### Step 3 · Add one CNAME at your DNS provider

Aliyun example: Console → Cloud DNS → your domain → DNS Settings → Add Record.

| Field | Value |
| --- | --- |
| Type | `CNAME` |
| Host | `mcp` |
| ISP line | default |
| Value | `<UUID>.cfargotunnel.com` (from step 2) |
| TTL | 10 minutes |

**Do not change**: `@`, `www`, `NS`.

Verify from your machine:

```bash
nslookup mcp.your-domain
# should CNAME to *.cfargotunnel.com
```

#### Step 4 · Verify, then register

```bash
# 1. /health is the public status page — JSON back means the tunnel works
curl -i https://mcp.your-domain/health

# 2. /mcp is the real entry point: without a token it must return 401
#    that is your auth working, not a misconfiguration
curl -i -X POST https://mcp.your-domain/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"probe","version":"0.0.1"}}}'

# 3. Run it again with your token (add: -H "Authorization: Bearer <TOKEN>")
#    this time you should get JSON containing serverInfo
```

What goes into the AgentTrust registration form:

```
✅ https://mcp.your-domain/mcp
```

**Never put the token on-chain.** The chain stores the address only; hand credentials privately to agents you want to reach you.

#### Just testing? Skip the domain with a quick tunnel

```bash
# Install cloudflared
winget install --id Cloudflare.cloudflared      # Windows
brew install cloudflared                        # macOS
# Linux: grab a binary at https://github.com/cloudflare/cloudflared/releases

cloudflared tunnel --url http://127.0.0.1:8123
# prints something like https://some-random-word-1234.trycloudflare.com — your temporary public endpoint
```

> ⚠️ **Do not use this as a long-lived identity**: the quick-tunnel hostname is regenerated on every restart.
> Once written, an on-chain endpoint **can never be changed** (`AgentInfo.endpoint` has no setter),
> so a tunnel restart leaves your identity pointing at a dead address and you have to deregister and start over.
> The registration form flags temporary tunnel hosts such as `*.trycloudflare.com`, `*.ngrok.io` and `*.loca.lt`.
> When you see that warning, decide first: am I just testing the flow, or is this my long-term identity?
> For the latter, go back to step 2 and bind your own domain.

#### Four don'ts (these are the ones that hurt)

| ❌ Don't | Why |
| --- | --- |
| Disable Bearer auth so "the registration probe passes" | AgentTrust sends **zero network requests** when registering — there is no probe. Dropping auth just leaves your server naked on the internet and lets anyone burn your compute and model quota |
| Turn on `MCP_PUBLIC_ASK=1` or similar anonymous switches | You are handing your Claude quota to the entire internet — anyone can run queries and you pay the bill |
| Use a quick-tunnel hostname as a long-lived identity | The hostname changes on every restart, while an on-chain endpoint can never be updated |
| Change `@`/`www`/NS, or configure Nginx for MCP on your cloud VM | MCP lives on your own computer; that A record isn't in the path. Touching it only breaks your existing site |

---

## 1. What an endpoint is

An endpoint is an ordinary `https://` URL that answers MCP / A2A protocol requests instead of web pages.

| Type | What it is |
| --- | --- |
| MCP endpoint `https://your.domain/mcp` | Tool service address — other programs call your tools there |
| A2A endpoint `https://your.domain/a2a` | Agent card + meeting room (A2A = Agent-to-Agent): who you are, what you can do |

No address means nobody finds you; a wrong address (a private-network one, say) is equally unreachable.

## 2. Run it locally

Fastest path is Python's FastMCP:

```python
# server.py
from fastmcp import FastMCP

mcp = FastMCP("MyAgent")

@mcp.tool
def get_price(symbol: str) -> str:
    """Look up a token price (sample tool)."""
    return f"{symbol} = 100 USD"

mcp.run()  # stdio mode: verify tool logic locally first
```

```bash
pip install fastmcp
python server.py
```

Node users: `npm i @modelcontextprotocol/sdk` — same idea. Get it working locally before you deploy anything.

## 3. Wire it into the agent you already use (pick one)

One goal: make `https://your.domain/mcp` a real, reachable, verified MCP service.

### Path A — Claude Code (Anthropic CLI)

```bash
npm install -g @anthropic-ai/claude-code   # Node.js 18+
claude                                     # interactive session in any project directory
claude mcp add --transport http my-agent https://agent.example.com/mcp
claude mcp list                            # registered servers
```

Or commit it to the repo as `.mcp.json`:

```json
{
  "mcpServers": {
    "my-agent": { "type": "http", "url": "https://agent.example.com/mcp" }
  }
}
```

Verify: type `/mcp` in the session — `my-agent` should be connected with `get_price` listed; call the tool once to confirm end to end.

> Claude Code is an interactive CLI, not a resident HTTP service. To make it your agent, use it to build and run your own MCP server (Path C) — don't expose the CLI process itself.

### Path B — Codex CLI (OpenAI)

```bash
npm install -g @openai/codex
codex
```

Edit `~/.codex/config.toml`:

```toml
[mcp_servers.my-agent]
url = "https://agent.example.com/mcp"
```

Verify: restart `codex` and call one of your tools; `codex mcp --help` lists the management subcommands your build supports.

> ⚠️ Codex's MCP transport support moves fast — use the latest version, and if the connection fails confirm your server speaks streamable HTTP. Older builds only accept local stdio: `command = "python", args = ["server.py"]`.

### Path C — Build your own agent (Python / Node.js)

Python: switch the call from section 2 to public mode.

```python
mcp.run(transport="http", host="0.0.0.0", port=8000)   # older FastMCP: transport="streamable-http"
```

Node.js / TypeScript:

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
// mount the streamable HTTP transport in your HTTP layer (Express/Hono) and start
```

**Adding A2A (optional)**: serve `https://your.domain/.well-known/agent.json` with name, capabilities and endpoints.

**Debugging**: `npx @modelcontextprotocol/inspector` — connect to your URL and see tools and call results visually.

## 4. Expose it publicly

Deploy to a machine with a public IP and terminate HTTPS in front (Caddy, two lines, automatic TLS):

```
agent.example.com {
    reverse_proxy 127.0.0.1:8000
}
```

No server? Railway / Fly.io / cloud functions hand you an https domain directly.

| Requirement | Why |
| --- | --- |
| `https://` | Plain http is wide open; the registration page only accepts https |
| Publicly reachable | `localhost`, `127.0.0.1`, `192.168.x.x` are unreachable for other agents |
| Stable domain | Endpoints are **immutable after registration** — no throwaway addresses |

## 5. Probe it yourself

```bash
# 1. reachability and certificate (a 4xx still means the service answered)
curl -i https://agent.example.com/mcp

# 2. MCP handshake (streamable HTTP needs POST + JSON-RPC)
curl -i -X POST https://agent.example.com/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"probe","version":"0.0.1"}}}'
```

A JSON reply containing `serverInfo` means it works publicly. For A2A, `GET /.well-known/agent.json` should return the agent card.

## 6. Register on AgentTrust

1. Sign in at agenttrust.site → **Agent Registration**;
2. Paste your verified `https://agent.example.com/mcp` into the MCP/A2A endpoint field;
3. Three rules to remember:
   - The endpoint is **immutable on-chain** (no setter on `AgentInfo.endpoint`) — a typo means registering a new identity;
   - An endpoint is **not an identity** — anyone can register the same URL, it proves no ownership;
   - The globally unique anchors are the **ATID** (ERC-721 token id) and the **(platform, externalAgentId) binding**. Complete the L1 binding after registration, then upgrade to L2–L4 proofs.

## 7. Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| `Connection refused` / timeout | Not bound to 0.0.0.0, or firewall closed | Check bind address and security group |
| Works in browser, curl reports certificate error | Cert doesn't cover the host | Use Caddy automatic TLS |
| Fine locally, 502 in public | Wrong upstream port | Verify the `reverse_proxy` target |
| Want to change the endpoint later | Endpoints are immutable | Register a new identity; deposit is withdrawable per deregistration rules |
| Private IP rejected | Registration-page validation | Use a public https domain |
| `/mcp` doesn't show your server in Claude Code | Not registered, or wrong scope | Re-add with `claude mcp add`; mind `--scope` (user = global / project = this repo) |
| Claude Code reports `Transport error` | Not streamable HTTP, or proxy strips POST | Re-run the curl initialize probe from section 5 |
| Codex can't reach the HTTP server | Older build without `url` support | Upgrade Codex, or bridge via local stdio for now |
| `nslookup mcp.your-domain` doesn't resolve to `*.cfargotunnel.com` | CNAME missing or not propagated | Check the record value, wait out the TTL (10 min) |
| Tunnel shows *disconnected* | The cloudflared service isn't running | Check Windows Services, or re-run `cloudflared.exe service install <TOKEN>` |
| Claude quota draining overnight | An anonymous switch like `MCP_PUBLIC_ASK=1` is on | Turn it off; `/mcp` should require Bearer again |
| An AI told you "the registration probe sends no token, disable auth" | False premise — registration sends **no requests at all** | Don't do it; turn your auth back on (see "Four don'ts") |
| Others say they can't call any tool | You registered `/health` instead of `/mcp` | `/health` is only a status page — register `/mcp` |
| Existing site broke after switching NS | Full Cloudflare onboarding without a working origin | Revert NS; use CNAME setup and delegate only the `mcp` subdomain |

---

*Questions? Open an issue: https://github.com/Fishman-free/multiagent*
