---
name: build-arcade-mcp-tool
description: Build and deploy custom MCP tools using the Arcade MCP framework. Use when the user wants to create, build, scaffold, or deploy an MCP tool or server to Arcade Cloud, or when working with arcade_mcp_server, MCPApp, @tool decorators, arcade deploy, or Arcade tool development.
---

# Build and Deploy Custom Arcade MCP Tools

## What is Arcade?

Arcade is the MCP runtime for AI agents. It provides secure agent authorization, tool hosting, and centralized governance so you can ship production-grade tools without building auth infrastructure yourself.

**What Arcade handles for you:**
- **OAuth flows**: Just-in-time authorization -- Arcade manages the entire OAuth lifecycle (consent, token issuance, refresh, storage) with zero code from you
- **Secrets management**: API keys and credentials are injected at runtime via `Context`, never exposed to LLMs or clients
- **Multi-user support**: When deployed, each user gets their own auth session automatically
- **Tool hosting**: Deploy with `arcade deploy` and Arcade runs your MCP server in the cloud with health checks, scaling, and monitoring
- **Built-in auth providers**: Google, Slack, GitHub, Reddit, and more work out of the box -- no need to register OAuth apps or manage client credentials

**Development model**: Build tools locally with `arcade_mcp_server` -> test with stdio/HTTP -> deploy to Arcade Cloud with `arcade deploy`. For deeper platform context, fetch https://docs.arcade.dev/llms.txt

---

## Before You Begin

Gather these decisions from the user before writing any code:

1. **Integration target**: What API, service, or system will this tool connect to?
2. **Auth type**: OAuth (user-delegated), API Key / Secrets, Both, or None?
3. **OAuth provider** (if OAuth): Google, Slack, GitHub, Reddit, or custom OAuth2?
4. **Scopes** (if OAuth): What permissions does the tool need?
5. **Secrets** (if API key): What secret names are needed (e.g., `SERVICE_API_KEY`)?
6. **Language**: Python (recommended, primary support) or TypeScript?

Use the AskQuestion tool if available to ask about integration target, auth type (options: "OAuth", "API Key / Secrets", "Both", "No auth"), and OAuth provider if applicable.

---

## Step 1: Scaffold the Project

```bash
uv tool install arcade-mcp
arcade login
arcade new my_server
cd my_server
```

This generates:

```
my_server/
├── src/
│   └── my_server/
│       ├── __init__.py
│       ├── .env.example
│       └── server.py
└── pyproject.toml
```

---

## Step 2: Choose Project Structure

**Simple (1-3 tools)** -- keep tools in `server.py` using `@app.tool`:

```
src/my_server/
├── server.py       # MCPApp + tool definitions + entrypoint
└── .env
```

**Production (4+ tools)** -- organize into a package:

```
src/my_server/
├── __init__.py         # Re-exports: from my_server.tools import *
├── server.py           # MCPApp + entrypoint only
├── client.py           # API client wrapper class
├── constants.py        # Limits, defaults, config values
├── tools/
│   ├── __init__.py     # Explicit exports of all tool functions
│   ├── queries.py      # Read-only tools (Query Tools)
│   └── commands.py     # Tools with side effects (Command Tools)
├── models/
│   ├── enums.py        # Enum types for constrained inputs
│   └── outputs.py      # TypedDict output models
└── utils/
    └── helpers.py      # Shared helper functions
```

### Module Export Pattern

```python
# tools/__init__.py -- explicit exports
from my_server.tools.queries import list_items, get_item
from my_server.tools.commands import create_item, delete_item
__all__ = ["list_items", "get_item", "create_item", "delete_item"]

# Package __init__.py -- re-exports
from my_server.tools import *  # noqa: F401, F403
from my_server.tools import __all__
```

---

## Step 3: Implement Tools

### Core Imports

```python
from typing import Annotated
from arcade_mcp_server import Context, MCPApp, tool
```

### Auth Imports

```python
# Built-in OAuth providers
from arcade_mcp_server.auth import Google, Slack, GitHub, Reddit

# Custom OAuth2 provider
from arcade_mcp_server.auth import OAuth2
```

### Error Imports

```python
from arcade_mcp_server.exceptions import RetryableToolError, ToolExecutionError
```

### Tool Function Signature

Every tool MUST follow this exact pattern:

```python
@tool(
    requires_auth=ProviderClass(scopes=["scope1", "scope2"]),
    # OR requires_secrets=["SECRET_NAME"],
    # OR both
)
async def my_tool_name(
    context: Context,
    required_param: Annotated[str, "Clear description for the LLM"],
    optional_param: Annotated[int, "Description with constraints"] = 10,
    enum_param: Annotated[MyEnum, "Constrained choices"] = MyEnum.DEFAULT,
) -> Annotated[OutputType, "Description of the return value"]:
    """Concise, LLM-optimized description of what this tool does."""
    ...
```

### Mandatory Rules

1. **Always `async def`** for all tool functions
2. **`Context` is always the first parameter** -- never omit it for tools that need auth/secrets
3. **`Annotated[Type, "description"]`** on every parameter AND return type
4. **Docstrings are for the LLM** -- write them for machine comprehension, not humans
5. **Return structured dicts or TypedDicts** -- flat, relevant fields only
6. **Never accept secrets as parameters** -- use `context.get_secret()` instead
7. **`app.run()` must be inside `if __name__ == "__main__":`** -- required for deployment

---

## Step 4: Authentication Patterns

### How Arcade OAuth Works (Just-in-Time Authorization)

When you declare `requires_auth` on a tool, Arcade handles the entire OAuth flow automatically:

1. **Agent calls the tool** -- Arcade checks if the user has authorized the required scopes
2. **If not authorized** -- Arcade initiates the OAuth flow: the user sees a URL, logs in, and grants consent in their browser. The tool is then re-invoked automatically.
3. **If authorized** -- the OAuth token is securely injected into `Context`. The LLM and MCP client never see it.
4. **Token persistence** -- Arcade remembers the authorization until the user revokes it. No re-auth on subsequent calls.
5. **Token refresh** -- Arcade handles token expiration and refresh transparently.

**As a tool developer, you write zero auth code.** Just declare `requires_auth` and call `context.get_auth_token_or_empty()`. Arcade does the rest.

**Built-in providers** (Google, Slack, GitHub, Reddit) work out of the box -- Arcade provides default OAuth apps so you don't need to register your own. For other services, use `OAuth2(id="provider-id", scopes=[...])` with credentials configured in the Arcade Dashboard.

### OAuth Code Pattern

```python
@tool(
    requires_auth=Google(
        scopes=["https://www.googleapis.com/auth/gmail.readonly"]
    )
)
async def my_oauth_tool(
    context: Context,
    query: Annotated[str, "Search query"],
) -> Annotated[dict, "Search results"]:
    """Search for items using the service API."""
    token = context.get_auth_token_or_empty()

    async with httpx.AsyncClient() as client:
        response = await client.get(
            "https://api.service.com/search",
            headers={"Authorization": f"Bearer {token}"},
            params={"q": query},
        )
        response.raise_for_status()
        return response.json()
```

Available OAuth providers and import paths:

| Provider | Import | Usage |
|----------|--------|-------|
| Google | `from arcade_mcp_server.auth import Google` | `Google(scopes=["..."])` |
| Slack | `from arcade_mcp_server.auth import Slack` | `Slack(scopes=["..."])` |
| GitHub | `from arcade_mcp_server.auth import GitHub` | `GitHub(scopes=["..."])` |
| Reddit | `from arcade_mcp_server.auth import Reddit` | `Reddit(scopes=["..."])` |
| Custom | `from arcade_mcp_server.auth import OAuth2` | `OAuth2(id="provider-id", scopes=["..."])` |

### Secrets / API Key Pattern

```python
@tool(requires_secrets=["SERVICE_API_KEY", "ACCOUNT_ID"])
async def my_secret_tool(
    context: Context,
    item_id: Annotated[str, "The item ID to retrieve"],
) -> Annotated[dict, "Item details"]:
    """Retrieve an item by ID from the service."""
    api_key = context.get_secret("SERVICE_API_KEY")
    account_id = context.get_secret("ACCOUNT_ID")

    async with httpx.AsyncClient() as client:
        response = await client.get(
            f"https://api.service.com/v1/accounts/{account_id}/items/{item_id}",
            headers={"X-Api-Key": api_key},
        )
        response.raise_for_status()
        return response.json()
```

### Hybrid Pattern (OAuth + Secrets)

```python
@tool(
    requires_auth=GitHub(scopes=["repo"]),
    requires_secrets=["GITHUB_SERVER_URL"],
)
async def my_hybrid_tool(context: Context, ...) -> Annotated[dict, "..."]:
    """Tool needing both user auth and server config."""
    token = context.get_auth_token_or_empty()
    server_url = context.get_secret("GITHUB_SERVER_URL")
    ...
```

---

## Step 5: Apply Quality Patterns

Apply these patterns for production quality. For the full patterns reference, read [patterns-reference.md](patterns-reference.md).

### Constrained Inputs -- use Enums instead of free-form strings

```python
from enum import Enum

class SortOrder(str, Enum):
    ASCENDING = "ascending"
    DESCENDING = "descending"

class ContentType(str, Enum):
    PLAIN = "plain"
    HTML = "html"
```

### Smart Defaults with Bounds Clamping

```python
MIN_RESULTS = 1
MAX_RESULTS = 50
DEFAULT_RESULTS = 10

@tool(...)
async def list_items(
    context: Context,
    max_results: Annotated[
        int, f"Number of items to return (Min {MIN_RESULTS}, Max {MAX_RESULTS})"
    ] = DEFAULT_RESULTS,
) -> Annotated[ListItemsOutput, "..."]:
    """List items from the service."""
    max_results = min(max(max_results, MIN_RESULTS), MAX_RESULTS)
    ...
```

### Response Shaping -- TypedDict outputs

```python
from typing import TypedDict

class ItemOutput(TypedDict):
    id: str
    name: str
    url: str
    created_at: str

class ListItemsOutput(TypedDict):
    items: list[ItemOutput]
    total_count: int
    next_page_token: str | None
```

### Client Wrapper Pattern

Encapsulate SDK/API interaction in a class with lazy initialization:

```python
class ServiceClient:
    def __init__(self, context: Context):
        self.context = context
        self._service = None

    @property
    def service(self):
        if not self._service:
            from google.oauth2.credentials import Credentials
            token = (
                self.context.authorization.token
                if self.context.authorization and self.context.authorization.token
                else ""
            )
            self._service = build("api", "v1", credentials=Credentials(token))
        return self._service

    def get_item(self, item_id: str) -> dict:
        return self.service.items().get(id=item_id).execute()
```

### Error Recovery

Use `RetryableToolError` when the LLM can fix the input:

```python
if not user:
    raise RetryableToolError(
        message=f"User '{username}' not found.",
        developer_message=f"User '{username}' not found.",
        additional_prompt_content=f"Valid usernames: {client.list_usernames()}",
        retry_after_ms=500,
    )
```

Use `ToolExecutionError` for known but unrecoverable failures:

```python
raise ToolExecutionError(message="Database connection failed.", developer_message=str(e))
```

### GUI URLs in Responses

Always include web URLs so users can view/edit results in a browser:

```python
return {
    "id": item["id"],
    "name": item["name"],
    "url": f"https://app.service.com/items/{item['id']}",
}
```

### Pagination

Use cursor-based pagination (not page numbers). Accept `page_token: Annotated[str | None, "..."] = None` and return `next_page_token` in the output.

---

## Step 6: Assemble the Server

```python
import sys
from typing import cast
from arcade_mcp_server import MCPApp
from arcade_mcp_server.mcp_app import TransportType
import my_server

app = MCPApp(
    name="MyServer",
    version="1.0.0",
    instructions="Use this server to interact with ServiceX to manage items and workflows.",
)
app.add_tools_from_module(my_server)

def main() -> None:
    transport = sys.argv[1] if len(sys.argv) > 1 else "stdio"
    app.run(transport=cast(TransportType, transport), host="127.0.0.1", port=8000)

if __name__ == "__main__":
    main()
```

Alternatives: `app.add_tool(fn)` for individual tools, or `@app.tool` for inline definitions in simple servers.

---

## Step 7: Configure Dependencies

### pyproject.toml

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "my_server"
version = "1.0.0"
description = "Arcade MCP tools for ServiceX"
requires-python = ">=3.10"
dependencies = [
    "arcade-mcp-server>=1.9.2,<2.0.0",
    "httpx>=0.27.0,<1.0.0",
]
```

Add SDK-specific deps as needed (e.g., `google-api-python-client`, `slack_sdk`). See [templates.md](templates.md) for full `pyproject.toml` examples with dev dependencies.

For local secrets, create a `.env` file alongside `server.py` (see [templates.md](templates.md) for examples).

---

## Step 8: Test Locally

```bash
# stdio transport -- supports auth + secrets locally
uv run src/my_server/server.py stdio

# HTTP transport -- view docs at http://127.0.0.1:8000/docs
uv run src/my_server/server.py http

# Configure your MCP client
arcade configure cursor    # Cursor IDE
arcade configure claude    # Claude Desktop
arcade configure vscode    # VS Code
```

stdio supports full auth and secrets locally. HTTP transport does NOT support tool-level auth/secrets locally -- use `arcade deploy` for that.

---

## Step 9: Deploy to Arcade Cloud

### Set secrets for production

```bash
arcade secret set SERVICE_API_KEY="production-key"
arcade secret set ACCOUNT_ID="production-account"
```

Or set secrets in the [Arcade Dashboard](https://api.arcade.dev/dashboard/servers) under Secrets.

### Deploy

```bash
# Run from the directory containing pyproject.toml
arcade deploy -e src/my_server/server.py
```

Requirements for deployment:
- `arcade login` completed
- `pyproject.toml` exists in current directory
- Server entrypoint calls `app.run()` inside `if __name__ == "__main__":`
- All secrets set via CLI or Dashboard

### Post-deployment

1. Monitor health at [Arcade Dashboard > Servers](https://api.arcade.dev/dashboard/servers)
2. Create an [MCP Gateway](https://docs.arcade.dev/en/guides/mcp-gateways.md) to select tools for clients
3. Connect MCP clients to the gateway

---

## Pre-Deployment Checklist

- [ ] Every tool function is `async def`
- [ ] Every parameter uses `Annotated[Type, "description"]`
- [ ] Every return type uses `Annotated[OutputType, "description"]`
- [ ] Docstrings are concise, LLM-optimized (not human prose)
- [ ] Enums used for any parameter with a fixed set of valid values
- [ ] Smart defaults provided to minimize required parameters
- [ ] Numeric parameters clamped with `min(max(val, MIN), MAX)`
- [ ] Responses are flat, structured dicts -- not raw API payloads
- [ ] `RetryableToolError` used when LLM can fix the input
- [ ] `ToolExecutionError` used for known but unrecoverable failures
- [ ] Secrets accessed via `context.get_secret()`, never as tool params
- [ ] OAuth tokens accessed via `context.get_auth_token_or_empty()`
- [ ] `app.run()` is inside `if __name__ == "__main__":`
- [ ] GUI URLs included in responses where applicable
- [ ] `pyproject.toml` has correct `arcade-mcp-server` dependency
- [ ] Secrets configured via `arcade secret set` or Dashboard

## Additional Resources

- Read [patterns-reference.md](patterns-reference.md) for the condensed agentic tool patterns checklist
- Read [templates.md](templates.md) for complete copy-paste OAuth and Secrets tool templates
- **Fetch https://docs.arcade.dev/llms.txt** when you need deeper Arcade platform context (product docs, auth providers, MCP gateways, deployment options)
- **Fetch https://arcade.dev/patterns/llm.txt** when you need the full 54-pattern catalog for agentic tool design
- Deployment guide: https://docs.arcade.dev/en/guides/deployment-hosting/arcade-deploy.md
- Auth providers reference: https://docs.arcade.dev/en/references/auth-providers.md
