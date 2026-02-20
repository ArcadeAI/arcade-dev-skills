# Arcade MCP Tool Templates

Complete, deployment-ready templates. Copy and adapt for your integration.

---

## Template A: OAuth Tool (Google Example)

A production-quality server using OAuth with Google, demonstrating all key patterns.

### Project Structure

```
my_google_server/
├── src/
│   └── my_google_server/
│       ├── __init__.py
│       ├── server.py
│       ├── client.py
│       ├── constants.py
│       ├── tools/
│       │   ├── __init__.py
│       │   ├── queries.py
│       │   └── commands.py
│       └── models/
│           ├── enums.py
│           └── outputs.py
└── pyproject.toml
```

### pyproject.toml

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "my_google_server"
version = "1.0.0"
description = "Arcade MCP tools for Google Service"
requires-python = ">=3.10"
dependencies = [
    "arcade-mcp-server>=1.9.2,<2.0.0",
    "google-api-python-client>=2.137.0,<3.0.0",
    "google-auth>=2.30.0,<3.0.0",
]

[project.optional-dependencies]
dev = [
    "arcade-mcp[evals]>=1.5.6,<2.0.0",
    "pytest>=8.3.0,<8.4.0",
    "ruff>=0.7.4,<0.8.0",
]
```

### models/enums.py

```python
from enum import Enum


class SortOrder(str, Enum):
    ASCENDING = "ascending"
    DESCENDING = "descending"


class DateRange(str, Enum):
    TODAY = "today"
    THIS_WEEK = "this_week"
    THIS_MONTH = "this_month"
    THIS_YEAR = "this_year"
```

### models/outputs.py

```python
from typing import TypedDict


class ItemOutput(TypedDict):
    id: str
    name: str
    description: str
    created_at: str
    url: str


class ListItemsOutput(TypedDict):
    items: list[ItemOutput]
    total_count: int
    next_page_token: str | None


class CreateItemOutput(TypedDict):
    id: str
    name: str
    url: str
```

### constants.py

```python
MIN_RESULTS = 1
MAX_RESULTS = 50
DEFAULT_RESULTS = 10

SCOPES_READONLY = ["https://www.googleapis.com/auth/service.readonly"]
SCOPES_WRITE = ["https://www.googleapis.com/auth/service"]

APP_BASE_URL = "https://app.service.com"
```

### client.py

```python
import logging
from typing import Any

from arcade_mcp_server import Context
from arcade_mcp_server.exceptions import ToolExecutionError
from google.oauth2.credentials import Credentials
from googleapiclient.discovery import build

logger = logging.getLogger(__name__)


class ServiceClient:
    def __init__(self, context: Context):
        self.context = context
        self._service = None

    @property
    def service(self) -> Any:
        if not self._service:
            self._service = self._build_service()
        return self._service

    def _build_service(self) -> Any:
        try:
            token = (
                self.context.authorization.token
                if self.context.authorization and self.context.authorization.token
                else ""
            )
            credentials = Credentials(token)
            return build("service", "v1", credentials=credentials)
        except Exception as e:
            raise ToolExecutionError(
                message="Failed to build service client.",
                developer_message=str(e),
            )

    def list_items(self, query: str | None, limit: int) -> list[dict]:
        params: dict[str, Any] = {"maxResults": limit}
        if query:
            params["q"] = query
        response = self.service.items().list(**params).execute()
        return response.get("items", [])

    def get_item(self, item_id: str) -> dict:
        return self.service.items().get(id=item_id).execute()

    def create_item(self, name: str, description: str) -> dict:
        body = {"name": name, "description": description}
        return self.service.items().create(body=body).execute()

    def delete_item(self, item_id: str) -> None:
        self.service.items().delete(id=item_id).execute()
```

### tools/queries.py

```python
from typing import Annotated

from arcade_mcp_server import Context, tool
from arcade_mcp_server.auth import Google

from my_google_server.client import ServiceClient
from my_google_server.constants import (
    APP_BASE_URL,
    DEFAULT_RESULTS,
    MAX_RESULTS,
    MIN_RESULTS,
    SCOPES_READONLY,
)
from my_google_server.models.enums import SortOrder
from my_google_server.models.outputs import ItemOutput, ListItemsOutput


@tool(requires_auth=Google(scopes=SCOPES_READONLY))
async def list_items(
    context: Context,
    query: Annotated[str | None, "Search query to filter items"] = None,
    max_results: Annotated[
        int,
        f"Number of items to return (Min {MIN_RESULTS}, Max {MAX_RESULTS})",
    ] = DEFAULT_RESULTS,
    sort_order: Annotated[
        SortOrder, "Sort order for results"
    ] = SortOrder.DESCENDING,
) -> Annotated[ListItemsOutput, "Paginated list of items with metadata"]:
    """List items from the service, optionally filtered by a search query."""
    client = ServiceClient(context)
    max_results = min(max(max_results, MIN_RESULTS), MAX_RESULTS)

    raw_items = client.list_items(query, max_results)
    if not raw_items:
        return {"items": [], "total_count": 0, "next_page_token": None}

    items: list[ItemOutput] = [
        {
            "id": item["id"],
            "name": item.get("name", ""),
            "description": item.get("description", ""),
            "created_at": item.get("createdTime", ""),
            "url": f"{APP_BASE_URL}/items/{item['id']}",
        }
        for item in raw_items
    ]

    return {
        "items": items,
        "total_count": len(items),
        "next_page_token": None,
    }


@tool(requires_auth=Google(scopes=SCOPES_READONLY))
async def get_item(
    context: Context,
    item_id: Annotated[str, "The ID of the item to retrieve"],
) -> Annotated[ItemOutput, "Full details of the requested item"]:
    """Get a single item by its ID."""
    client = ServiceClient(context)
    item = client.get_item(item_id)

    return {
        "id": item["id"],
        "name": item.get("name", ""),
        "description": item.get("description", ""),
        "created_at": item.get("createdTime", ""),
        "url": f"{APP_BASE_URL}/items/{item['id']}",
    }
```

### tools/commands.py

```python
from typing import Annotated

from arcade_mcp_server import Context, tool
from arcade_mcp_server.auth import Google

from my_google_server.client import ServiceClient
from my_google_server.constants import APP_BASE_URL, SCOPES_WRITE
from my_google_server.models.outputs import CreateItemOutput


@tool(requires_auth=Google(scopes=SCOPES_WRITE))
async def create_item(
    context: Context,
    name: Annotated[str, "Name of the item to create"],
    description: Annotated[str, "Description of the item"],
) -> Annotated[CreateItemOutput, "The newly created item with its URL"]:
    """Create a new item in the service."""
    client = ServiceClient(context)
    item = client.create_item(name, description)

    return {
        "id": item["id"],
        "name": item.get("name", ""),
        "url": f"{APP_BASE_URL}/items/{item['id']}",
    }


@tool(requires_auth=Google(scopes=SCOPES_WRITE))
async def delete_item(
    context: Context,
    item_id: Annotated[str, "The ID of the item to delete"],
) -> Annotated[str, "Confirmation message"]:
    """Delete an item by its ID. This action is irreversible."""
    client = ServiceClient(context)
    client.delete_item(item_id)
    return f"Item {item_id} deleted successfully."
```

### tools/__init__.py

```python
from my_google_server.tools.commands import create_item, delete_item
from my_google_server.tools.queries import get_item, list_items

__all__ = ["list_items", "get_item", "create_item", "delete_item"]
```

### __init__.py

```python
from my_google_server.tools import *  # noqa: F401, F403
from my_google_server.tools import __all__
```

### server.py

```python
import sys
from typing import cast

from arcade_mcp_server import MCPApp
from arcade_mcp_server.mcp_app import TransportType

import my_google_server

app = MCPApp(
    name="MyGoogleServer",
    version="1.0.0",
    instructions=(
        "Use this server when you need to manage items in Google Service. "
        "You can list, search, create, and delete items."
    ),
)

app.add_tools_from_module(my_google_server)


def main() -> None:
    transport = sys.argv[1] if len(sys.argv) > 1 else "stdio"
    host = sys.argv[2] if len(sys.argv) > 2 else "127.0.0.1"
    port = int(sys.argv[3]) if len(sys.argv) > 3 else 8000
    app.run(transport=cast(TransportType, transport), host=host, port=port)


if __name__ == "__main__":
    main()
```

### Deploy

```bash
arcade deploy -e src/my_google_server/server.py
```

---

## Template B: API Key / Secrets Tool

A production-quality server using API key authentication via secrets.

### Project Structure

```
my_api_server/
├── src/
│   └── my_api_server/
│       ├── __init__.py
│       ├── server.py
│       ├── client.py
│       ├── constants.py
│       ├── tools/
│       │   ├── __init__.py
│       │   └── tools.py
│       └── models/
│           └── outputs.py
├── .env
├── .env.example
└── pyproject.toml
```

### pyproject.toml

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "my_api_server"
version = "1.0.0"
description = "Arcade MCP tools for External API Service"
requires-python = ">=3.10"
dependencies = [
    "arcade-mcp-server>=1.9.2,<2.0.0",
    "httpx>=0.27.0,<1.0.0",
]

[project.optional-dependencies]
dev = [
    "arcade-mcp[evals]>=1.5.6,<2.0.0",
    "pytest>=8.3.0,<8.4.0",
    "ruff>=0.7.4,<0.8.0",
]
```

### .env.example

```bash
SERVICE_API_KEY="your-api-key-here"
SERVICE_BASE_URL="https://api.service.com/v1"
```

### constants.py

```python
MIN_RESULTS = 1
MAX_RESULTS = 100
DEFAULT_RESULTS = 20
```

### models/outputs.py

```python
from typing import TypedDict


class RecordOutput(TypedDict):
    id: str
    title: str
    status: str
    created_at: str
    url: str


class ListRecordsOutput(TypedDict):
    records: list[RecordOutput]
    total_count: int


class CreateRecordOutput(TypedDict):
    id: str
    title: str
    url: str
```

### client.py

```python
from typing import Any

import httpx

from arcade_mcp_server import Context
from arcade_mcp_server.exceptions import ToolExecutionError


class APIClient:
    def __init__(self, context: Context):
        self.api_key = context.get_secret("SERVICE_API_KEY")
        self.base_url = context.get_secret("SERVICE_BASE_URL")

    def _headers(self) -> dict[str, str]:
        return {
            "Authorization": f"Bearer {self.api_key}",
            "Content-Type": "application/json",
        }

    async def list_records(self, limit: int) -> list[dict[str, Any]]:
        async with httpx.AsyncClient() as client:
            response = await client.get(
                f"{self.base_url}/records",
                headers=self._headers(),
                params={"limit": limit},
            )
            response.raise_for_status()
            return response.json().get("records", [])

    async def get_record(self, record_id: str) -> dict[str, Any]:
        async with httpx.AsyncClient() as client:
            response = await client.get(
                f"{self.base_url}/records/{record_id}",
                headers=self._headers(),
            )
            response.raise_for_status()
            return response.json()

    async def create_record(self, title: str, status: str) -> dict[str, Any]:
        async with httpx.AsyncClient() as client:
            response = await client.post(
                f"{self.base_url}/records",
                headers=self._headers(),
                json={"title": title, "status": status},
            )
            response.raise_for_status()
            return response.json()
```

### tools/tools.py

```python
from typing import Annotated

from arcade_mcp_server import Context, tool
from arcade_mcp_server.exceptions import RetryableToolError

from my_api_server.client import APIClient
from my_api_server.constants import DEFAULT_RESULTS, MAX_RESULTS, MIN_RESULTS
from my_api_server.models.outputs import (
    CreateRecordOutput,
    ListRecordsOutput,
    RecordOutput,
)


@tool(requires_secrets=["SERVICE_API_KEY", "SERVICE_BASE_URL"])
async def list_records(
    context: Context,
    max_results: Annotated[
        int,
        f"Number of records to return (Min {MIN_RESULTS}, Max {MAX_RESULTS})",
    ] = DEFAULT_RESULTS,
) -> Annotated[ListRecordsOutput, "List of records with metadata"]:
    """List records from the service."""
    client = APIClient(context)
    max_results = min(max(max_results, MIN_RESULTS), MAX_RESULTS)

    raw_records = await client.list_records(max_results)
    if not raw_records:
        return {"records": [], "total_count": 0}

    records: list[RecordOutput] = [
        {
            "id": r["id"],
            "title": r.get("title", ""),
            "status": r.get("status", ""),
            "created_at": r.get("created_at", ""),
            "url": f"https://app.service.com/records/{r['id']}",
        }
        for r in raw_records
    ]

    return {"records": records, "total_count": len(records)}


@tool(requires_secrets=["SERVICE_API_KEY", "SERVICE_BASE_URL"])
async def get_record(
    context: Context,
    record_id: Annotated[str, "The ID of the record to retrieve"],
) -> Annotated[RecordOutput, "Full details of the requested record"]:
    """Get a single record by its ID."""
    client = APIClient(context)
    r = await client.get_record(record_id)

    return {
        "id": r["id"],
        "title": r.get("title", ""),
        "status": r.get("status", ""),
        "created_at": r.get("created_at", ""),
        "url": f"https://app.service.com/records/{r['id']}",
    }


VALID_STATUSES = ["open", "in_progress", "done", "cancelled"]


@tool(requires_secrets=["SERVICE_API_KEY", "SERVICE_BASE_URL"])
async def create_record(
    context: Context,
    title: Annotated[str, "Title of the record to create"],
    status: Annotated[str, "Status of the record: open, in_progress, done, cancelled"] = "open",
) -> Annotated[CreateRecordOutput, "The newly created record with its URL"]:
    """Create a new record in the service."""
    if status not in VALID_STATUSES:
        raise RetryableToolError(
            message=f"Invalid status: '{status}'.",
            developer_message=f"Status '{status}' is not a valid status.",
            additional_prompt_content=f"Valid statuses: {VALID_STATUSES}",
            retry_after_ms=500,
        )

    client = APIClient(context)
    r = await client.create_record(title, status)

    return {
        "id": r["id"],
        "title": r.get("title", ""),
        "url": f"https://app.service.com/records/{r['id']}",
    }
```

### tools/__init__.py

```python
from my_api_server.tools.tools import create_record, get_record, list_records

__all__ = ["list_records", "get_record", "create_record"]
```

### __init__.py

```python
from my_api_server.tools import *  # noqa: F401, F403
from my_api_server.tools import __all__
```

### server.py

```python
import sys
from typing import cast

from arcade_mcp_server import MCPApp
from arcade_mcp_server.mcp_app import TransportType

import my_api_server

app = MCPApp(
    name="MyAPIServer",
    version="1.0.0",
    instructions=(
        "Use this server to manage records in the External API Service. "
        "You can list, retrieve, and create records."
    ),
)

app.add_tools_from_module(my_api_server)


def main() -> None:
    transport = sys.argv[1] if len(sys.argv) > 1 else "stdio"
    host = sys.argv[2] if len(sys.argv) > 2 else "127.0.0.1"
    port = int(sys.argv[3]) if len(sys.argv) > 3 else 8000
    app.run(transport=cast(TransportType, transport), host=host, port=port)


if __name__ == "__main__":
    main()
```

### Set secrets and deploy

```bash
# Set secrets for production
arcade secret set SERVICE_API_KEY="your-production-key"
arcade secret set SERVICE_BASE_URL="https://api.service.com/v1"

# Deploy
arcade deploy -e src/my_api_server/server.py
```

---

## Template C: Simple Inline Server (Quick Start)

For rapid prototyping or servers with 1-3 tools. Single file, no package structure.

### server.py

```python
#!/usr/bin/env python3
import sys
from typing import Annotated

import httpx
from arcade_mcp_server import Context, MCPApp
from arcade_mcp_server.auth import Reddit

app = MCPApp(name="my_quick_server", version="1.0.0", log_level="DEBUG")


@app.tool
def greet(name: Annotated[str, "The name of the person to greet"]) -> str:
    """Greet a person by name."""
    return f"Hello, {name}!"


@app.tool(requires_secrets=["MY_SECRET"])
def check_secret(context: Context) -> Annotated[str, "Masked secret confirmation"]:
    """Confirm a secret is configured by showing its last 4 characters."""
    try:
        secret = context.get_secret("MY_SECRET")
    except Exception as e:
        return str(e)
    return f"Secret configured (ends with: ...{secret[-4:]})"


@app.tool(requires_auth=Reddit(scopes=["read"]))
async def get_subreddit_posts(
    context: Context,
    subreddit: Annotated[str, "Name of the subreddit (without r/ prefix)"],
) -> Annotated[dict, "Top posts from the subreddit"]:
    """Get the top posts from a subreddit."""
    subreddit = subreddit.lower().replace("r/", "").strip()
    token = context.get_auth_token_or_empty()

    async with httpx.AsyncClient() as client:
        response = await client.get(
            f"https://oauth.reddit.com/r/{subreddit}/hot",
            headers={
                "Authorization": f"Bearer {token}",
                "User-Agent": "my-mcp-server",
            },
            params={"limit": 5},
        )
        response.raise_for_status()
        return response.json()


if __name__ == "__main__":
    transport = sys.argv[1] if len(sys.argv) > 1 else "stdio"
    app.run(transport=transport, host="127.0.0.1", port=8000)
```

### Deploy

```bash
arcade deploy -e server.py
```
