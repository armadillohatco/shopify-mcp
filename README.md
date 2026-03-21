# Shopify MCP Server

A [Model Context Protocol (MCP)](https://modelcontextprotocol.io) server that connects Claude (or any MCP-compatible AI) directly to your Shopify store. Manage products, orders, customers, collections, inventory, fulfillments, and webhooks — all through natural language.

---

## What this does

Once connected, you can talk to your Shopify store like this:

- *"Show me all unfulfilled orders from today"*
- *"Create a new product called Summer T-Shirt, price €29.99, status draft"*
- *"How many active products do we have?"*
- *"Update the inventory for product X to 50 units at location Y"*

---

## Requirements

- Python 3.11+
- A Shopify store (any plan)
- A Shopify Custom App with Admin API access (see setup below)
- Claude.ai Pro/Team/Enterprise **or** any MCP-compatible client

---

## Step 1 — Get your Shopify credentials

You need two things: your **store name** and an **access token**.

### 1a. Find your store name

Your store name is the part before `.myshopify.com`. If your URL is `acme-store.myshopify.com`, your store name is `acme-store`.

### 1b. Create a Custom App and get your Access Token

> ⚠️ **Important:** Shopify's regular API key will NOT work here. You need an **Admin API access token** from a Custom App. Follow these steps exactly.

1. Go to your **Shopify Admin** → `Settings` → `Apps and sales channels`
2. Click **Develop apps** (top right corner)
3. If prompted, click **Allow custom app development**
4. Click **Create an app** and give it a name (e.g. `MCP Server`)
5. Go to **Configuration** → click **Configure Admin API scopes**
6. Select the permissions you need. For full access, enable:
   - `read_products`, `write_products`
   - `read_orders`, `write_orders`
   - `read_customers`, `write_customers`
   - `read_inventory`, `write_inventory`
   - `read_fulfillments`, `write_fulfillments`
   - `read_webhooks`, `write_webhooks`
7. Click **Save**
8. Go to the **API credentials** tab → click **Install app**
9. Click **Reveal token once** — copy this token immediately, it starts with `shpat_...`

> 💡 You can only see this token once. If you lose it, you'll need to uninstall and reinstall the app to generate a new one.

---

## Step 2 — Set up the server

### Clone the repo

```bash
git clone https://github.com/daanjonk/shopify-mcp.git
cd shopify-mcp
```

### Install dependencies

```bash
pip install -r requirements.txt
```

### Configure your environment variables

Copy the example file and fill in your values:

```bash
cp .env.example .env
```

Open `.env` and fill in:

```env
SHOPIFY_STORE=your-store-name       # e.g. acme-store (NOT acme-store.myshopify.com)
SHOPIFY_ACCESS_TOKEN=shpat_xxxx...  # The token you copied in Step 1
```

That's it — the other values have sensible defaults.

### Start the server

```bash
python server.py
```

You should see:

```
INFO  Token mode: static SHOPIFY_ACCESS_TOKEN (no auto-refresh)
INFO  Uvicorn running on http://0.0.0.0:8000
```

Your MCP server is now running at `http://localhost:8000/mcp`.

---

## Step 3 — Deploy to the cloud (for remote access)

To use this with Claude.ai, the server needs to be publicly accessible on the internet. The easiest option is [Railway](https://railway.app) — free tier works fine.

### Deploy on Railway

1. Go to [railway.app](https://railway.app) and sign in with GitHub
2. Click **New Project** → **Deploy from GitHub repo**
3. Select your forked `shopify-mcp` repo
4. Railway will detect the `Dockerfile` and start building
5. Once deployed, go to **Settings** → **Networking** → **Generate Domain**
6. Copy your public URL (e.g. `https://shopify-mcp-production.up.railway.app`)

### Set environment variables on Railway

In your Railway project, go to **Variables** and add:

| Variable | Value |
|---|---|
| `SHOPIFY_STORE` | `your-store-name` |
| `SHOPIFY_ACCESS_TOKEN` | `shpat_xxxx...` |
| `PORT` | `8000` |
| `MCP_TRANSPORT` | `streamable-http` |

Railway will restart the server automatically after saving.

---

## Step 4 — Connect to Claude

### Get the MCP server URL

Your MCP endpoint is your Railway URL + `/mcp`:

```
https://your-app.up.railway.app/mcp
```

### Add the MCP server in Claude.ai

> ⚠️ **Authentication token required.** Claude.ai requires you to set up an authentication token when connecting to a remote MCP server. This is separate from your Shopify token — it protects your MCP server from unauthorized access.

**To add the server in Claude:**

1. Go to [claude.ai](https://claude.ai) → click your profile (bottom left) → **Settings**
2. Go to **Integrations** (or **MCP Servers** depending on your plan)
3. Click **Add integration** / **Add MCP server**
4. Fill in:
   - **Name:** `Shopify`
   - **URL:** `https://your-app.up.railway.app/mcp`
5. If prompted for an **authentication token**, you have two options:

   **Option A — No auth (development only):** Leave blank if your server has no authentication middleware. Only do this if your Railway app URL is not publicly shared.

   **Option B — Add auth to your server:** Set a `BEARER_TOKEN` environment variable in Railway, then add authentication middleware to `server.py` (see below). Then paste that same token into Claude's integration settings.

### Adding bearer token authentication (recommended for production)

Add these lines to `server.py` right after the FastMCP initialization:

```python
import secrets
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.responses import Response

BEARER_TOKEN = os.environ.get("BEARER_TOKEN", "")

class BearerAuthMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        if BEARER_TOKEN:
            auth = request.headers.get("Authorization", "")
            if auth != f"Bearer {BEARER_TOKEN}":
                return Response("Unauthorized", status_code=401)
        return await call_next(request)

mcp.app.add_middleware(BearerAuthMiddleware)
```

Then add `BEARER_TOKEN=your-secret-token-here` to your Railway environment variables, and paste the same value into Claude's integration settings as the authentication token.

---

## Available tools

Once connected, Claude has access to these tools:

| Tool | Description |
|---|---|
| `shopify_list_products` | List products with filters |
| `shopify_get_product` | Get a single product by ID |
| `shopify_create_product` | Create a new product |
| `shopify_update_product` | Update an existing product |
| `shopify_delete_product` | Delete a product |
| `shopify_count_products` | Count products |
| `shopify_list_orders` | List orders with filters |
| `shopify_get_order` | Get a single order by ID |
| `shopify_count_orders` | Count orders |
| `shopify_close_order` | Close an order |
| `shopify_cancel_order` | Cancel an order |
| `shopify_list_customers` | List customers |
| `shopify_search_customers` | Search customers |
| `shopify_get_customer` | Get a single customer |
| `shopify_create_customer` | Create a new customer |
| `shopify_update_customer` | Update a customer |
| `shopify_get_customer_orders` | Get all orders for a customer |
| `shopify_list_collections` | List custom or smart collections |
| `shopify_get_collection_products` | Get products in a collection |
| `shopify_list_locations` | List inventory locations |
| `shopify_get_inventory_levels` | Get inventory levels |
| `shopify_set_inventory_level` | Set inventory quantity |
| `shopify_list_fulfillments` | List fulfillments for an order |
| `shopify_create_fulfillment` | Fulfill an order |
| `shopify_get_shop` | Get store info |
| `shopify_list_webhooks` | List webhooks |
| `shopify_create_webhook` | Create a webhook |

---

## Environment variables reference

| Variable | Required | Default | Description |
|---|---|---|---|
| `SHOPIFY_STORE` | ✅ Yes | — | Your store name (e.g. `my-store`) |
| `SHOPIFY_ACCESS_TOKEN` | ✅ Yes* | — | Static Admin API access token (`shpat_...`) |
| `SHOPIFY_CLIENT_ID` | No | — | OAuth client ID (alternative to static token) |
| `SHOPIFY_CLIENT_SECRET` | No | — | OAuth client secret (alternative to static token) |
| `SHOPIFY_API_VERSION` | No | `2024-10` | Shopify API version |
| `PORT` | No | `8000` | Server port |
| `MCP_TRANSPORT` | No | `streamable-http` | Transport protocol |
| `BEARER_TOKEN` | No | — | Auth token to protect your MCP endpoint |

*Either `SHOPIFY_ACCESS_TOKEN` **or** `SHOPIFY_CLIENT_ID` + `SHOPIFY_CLIENT_SECRET` is required.

---

## Troubleshooting

**"Authentication failed" error**
→ Double-check your `SHOPIFY_ACCESS_TOKEN`. Make sure it starts with `shpat_` and that the Custom App is installed on your store.

**"Permission denied" / 403 error**
→ Your token doesn't have the required API scopes. Go back to your Custom App in Shopify Admin, add the missing scopes, and reinstall the app (this generates a new token).

**"Missing SHOPIFY_STORE environment variable"**
→ Make sure `SHOPIFY_STORE` is set and contains just the store name, not the full URL. ✅ `my-store` ❌ `my-store.myshopify.com`

**Claude can't connect to the server**
→ Make sure your Railway deployment is active and the domain is generated. Test the endpoint by visiting `https://your-app.up.railway.app/mcp` in your browser — you should get a response (not a 404).

**Token shows once then disappears in Shopify**
→ That's normal. Shopify only shows the token once. If you lost it, uninstall the app in Shopify Admin → API credentials → Uninstall, then reinstall to generate a new token.

---

## License

MIT
