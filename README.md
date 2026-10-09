# OTTO Market MCP Server

**English** · [Deutsch](docs/README.de.md)

**Connect OTTO Market to Claude, ChatGPT and Copilot: orders, products, returns, stock and price updates as MCP tools.** Powered by [AnythingMCP](https://github.com/HelpCode-ai/anythingmcp).

OTTO Market MCP Server gives Claude, ChatGPT, Copilot and Cursor 8 tools for OTTO Market: orders, products, returns, stock and price updates. 6 tools read and 2 can change data. It runs on AnythingMCP: one click on AnythingMCP Cloud, or self-hosted with Docker. Credentials are stored encrypted and every call is audited.

**Status:** not yet verified against a live system. The adapter follows the vendor's API documentation; please report what you find.  
**Adapter synced:** <!-- synced -->2026-10-09

Maintained by [KOCH Freiburg GmbH](https://www.kochfreiburg.de/), which runs AnythingMCP in production. Built on [AnythingMCP](https://github.com/HelpCode-ai/anythingmcp) by helpcode.ai.

## Quick start (AnythingMCP Cloud)

1. Sign in at [cloud.anythingmcp.com](https://cloud.anythingmcp.com) and open the [install link](https://cloud.anythingmcp.com/connectors/store?install=otto-market).
2. Enter `OTTO_USERNAME`, `OTTO_PASSWORD` (see [Authentication](#authentication)).
3. Copy the URL of your MCP server under **MCP Servers** and add it to your AI client ([below](#connect-claude-chatgpt-copilot-or-cursor)).

AnythingMCP Cloud is the same open-source code, operated by helpcode.ai in Frankfurt, Germany.

## Self-hosted (Docker)

Needs Docker 24+, openssl and Node 18+.

```bash
git clone https://github.com/kochfreiburg/otto-market-mcp-server.git
cd otto-market-mcp-server
./scripts/install.sh
```

`install.sh` writes `.env` with fresh secrets, starts AnythingMCP, creates the first admin, installs the connector if `OTTO_USERNAME` and `OTTO_PASSWORD` are set in `.env` and creates an MCP API key. Without credentials it prints the install link instead: `http://localhost:3000/connectors/store?install=otto-market`. Then check the whole chain:

```bash
npm install && node scripts/smoke.mjs
```

## Connect Claude, ChatGPT, Copilot or Cursor

- **Claude (claude.ai, Desktop, mobile):** *Customize → Connectors → Add custom connector*, paste your MCP server URL and sign in. Claude connects from Anthropic's cloud, so the URL must be public HTTPS: your AnythingMCP Cloud URL, or your own instance behind TLS.
- **Claude Code:**

  ```bash
  claude mcp add --transport http otto-market-mcp-server http://localhost:4000/mcp --header "X-API-Key: <MCP_API_KEY>"
  ```
- **Cursor** (`.cursor/mcp.json`) and **VS Code / GitHub Copilot** (`.vscode/mcp.json`, key `servers` instead of `mcpServers`, plus `"type": "http"`):

  ```json
  { "mcpServers": { "otto-market-mcp-server": { "url": "http://localhost:4000/mcp", "headers": { "X-API-Key": "<MCP_API_KEY>" } } } }
  ```
- **ChatGPT:** add the public HTTPS URL as a connector (app) in ChatGPT's settings. A `localhost` URL does not work there.

## Tools

8 tools, generated from [`adapter/otto-market.json`](adapter/otto-market.json). **read** tools cannot change anything in the source system.

<!-- tools:start (generated from adapter/*.json, do not edit) -->
| Tool | What it does | Access |
|---|---|---|
| `otto_market_list_orders` | List orders with their positions, buyer, delivery address and fulfilment status, optionally from/to an order date. | read |
| `otto_market_get_order` | Read one order in full: every position with SKU, price and status, the delivery and invoice addresses, and the payment method. | read |
| `otto_market_list_products` | List the seller's product variations with their SKU, EAN, product reference and category, optionally filtered. | read |
| `otto_market_get_product` | Read one product variation by SKU: its attributes, category and media. | read |
| `otto_market_list_quantities` | Read the current stock quantities OTTO holds for the seller's SKUs (up to 200 per page), so a discrepancy with the ERP can be spotted. | read |
| `otto_market_list_returns` | List returned position items by return status, with their SKU, reason and the order they belong to — the input to any returns-rate question. | read |
| `otto_market_update_quantity` | Set the available stock for one SKU, as OTTO's availability interface holds it. | write |
| `otto_market_update_price` | Set the standard price for one SKU. | write |
<!-- tools:end -->

## Example prompts

- Which OTTO orders since Monday are not shipped yet?
- Show order 1234567 with every position and the delivery address.
- Which returns came in this week, and what were the reasons?
- What stock does OTTO show for SKU DR-1001, and does it match our ERP?
- Which of my products are not live on otto.de, and why?
- Set the stock of SKU LK-2040 to 50 on OTTO. (write)

More in [examples/prompts.md](examples/prompts.md).

## Authentication

**Getting credentials**
1. You must already be an OTTO Market partner. In the **OTTO Partner Connect** portal open the API area and create API credentials for your account.
2. OTTO issues a **username and password** for the token endpoint rather than a classic client id/secret pair; the token request is a password grant against `https://api.otto.market/v1/token`.
3. Set `OTTO_USERNAME` and `OTTO_PASSWORD`. AnythingMCP exchanges them for an access token and refreshes it automatically.

**Every endpoint is versioned separately.** OTTO versions per resource, not per API: orders are on `/v4/orders`, quantities on `/v1/quantities`, prices on `/v3/prices`. The tools below carry the version each resource is actually on — do not assume one prefix works everywhere.

**Orders page with a cursor.** `otto_market_list_orders` returns `nextCursor`; pass it back as `nextCursor` to walk the list. Do not compute an offset; OTTO's order stream is append-only and offsets drift.

**`fromOrderDate` is required and is a hard filter.** Asking without it returns the default window, which is much shorter than most people expect. State the date you mean.

**Stock and price are eventually consistent.** A quantity update is accepted with a 202 and applied asynchronously; reading the value back immediately will often still show the old number. That is OTTO's design, not a failed write.

**Unverified.** The paths and field names below follow the vendor's published documentation and have not been exercised against a live tenant. Treat a 404 as a docs-versus-reality gap and check the developer portal before concluding that the credential is wrong. A correction from anyone running this against a real system is genuinely welcome.

**Writes**: `otto_market_update_quantity` and `otto_market_update_price` change what shoppers on otto.de see.

## Security

- **Read or write is your choice.** 6 of the tools only read; `otto_market_update_quantity`, `otto_market_update_price` can change data. Assign the connector to an MCP server whose role whitelists only the tools you want, and the rest are invisible to that client.
- **Credentials** are encrypted with AES-256-GCM and never shown to the model.
- **Response mapping** drops or reshapes fields per tool before they reach the model, e.g. bank details or personal data.
- **Audit log:** every call is recorded with input, output, duration and status, in your own database when self-hosted.
- **SSO, RBAC and SCIM** are included in the self-hosted build.

## FAQ

### Is there an OTTO Market MCP server?
Yes, this one. It connects the OTTO Market partner API to Claude, ChatGPT and Copilot through AnythingMCP: 8 tools for orders, products, stock, returns, and stock and price updates.

### What do I need to connect it?
An OTTO Market partner account and API credentials from OTTO Partner Connect. OTTO issues a username and password for its token endpoint; AnythingMCP exchanges them for an access token and renews it.

### Can the AI change stock or prices?
Yes, two tools can: `otto_market_update_quantity` and `otto_market_update_price`. OTTO applies both asynchronously, so a read straight afterwards may still show the old value. Leave them off the MCP server's role if the AI should only read.

### Has this been tested against a live OTTO account?
Not yet. The adapter follows OTTO's published API documentation. If you run it against your account, please report what works and what doesn't.

## Troubleshooting

| Problem | Fix |
|---|---|
| `401` / `403` from the vendor | The credentials are wrong or lack rights. Re-enter them on the connector page; the import runs a test call and shows the result. |
| Tools missing in the AI client | The connector is not assigned to the MCP server the client uses. Check **MCP Servers**, then run `node scripts/smoke.mjs`. |
| The host is on your internal network | Self-host AnythingMCP on that network and add the hostname to `SSRF_ALLOWED_HOSTS`, or the outbound guard blocks the call. |
| Works locally, fails on AnythingMCP Cloud | The system must be reachable from the internet with a valid TLS certificate. |

## Related

- [ecommerce-mcp-server](https://github.com/HelpCode-ai/ecommerce-mcp-server): E-commerce MCP server: connect Amazon, eBay, WooCommerce, Shopware, Kaufland, OTTO and 7 more to Claude & ChatGPT.
- [kaufland-mcp-server](https://github.com/kochfreiburg/kaufland-mcp-server): Kaufland Marketplace MCP server: Claude & ChatGPT read your Kaufland seller orders, units, shipments, tickets and storefronts.
- [billbee-mcp-server](https://github.com/kochfreiburg/billbee-mcp-server): Billbee MCP server: connect Billbee order management to Claude & ChatGPT. Orders, products, customers and shipping providers.
- [AnythingMCP](https://github.com/HelpCode-ai/anythingmcp): the open-source MCP server and gateway this repository is built on.

## License

AGPL-3.0-only. The adapter definition in `adapter/` comes from AnythingMCP (AGPL-3.0).
