# OTTO Market MCP Server

[English](../README.md) · **Deutsch**

**Verbinde OTTO Market mit Claude, ChatGPT und Copilot: orders, products, returns, stock and price updates als MCP-Tools.** Basiert auf [AnythingMCP](https://github.com/HelpCode-ai/anythingmcp).

OTTO Market MCP Server gibt Claude, ChatGPT, Copilot und Cursor 8 Tools für OTTO Market: orders, products, returns, stock and price updates. 6 Tools lesen, 2 können Daten ändern. Es läuft auf AnythingMCP: mit einem Klick in AnythingMCP Cloud oder selbst gehostet mit Docker. Zugangsdaten werden verschlüsselt gespeichert, jeder Aufruf landet im Audit-Log.

**Status:** noch nicht gegen ein Live-System geprüft. Der Adapter folgt der API-Dokumentation des Herstellers; Rückmeldungen sind willkommen.  
**Adapter synchronisiert:** <!-- synced -->2026-09-26

Maintained by [KOCH Freiburg GmbH](https://www.kochfreiburg.de/), which runs AnythingMCP in production. Built on [AnythingMCP](https://github.com/HelpCode-ai/anythingmcp) by helpcode.ai.

## Schnellstart (AnythingMCP Cloud)

1. Melde dich bei [cloud.anythingmcp.com](https://cloud.anythingmcp.com) an und öffne den [Installationslink](https://cloud.anythingmcp.com/connectors/store?install=otto-market).
2. Trage `OTTO_USERNAME`, `OTTO_PASSWORD` ein (siehe [Authentifizierung](#authentifizierung)).
3. Kopiere die URL deines MCP-Servers unter **MCP Servers** und füge sie in deinen KI-Client ein ([siehe unten](#claude-chatgpt-copilot-oder-cursor-verbinden)).

AnythingMCP Cloud ist derselbe Open-Source-Code, betrieben von helpcode.ai in Frankfurt.

## Selbst gehostet (Docker)

Benötigt Docker 24+, openssl und Node 18+.

```bash
git clone https://github.com/kochfreiburg/otto-market-mcp-server.git
cd otto-market-mcp-server
./scripts/install.sh
```

`install.sh` schreibt `.env` mit neuen Secrets, startet AnythingMCP, legt den ersten Admin an, installiert den Connector, sofern `OTTO_USERNAME` und `OTTO_PASSWORD` in `.env` gesetzt sind, und erzeugt einen MCP-API-Key. Ohne Zugangsdaten gibt es stattdessen den Installationslink aus: `http://localhost:3000/connectors/store?install=otto-market`. Danach die ganze Kette prüfen:

```bash
npm install && node scripts/smoke.mjs
```

## Claude, ChatGPT, Copilot oder Cursor verbinden

- **Claude (claude.ai, Desktop, Mobil):** *Customize → Connectors → Add custom connector*, MCP-Server-URL einfügen und anmelden. Claude verbindet sich aus der Cloud von Anthropic, die URL muss also öffentlich per HTTPS erreichbar sein: deine AnythingMCP-Cloud-URL oder deine eigene Instanz mit TLS.
- **Claude Code:**

  ```bash
  claude mcp add --transport http otto-market-mcp-server http://localhost:4000/mcp --header "X-API-Key: <MCP_API_KEY>"
  ```
- **Cursor** (`.cursor/mcp.json`) und **VS Code / GitHub Copilot** (`.vscode/mcp.json`, Schlüssel `servers` statt `mcpServers`, dazu `"type": "http"`):

  ```json
  { "mcpServers": { "otto-market-mcp-server": { "url": "http://localhost:4000/mcp", "headers": { "X-API-Key": "<MCP_API_KEY>" } } } }
  ```
- **ChatGPT:** die öffentliche HTTPS-URL in den ChatGPT-Einstellungen als Connector (App) hinzufügen. Eine `localhost`-URL funktioniert dort nicht.

## Tools

8 Tools, erzeugt aus [`adapter/otto-market.json`](../adapter/otto-market.json). Tools mit **lesen** können im Quellsystem nichts ändern.

<!-- tools:start (generated from adapter/*.json, do not edit) -->
| Tool | Funktion | Zugriff |
|---|---|---|
| `otto_market_list_orders` | List orders from a date onwards, with their positions, buyer, delivery address and fulfilment status. | lesen |
| `otto_market_get_order` | Read one order in full: every position with SKU, price and status, the delivery and invoice addresses, and the payment method. | lesen |
| `otto_market_list_products` | List the seller's product variations with their SKU, EAN, product reference and current market status on otto.de. | lesen |
| `otto_market_get_product` | Read one product variation by SKU: its attributes, category, media and the current status of its listing on otto.de. | lesen |
| `otto_market_list_quantities` | Read the current stock quantities OTTO holds for the seller's SKUs, so a discrepancy with the ERP can be spotted. | lesen |
| `otto_market_list_returns` | List returns with their SKU, quantity, reason and the order they belong to — the input to any returns-rate question. | lesen |
| `otto_market_update_quantity` | Set the available stock for one SKU. | schreiben |
| `otto_market_update_price` | Set the price for one SKU. | schreiben |
<!-- tools:end -->

## Beispiel-Prompts

- Welche OTTO-Bestellungen seit Montag sind noch nicht versendet?
- Welche Retouren kamen diese Woche, und mit welchen Gründen?
- Welchen Bestand zeigt OTTO für SKU DR-1001, und passt er zu unserem ERP?
- Welche meiner Produkte sind auf otto.de nicht online, und warum?
- Setze den Bestand von SKU LK-2040 bei OTTO auf 50. (schreibend)
- Ändere den Preis von SKU DR-1002 auf 229,90 EUR. (schreibend)

Weitere (auf Englisch) in [examples/prompts.md](../examples/prompts.md).

## Authentifizierung

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

## Sicherheit

- **Lesen oder schreiben entscheidest du.** 6 der Tools lesen nur; `otto_market_update_quantity`, `otto_market_update_price` können Daten ändern. Weise den Connector einem MCP-Server zu, dessen Rolle nur die gewünschten Tools freigibt; die anderen sieht dieser Client gar nicht.
- **Zugangsdaten** werden mit AES-256-GCM verschlüsselt und nie an das Modell gegeben.
- **Response-Mapping** entfernt oder formt Felder pro Tool, bevor sie das Modell erreichen, etwa Bankdaten oder personenbezogene Daten.
- **Audit-Log:** Jeder Aufruf wird mit Eingabe, Ausgabe, Dauer und Status protokolliert, selbst gehostet in deiner eigenen Datenbank.
- **SSO, RBAC und SCIM** sind in der selbst gehosteten Version enthalten.

## FAQ

### Gibt es einen MCP-Server für OTTO Market?
Ja, diesen hier. Er verbindet die Partner-API von OTTO Market über AnythingMCP mit Claude, ChatGPT und Copilot: 8 Tools für Bestellungen, Produkte, Bestände, Retouren sowie Bestands- und Preisänderungen.

### Was brauche ich für die Verbindung?
Ein OTTO-Market-Partnerkonto und API-Zugangsdaten aus OTTO Partner Connect (Benutzername und Passwort für den Token-Endpunkt).

### Kann die KI Bestände oder Preise ändern?
Ja, zwei Tools können das. OTTO übernimmt Änderungen asynchron. Nimm sie aus der Rolle des MCP-Servers, wenn die KI nur lesen soll.

### Ist das mit einem echten OTTO-Konto getestet?
Noch nicht. Der Adapter folgt der veröffentlichten API-Dokumentation; Rückmeldungen aus dem Echtbetrieb sind willkommen.

## Fehlerbehebung

| Problem | Lösung |
|---|---|
| `401` / `403` vom Hersteller | Zugangsdaten falsch oder ohne Rechte. Auf der Connector-Seite neu eintragen; der Import macht einen Testaufruf und zeigt das Ergebnis. |
| Tools fehlen im KI-Client | Der Connector ist nicht dem MCP-Server zugewiesen, den der Client nutzt. **MCP Servers** prüfen, dann `node scripts/smoke.mjs` ausführen. |
| Das System steht im internen Netz | AnythingMCP in diesem Netz selbst hosten und den Hostnamen in `SSRF_ALLOWED_HOSTS` eintragen, sonst blockiert der Outbound-Guard den Aufruf. |
| Lokal ok, in AnythingMCP Cloud nicht | Das System muss aus dem Internet mit gültigem TLS-Zertifikat erreichbar sein. |

## Verwandte Repositories

- [ecommerce-mcp-server](https://github.com/HelpCode-ai/ecommerce-mcp-server): E-commerce MCP server: connect Amazon, eBay, WooCommerce, Shopware, Kaufland, OTTO and 7 more to Claude & ChatGPT.
- [kaufland-mcp-server](https://github.com/kochfreiburg/kaufland-mcp-server): Kaufland Marketplace MCP server: Claude & ChatGPT read your Kaufland seller orders, units, shipments, tickets and storefronts.
- [billbee-mcp-server](https://github.com/kochfreiburg/billbee-mcp-server): Billbee MCP server: connect Billbee order management to Claude & ChatGPT. Orders, products, customers and shipping providers.
- [AnythingMCP](https://github.com/HelpCode-ai/anythingmcp): der Open-Source-MCP-Server und -Gateway, auf dem dieses Repository aufbaut.

## Lizenz

AGPL-3.0-only. Die Adapter-Definition in `adapter/` stammt aus AnythingMCP (AGPL-3.0).
