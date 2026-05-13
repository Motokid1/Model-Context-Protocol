# Model Context Protocol (MCP) — Complete Developer Guide
### From Zero to Production: Hands-On with Theory, Architecture, and Code

---

> **Who this guide is for:** Developers who want to understand MCP from first principles, use existing MCP servers, and build their own — from a simple tool server to a full-featured multi-resource production server.

---

## Table of Contents

1. [What is MCP? (Theory + Mental Model)](#1-what-is-mcp)
2. [Architecture Deep Dive](#2-architecture-deep-dive)
3. [Core Concepts: The Building Blocks](#3-core-concepts)
4. [Setup & Environment](#4-setup--environment)
5. [Using Existing MCP Servers (Hands-On)](#5-using-existing-mcp-servers)
6. [Building Your First MCP Server — Basic](#6-building-your-first-mcp-server--basic)
7. [Tools — Decorators, Parameters & Patterns](#7-tools--decorators-parameters--patterns)
8. [Resources — Exposing Data](#8-resources--exposing-data)
9. [Prompts — Reusable Templates](#9-prompts--reusable-templates)
10. [Medium: Multi-Tool Servers with State](#10-medium-multi-tool-servers-with-state)
11. [Advanced: Full Production Server](#11-advanced-full-production-server)
12. [Transport Layer (stdio vs SSE vs HTTP)](#12-transport-layer)
13. [Error Handling & Validation](#13-error-handling--validation)
14. [Authentication & Security](#14-authentication--security)
15. [Sampling & LLM-in-Server Patterns](#15-sampling--llm-in-server-patterns)
16. [Roots & File System Context](#16-roots--file-system-context)
17. [Logging & Observability](#17-logging--observability)
18. [Testing MCP Servers](#18-testing-mcp-servers)
19. [Rarely Used but Powerful Concepts](#19-rarely-used-but-powerful-concepts)
20. [Common Architectures & Workflow Patterns](#20-common-architectures--workflow-patterns)
21. [MCP Registry & Distribution](#21-mcp-registry--distribution)
22. [Reference Cheatsheet](#22-reference-cheatsheet)

---

## 1. What is MCP?

### Definition

**Model Context Protocol (MCP)** is an open protocol developed by Anthropic that standardizes how AI models (like Claude) connect to external data sources, tools, and capabilities. Think of it as **USB-C for AI** — a universal connector that lets any LLM plug into any service without custom integration code.

Before MCP, every AI integration looked like this:

```
Claude ──custom code──► GitHub API
Claude ──custom code──► Database
Claude ──custom code──► File System
Claude ──custom code──► Slack
```

With MCP, it becomes:

```
Claude ──MCP──► MCP Server (GitHub)
Claude ──MCP──► MCP Server (Database)
Claude ──MCP──► MCP Server (File System)
Claude ──MCP──► MCP Server (Slack)
```

The protocol is the same. Only the server changes.

### Why MCP Matters

| Problem Before MCP | Solution with MCP |
|---|---|
| Every tool needed custom glue code | One protocol, any tool |
| No standard for how AI calls external APIs | Typed, validated tool definitions |
| Hard to give AI access to live data | Resources expose live data uniformly |
| Prompt engineering was ad hoc | Prompts are first-class server objects |
| Security boundary unclear | Server controls what AI can access |

### The Mental Model

MCP is a **client-server protocol** where:

- **MCP Host** = The AI application (e.g., Claude Desktop, your app)
- **MCP Client** = Lives inside the host; speaks the protocol
- **MCP Server** = Your code; exposes tools/data/prompts
- **LLM** = Uses tools exposed by the server, never talks to it directly

```
┌─────────────────────────────────┐
│         Host Application        │
│  ┌─────────┐    ┌────────────┐  │
│  │   LLM   │◄──►│ MCP Client │  │
│  └─────────┘    └─────┬──────┘  │
└────────────────────────┼────────┘
                         │ MCP Protocol (JSON-RPC 2.0)
              ┌──────────┴───────────┐
              │      MCP Server      │
              │  Tools │ Resources   │
              │  Prompts │ Sampling  │
              └──────────────────────┘
```

---

## 2. Architecture Deep Dive

### Protocol Layer: JSON-RPC 2.0

MCP uses **JSON-RPC 2.0** as its wire format. Every message is a JSON object:

```json
// Request (client → server)
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "get_weather",
    "arguments": { "city": "Hyderabad" }
  }
}

// Response (server → client)
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [
      { "type": "text", "text": "Temperature: 35°C, Humid" }
    ]
  }
}
```

You almost never write raw JSON-RPC — the SDK handles it. But understanding the wire format helps debug issues.

### Capability Negotiation (The Handshake)

When a client connects, they negotiate capabilities:

```
Client ──initialize──► Server
       ◄─capabilities── Server
Client ──initialized──► Server
       ◄── (ready) ─── Server
```

The server declares what it supports:

```json
{
  "capabilities": {
    "tools": {},
    "resources": { "subscribe": true, "listChanged": true },
    "prompts": { "listChanged": true },
    "logging": {},
    "sampling": {}
  }
}
```

### Lifecycle of a Tool Call

```
1. User says: "What's the weather in Hyderabad?"
2. LLM decides to call get_weather tool
3. MCP Client sends tools/call request to server
4. Server executes your Python/JS function
5. Server returns result as content blocks
6. LLM incorporates result into response
7. User sees final answer
```

---

## 3. Core Concepts

MCP servers expose **four primitives**:

### 3.1 Tools (Model-Controlled Actions)

Tools are **functions the LLM can call**. They have:
- A name
- A description (the LLM reads this to decide when to use it)
- An input schema (JSON Schema)
- An implementation

**Analogy:** Like API endpoints, but the LLM decides when to call them.

### 3.2 Resources (Application-Controlled Data)

Resources are **data the host/user can expose** to the LLM. Think of them as file-like objects with URIs.

**Analogy:** Like `GET` endpoints — read-only data access.

```
myapp://config/settings
file:///home/user/docs/notes.txt
db://users/123/profile
```

### 3.3 Prompts (User-Controlled Templates)

Prompts are **reusable message templates** that users or apps can invoke. They accept arguments and return structured message sequences.

**Analogy:** Like saved macros or slash commands.

### 3.4 Sampling (Server → LLM Calls)

Sampling lets the **server ask the LLM** to generate something. This is advanced — the server sends a completion request back through the client to the LLM.

**Analogy:** The server gets its own "AI brain" it can query.

---

## 4. Setup & Environment

### Prerequisites

```bash
# Python (recommended for beginners)
python --version  # 3.10+

# Node.js (for JS servers)
node --version    # 18+

# Install MCP SDK (Python)
pip install mcp

# Install MCP SDK (Node.js)
npm install @modelcontextprotocol/sdk
```

### Python SDK Structure

```bash
pip install mcp[cli]  # includes CLI tools
```

Key packages:

| Package | Purpose |
|---|---|
| `mcp` | Core SDK |
| `mcp.server` | Server base classes |
| `mcp.server.fastmcp` | FastMCP — high-level decorator API |
| `mcp.types` | Type definitions |
| `mcp.client` | Client for connecting to servers |

### Claude Desktop Configuration

MCP servers connect to Claude Desktop via `claude_desktop_config.json`:

**MacOS:** `~/Library/Application Support/Claude/claude_desktop_config.json`  
**Windows:** `%APPDATA%\Claude\claude_desktop_config.json`

```json
{
  "mcpServers": {
    "my-server": {
      "command": "python",
      "args": ["/path/to/server.py"],
      "env": {
        "API_KEY": "your-key"
      }
    },
    "node-server": {
      "command": "node",
      "args": ["/path/to/server.js"]
    }
  }
}
```

---

## 5. Using Existing MCP Servers

Before building, use some existing servers to understand the experience.

### 5.1 Filesystem MCP Server

The official filesystem server lets Claude read/write files.

```bash
# Install
npm install -g @modelcontextprotocol/server-filesystem

# Add to claude_desktop_config.json
```

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "@modelcontextprotocol/server-filesystem",
        "/home/youruser/documents"
      ]
    }
  }
}
```

Once configured, you can tell Claude: *"Read my notes.txt and summarize it."*

### 5.2 SQLite MCP Server

```bash
npm install -g @modelcontextprotocol/server-sqlite
```

```json
{
  "mcpServers": {
    "sqlite": {
      "command": "npx",
      "args": ["@modelcontextprotocol/server-sqlite", "--db-path", "/path/to/db.sqlite"]
    }
  }
}
```

Claude can now run SQL queries on your database.

### 5.3 GitHub MCP Server

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_xxxx"
      }
    }
  }
}
```

### 5.4 Testing with MCP Inspector

The **MCP Inspector** is a browser-based debugger:

```bash
npx @modelcontextprotocol/inspector python server.py
```

This opens a UI at `http://localhost:5173` where you can:
- See all tools, resources, and prompts
- Call tools manually
- Inspect messages
- Test without Claude Desktop

---

## 6. Building Your First MCP Server — Basic

### 6.1 Hello World (Python with FastMCP)

`FastMCP` is the high-level, decorator-based API. Start here.

```python
# server.py
from mcp.server.fastmcp import FastMCP

# Create the server instance
mcp = FastMCP("My First Server")

@mcp.tool()
def hello(name: str) -> str:
    """Say hello to someone."""
    return f"Hello, {name}! Welcome to MCP."

if __name__ == "__main__":
    mcp.run()
```

Run it:

```bash
python server.py
```

Test with Inspector:

```bash
npx @modelcontextprotocol/inspector python server.py
```

### 6.2 What FastMCP Does for You

The `@mcp.tool()` decorator automatically:
1. Reads the function signature → generates JSON Schema
2. Reads the docstring → sets the tool description
3. Handles type conversion for inputs/outputs
4. Registers the tool with the server
5. Handles the JSON-RPC plumbing

Without FastMCP, the same tool looks like this (low-level SDK):

```python
# Low-level equivalent — for reference only
from mcp.server import Server
from mcp import types

server = Server("My First Server")

@server.list_tools()
async def list_tools():
    return [
        types.Tool(
            name="hello",
            description="Say hello to someone.",
            inputSchema={
                "type": "object",
                "properties": {
                    "name": {"type": "string", "description": "Name to greet"}
                },
                "required": ["name"]
            }
        )
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict):
    if name == "hello":
        n = arguments.get("name", "World")
        return [types.TextContent(type="text", text=f"Hello, {n}!")]
```

FastMCP is dramatically cleaner. Use it unless you need low-level control.

### 6.3 Hello World (Node.js)

```typescript
// server.ts
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

const server = new McpServer({ name: "My First Server", version: "1.0.0" });

server.tool(
  "hello",
  "Say hello to someone",
  { name: z.string().describe("Name to greet") },
  async ({ name }) => ({
    content: [{ type: "text", text: `Hello, ${name}! Welcome to MCP.` }]
  })
);

const transport = new StdioServerTransport();
await server.connect(transport);
```

---

## 7. Tools — Decorators, Parameters & Patterns

### 7.1 The `@mcp.tool()` Decorator

```python
@mcp.tool()
def my_tool(param: type) -> return_type:
    """Tool description — THIS IS WHAT THE LLM READS."""
    ...
```

**The docstring is your tool's "marketing copy" for the LLM.** Write it clearly.

### 7.2 Type Annotations → JSON Schema Mapping

FastMCP reads Python type hints and converts them to JSON Schema automatically:

| Python Type | JSON Schema |
|---|---|
| `str` | `{"type": "string"}` |
| `int` | `{"type": "integer"}` |
| `float` | `{"type": "number"}` |
| `bool` | `{"type": "boolean"}` |
| `list[str]` | `{"type": "array", "items": {"type": "string"}}` |
| `dict` | `{"type": "object"}` |
| `Optional[str]` | `{"type": "string"}` (not required) |
| Pydantic Model | Full nested schema |

```python
from typing import Optional, List
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("Type Demo")

@mcp.tool()
def search_files(
    query: str,
    max_results: int = 10,
    file_types: Optional[List[str]] = None,
    case_sensitive: bool = False
) -> List[str]:
    """
    Search for files matching a query.
    
    Args:
        query: Search term to look for
        max_results: Maximum number of results to return (default: 10)
        file_types: List of extensions to filter by, e.g. ['.py', '.txt']
        case_sensitive: Whether search is case sensitive
    """
    # implementation
    return ["file1.py", "file2.txt"]
```

### 7.3 Pydantic Models as Input

For complex inputs, use Pydantic:

```python
from pydantic import BaseModel, Field

class EmailRequest(BaseModel):
    to: str = Field(description="Recipient email address")
    subject: str = Field(description="Email subject line")
    body: str = Field(description="Email body content")
    cc: Optional[List[str]] = Field(default=None, description="CC recipients")

@mcp.tool()
def send_email(request: EmailRequest) -> str:
    """Send an email with the specified parameters."""
    # FastMCP deserializes arguments into EmailRequest automatically
    print(f"Sending to {request.to}: {request.subject}")
    return f"Email sent to {request.to}"
```

### 7.4 Returning Rich Content

Tools can return more than strings:

```python
from mcp.types import TextContent, ImageContent, EmbeddedResource

@mcp.tool()
def get_chart_data(metric: str) -> dict:
    """Get chart data for a metric."""
    # Return dict — FastMCP converts to TextContent JSON
    return {"labels": ["Jan", "Feb"], "values": [100, 120]}

# For explicit content types, use the low-level approach:
@server.call_tool()
async def call_tool(name, arguments):
    if name == "get_image":
        return [
            ImageContent(
                type="image",
                data="base64encodeddata...",
                mimeType="image/png"
            )
        ]
```

### 7.5 Async Tools

All FastMCP tools can be async:

```python
import httpx

@mcp.tool()
async def fetch_weather(city: str) -> str:
    """Fetch current weather for a city."""
    async with httpx.AsyncClient() as client:
        resp = await client.get(
            f"https://wttr.in/{city}?format=3"
        )
        return resp.text
```

### 7.6 Context Object — Accessing MCP Features Inside Tools

FastMCP tools can receive a `Context` parameter for advanced features:

```python
from mcp.server.fastmcp import FastMCP, Context

mcp = FastMCP("Context Demo")

@mcp.tool()
async def long_running_task(steps: int, ctx: Context) -> str:
    """Run a multi-step task with progress reporting."""
    
    results = []
    for i in range(steps):
        # Report progress to the client
        await ctx.report_progress(i, steps)
        
        # Log a message (shows in MCP Inspector)
        await ctx.info(f"Processing step {i+1}/{steps}")
        
        # Read a resource from within the tool
        data = await ctx.read_resource("myapp://config/settings")
        
        results.append(f"Step {i+1} complete")
    
    return "\n".join(results)
```

**Context methods:**

| Method | Purpose |
|---|---|
| `ctx.info(msg)` | Log info message |
| `ctx.warning(msg)` | Log warning |
| `ctx.error(msg)` | Log error |
| `ctx.debug(msg)` | Log debug |
| `ctx.report_progress(current, total)` | Send progress update |
| `ctx.read_resource(uri)` | Read a server resource |
| `ctx.request_context` | Raw request metadata |

---

## 8. Resources — Exposing Data

Resources provide **read access to data**. The host/user decides when to include them in context.

### 8.1 Static Resource

```python
@mcp.resource("myapp://config/settings")
def get_settings() -> str:
    """Application configuration settings."""
    return """
DATABASE_URL=localhost:5432
MAX_CONNECTIONS=10
DEBUG=false
"""
```

### 8.2 Dynamic Resource with URI Templates

```python
@mcp.resource("users://{user_id}/profile")
def get_user_profile(user_id: str) -> str:
    """Get a user's profile by ID."""
    # Fetch from database
    user = db.get_user(user_id)
    return f"Name: {user.name}\nEmail: {user.email}\nCreated: {user.created_at}"
```

URI template parameters are automatically extracted and passed as function arguments.

### 8.3 Resource with MIME Type

```python
@mcp.resource("reports://{month}/summary", mime_type="application/json")
def get_monthly_report(month: str) -> str:
    """Get monthly sales report as JSON."""
    data = {
        "month": month,
        "revenue": 150000,
        "units_sold": 342
    }
    import json
    return json.dumps(data, indent=2)
```

### 8.4 Resource Returning Binary (Low-Level)

```python
import base64
from mcp.types import BlobResourceContents

@server.read_resource()
async def read_resource(uri: str):
    if uri.startswith("images://"):
        image_id = uri.split("/")[-1]
        with open(f"images/{image_id}.png", "rb") as f:
            data = base64.b64encode(f.read()).decode()
        return BlobResourceContents(
            uri=uri,
            mimeType="image/png",
            blob=data
        )
```

### 8.5 Resource Subscriptions (Live Updates)

When `resources.subscribe` capability is declared, clients can subscribe to resource changes:

```python
# Low-level: notify clients when resource changes
await server.request_context.session.send_resource_updated(
    uri="myapp://data/live-feed"
)
```

---

## 9. Prompts — Reusable Templates

Prompts are predefined message sequences users can invoke.

### 9.1 Simple Prompt

```python
from mcp.server.fastmcp import FastMCP
from mcp.types import PromptMessage, TextContent

mcp = FastMCP("Prompt Demo")

@mcp.prompt()
def code_review() -> str:
    """Request a detailed code review."""
    return "Please review the following code for bugs, security issues, and best practices."
```

### 9.2 Prompt with Arguments

```python
@mcp.prompt()
def analyze_data(dataset_name: str, analysis_type: str = "summary") -> str:
    """Generate a data analysis prompt for a specific dataset."""
    return f"""
Analyze the {dataset_name} dataset with a focus on {analysis_type} analysis.

Please provide:
1. Key statistics and distributions
2. Notable patterns or anomalies
3. Actionable insights
4. Recommended next steps
"""
```

### 9.3 Multi-Message Prompt (Low-Level)

```python
from mcp.types import GetPromptResult, PromptMessage, Role

@server.get_prompt()
async def get_prompt(name: str, arguments: dict):
    if name == "debug_session":
        error = arguments.get("error", "Unknown error")
        return GetPromptResult(
            description="Start a debugging session",
            messages=[
                PromptMessage(
                    role=Role.user,
                    content=TextContent(
                        type="text",
                        text=f"I'm encountering this error: {error}"
                    )
                ),
                PromptMessage(
                    role=Role.assistant,
                    content=TextContent(
                        type="text",
                        text="I'll help you debug this. Let me analyze the error..."
                    )
                ),
                PromptMessage(
                    role=Role.user,
                    content=TextContent(
                        type="text",
                        text="Please also check my recent code changes."
                    )
                )
            ]
        )
```

---

## 10. Medium: Multi-Tool Servers with State

Real servers have multiple tools, share state, and interact with external services.

### 10.1 Todo List Server (Complete Example)

```python
# todo_server.py
from mcp.server.fastmcp import FastMCP
from typing import List, Optional
from datetime import datetime
import json, os

mcp = FastMCP("Todo Manager")

# In-memory state (use a DB for production)
todos: List[dict] = []
_counter = 0

def _next_id() -> int:
    global _counter
    _counter += 1
    return _counter

@mcp.tool()
def add_todo(
    title: str,
    description: Optional[str] = None,
    priority: str = "medium",
    due_date: Optional[str] = None
) -> dict:
    """
    Add a new todo item.
    
    Args:
        title: Short title of the task
        description: Detailed description (optional)
        priority: 'low', 'medium', or 'high'
        due_date: ISO date string like '2025-12-31'
    """
    if priority not in ("low", "medium", "high"):
        raise ValueError(f"Invalid priority '{priority}'. Use: low, medium, high")
    
    todo = {
        "id": _next_id(),
        "title": title,
        "description": description,
        "priority": priority,
        "due_date": due_date,
        "completed": False,
        "created_at": datetime.now().isoformat()
    }
    todos.append(todo)
    return todo

@mcp.tool()
def list_todos(
    filter_status: Optional[str] = None,
    filter_priority: Optional[str] = None
) -> List[dict]:
    """
    List all todo items with optional filters.
    
    Args:
        filter_status: 'completed' or 'pending' to filter by status
        filter_priority: 'low', 'medium', or 'high' to filter by priority
    """
    result = todos.copy()
    
    if filter_status == "completed":
        result = [t for t in result if t["completed"]]
    elif filter_status == "pending":
        result = [t for t in result if not t["completed"]]
    
    if filter_priority:
        result = [t for t in result if t["priority"] == filter_priority]
    
    return result

@mcp.tool()
def complete_todo(todo_id: int) -> dict:
    """Mark a todo item as completed by its ID."""
    for todo in todos:
        if todo["id"] == todo_id:
            todo["completed"] = True
            todo["completed_at"] = datetime.now().isoformat()
            return todo
    raise ValueError(f"Todo with id {todo_id} not found")

@mcp.tool()
def delete_todo(todo_id: int) -> str:
    """Delete a todo item by its ID."""
    global todos
    initial_len = len(todos)
    todos = [t for t in todos if t["id"] != todo_id]
    if len(todos) == initial_len:
        raise ValueError(f"Todo with id {todo_id} not found")
    return f"Deleted todo {todo_id}"

# Expose todos as a resource
@mcp.resource("todos://all")
def get_all_todos() -> str:
    """All todo items as JSON."""
    return json.dumps(todos, indent=2)

@mcp.resource("todos://summary")
def get_todo_summary() -> str:
    """Summary statistics for todos."""
    total = len(todos)
    completed = sum(1 for t in todos if t["completed"])
    pending = total - completed
    high_priority = sum(1 for t in todos if t["priority"] == "high" and not t["completed"])
    return f"""Todo Summary
Total: {total}
Completed: {completed}
Pending: {pending}
High Priority Pending: {high_priority}
"""

# A helpful prompt
@mcp.prompt()
def daily_review() -> str:
    """Prompt for reviewing today's todos."""
    return "Please review my todo list and help me prioritize tasks for today based on priority and due dates."

if __name__ == "__main__":
    mcp.run()
```

### 10.2 Database-Backed Server

```python
# db_server.py
import sqlite3
from contextlib import contextmanager
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("Database Server")
DB_PATH = "app.db"

@contextmanager
def get_db():
    conn = sqlite3.connect(DB_PATH)
    conn.row_factory = sqlite3.Row
    try:
        yield conn
        conn.commit()
    except Exception:
        conn.rollback()
        raise
    finally:
        conn.close()

# Initialize schema on startup
with get_db() as conn:
    conn.execute("""
        CREATE TABLE IF NOT EXISTS notes (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            title TEXT NOT NULL,
            content TEXT,
            tags TEXT,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """)

@mcp.tool()
def create_note(title: str, content: str, tags: Optional[str] = None) -> dict:
    """Create a new note and save it to the database."""
    with get_db() as conn:
        cursor = conn.execute(
            "INSERT INTO notes (title, content, tags) VALUES (?, ?, ?)",
            (title, content, tags)
        )
        return {"id": cursor.lastrowid, "title": title, "created": True}

@mcp.tool()
def search_notes(query: str) -> list:
    """Full-text search across note titles and content."""
    with get_db() as conn:
        rows = conn.execute(
            "SELECT * FROM notes WHERE title LIKE ? OR content LIKE ?",
            (f"%{query}%", f"%{query}%")
        ).fetchall()
        return [dict(row) for row in rows]

@mcp.resource("notes://{note_id}")
def get_note(note_id: str) -> str:
    """Get a specific note by ID."""
    with get_db() as conn:
        row = conn.execute(
            "SELECT * FROM notes WHERE id = ?", (note_id,)
        ).fetchone()
        if not row:
            return "Note not found"
        d = dict(row)
        return f"# {d['title']}\nTags: {d['tags']}\n\n{d['content']}"
```

---

## 11. Advanced: Full Production Server

### 11.1 Weather + Geocoding Server with API Integration

```python
# weather_server.py
import httpx
import os
from typing import Optional
from mcp.server.fastmcp import FastMCP, Context
from pydantic import BaseModel

mcp = FastMCP("Weather Service", dependencies=["httpx"])

WEATHER_API_KEY = os.environ.get("OPENWEATHER_API_KEY", "")
BASE_URL = "https://api.openweathermap.org/data/2.5"

class WeatherResult(BaseModel):
    city: str
    country: str
    temperature_c: float
    feels_like_c: float
    humidity_percent: int
    condition: str
    wind_speed_ms: float
    visibility_km: float

@mcp.tool()
async def get_current_weather(
    city: str,
    country_code: Optional[str] = None,
    ctx: Context = None
) -> WeatherResult:
    """
    Get current weather conditions for a city.
    
    Args:
        city: City name (e.g., 'Hyderabad', 'London')
        country_code: ISO 3166 country code (e.g., 'IN', 'GB') to disambiguate
    """
    query = f"{city},{country_code}" if country_code else city
    
    if ctx:
        await ctx.info(f"Fetching weather for: {query}")
    
    async with httpx.AsyncClient(timeout=10.0) as client:
        resp = await client.get(
            f"{BASE_URL}/weather",
            params={"q": query, "appid": WEATHER_API_KEY, "units": "metric"}
        )
        resp.raise_for_status()
        data = resp.json()
    
    return WeatherResult(
        city=data["name"],
        country=data["sys"]["country"],
        temperature_c=data["main"]["temp"],
        feels_like_c=data["main"]["feels_like"],
        humidity_percent=data["main"]["humidity"],
        condition=data["weather"][0]["description"].title(),
        wind_speed_ms=data["wind"]["speed"],
        visibility_km=data.get("visibility", 0) / 1000
    )

@mcp.tool()
async def get_forecast(city: str, days: int = 5) -> list:
    """
    Get weather forecast for the next N days.
    
    Args:
        city: City name
        days: Number of days (1-5)
    """
    days = min(max(days, 1), 5)
    
    async with httpx.AsyncClient(timeout=10.0) as client:
        resp = await client.get(
            f"{BASE_URL}/forecast",
            params={"q": city, "appid": WEATHER_API_KEY, "units": "metric", "cnt": days * 8}
        )
        resp.raise_for_status()
        data = resp.json()
    
    # Group by day
    from collections import defaultdict
    by_day = defaultdict(list)
    for item in data["list"]:
        day = item["dt_txt"].split(" ")[0]
        by_day[day].append(item)
    
    result = []
    for day, items in list(by_day.items())[:days]:
        temps = [i["main"]["temp"] for i in items]
        result.append({
            "date": day,
            "min_temp_c": round(min(temps), 1),
            "max_temp_c": round(max(temps), 1),
            "conditions": list(set(i["weather"][0]["description"] for i in items))
        })
    
    return result

@mcp.resource("weather://cache/{city}")
async def get_cached_weather(city: str) -> str:
    """Return cached weather data if available."""
    # In production, check Redis/memcache
    return f"No cached data for {city}"

@mcp.prompt()
def weather_trip_advisor(destination: str, travel_date: str) -> str:
    """Generate a prompt for travel weather advice."""
    return f"""I'm planning to travel to {destination} on {travel_date}.
Please check the weather forecast and advise me on:
1. What to pack
2. Best time of day for outdoor activities
3. Any weather warnings I should know about"""

if __name__ == "__main__":
    mcp.run()
```

### 11.2 Lifespan Management (Startup/Shutdown)

```python
from contextlib import asynccontextmanager
from mcp.server.fastmcp import FastMCP

# Use lifespan for resources that need setup/teardown
@asynccontextmanager
async def lifespan(server):
    # STARTUP
    print("Server starting — connecting to DB...")
    db = await create_db_pool()
    server.state["db"] = db
    
    yield  # Server is running here
    
    # SHUTDOWN
    print("Server stopping — closing DB pool...")
    await db.close()

mcp = FastMCP("Production Server", lifespan=lifespan)

@mcp.tool()
async def query_db(sql: str, ctx: Context) -> list:
    """Execute a read-only SQL query."""
    db = ctx.request_context.lifespan_context["db"]
    results = await db.fetch(sql)
    return [dict(r) for r in results]
```

### 11.3 Dependency Injection Pattern

```python
from functools import lru_cache
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("Injected Server")

class AppConfig:
    def __init__(self):
        self.db_url = os.environ["DATABASE_URL"]
        self.api_key = os.environ["API_KEY"]
        self.cache_ttl = int(os.environ.get("CACHE_TTL", "300"))

@lru_cache
def get_config() -> AppConfig:
    return AppConfig()

@mcp.tool()
def get_service_status() -> dict:
    """Get current service configuration status."""
    config = get_config()
    return {
        "db_connected": True,
        "cache_ttl": config.cache_ttl,
        "api_configured": bool(config.api_key)
    }
```

---

## 12. Transport Layer

MCP supports multiple transport mechanisms.

### 12.1 stdio (Standard Input/Output)

**Most common.** Process communicates via stdin/stdout. Used by Claude Desktop.

```python
# Default — stdio is automatic with mcp.run()
mcp.run(transport="stdio")
```

```python
# Explicit stdio setup (low-level)
from mcp.server.stdio import stdio_server
import asyncio

async def main():
    async with stdio_server() as (read_stream, write_stream):
        await server.run(read_stream, write_stream, init_options)

asyncio.run(main())
```

**When to use:** Local tools, Claude Desktop integrations.

### 12.2 SSE (Server-Sent Events over HTTP)

For **web-based** deployments where the client connects over HTTP.

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("HTTP Server")

# Tools defined here...

if __name__ == "__main__":
    mcp.run(transport="sse")  # Starts HTTP server on port 8000
```

```python
# With custom host/port
mcp.run(transport="sse", host="0.0.0.0", port=8080)
```

### 12.3 Streamable HTTP (Newer Standard)

Replaces SSE in newer MCP versions; uses a single HTTP endpoint with streaming:

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("Streamable Server")

if __name__ == "__main__":
    mcp.run(transport="http", port=8000)
```

### 12.4 Transport Comparison

| Transport | Use Case | Client Location | Auth |
|---|---|---|---|
| stdio | Local tools, desktop apps | Same machine | OS process isolation |
| SSE | Web deployments | Remote/browser | HTTP headers |
| Streamable HTTP | Web deployments (modern) | Remote | HTTP headers |

### 12.5 Connecting a Client (Python)

```python
# client.py — useful for testing or building host apps
import asyncio
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async def main():
    server_params = StdioServerParameters(
        command="python",
        args=["server.py"]
    )
    
    async with stdio_client(server_params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()
            
            # List available tools
            tools = await session.list_tools()
            print("Tools:", [t.name for t in tools.tools])
            
            # Call a tool
            result = await session.call_tool("hello", {"name": "World"})
            print("Result:", result.content[0].text)
            
            # List resources
            resources = await session.list_resources()
            print("Resources:", [r.uri for r in resources.resources])
            
            # Read a resource
            content = await session.read_resource("todos://all")
            print("Resource:", content.contents[0].text)

asyncio.run(main())
```

---

## 13. Error Handling & Validation

### 13.1 Tool Errors

Tools should raise exceptions for user-facing errors:

```python
@mcp.tool()
def divide(a: float, b: float) -> float:
    """Divide a by b."""
    if b == 0:
        raise ValueError("Cannot divide by zero")
    return a / b
```

FastMCP catches the exception and returns it as a tool error, which the LLM can handle gracefully.

### 13.2 Structured Error Responses (Low-Level)

```python
from mcp.types import CallToolResult, TextContent

@server.call_tool()
async def call_tool(name, arguments):
    try:
        result = await execute_tool(name, arguments)
        return CallToolResult(content=[TextContent(type="text", text=result)])
    except ValueError as e:
        return CallToolResult(
            content=[TextContent(type="text", text=str(e))],
            isError=True
        )
```

### 13.3 Input Validation with Pydantic

```python
from pydantic import BaseModel, Field, field_validator

class SearchQuery(BaseModel):
    query: str = Field(min_length=1, max_length=500, description="Search query")
    max_results: int = Field(default=10, ge=1, le=100)
    sort_by: str = Field(default="relevance")
    
    @field_validator("sort_by")
    @classmethod
    def valid_sort(cls, v):
        allowed = {"relevance", "date", "popularity"}
        if v not in allowed:
            raise ValueError(f"sort_by must be one of: {allowed}")
        return v

@mcp.tool()
def search(params: SearchQuery) -> list:
    """Search with validated parameters."""
    # Pydantic validation runs before your function
    return perform_search(params.query, params.max_results, params.sort_by)
```

### 13.4 Timeout Handling

```python
import asyncio

@mcp.tool()
async def slow_operation(timeout_seconds: int = 30) -> str:
    """Operation with configurable timeout."""
    try:
        result = await asyncio.wait_for(
            _actual_slow_work(),
            timeout=timeout_seconds
        )
        return result
    except asyncio.TimeoutError:
        raise TimeoutError(f"Operation timed out after {timeout_seconds}s")
```

---

## 14. Authentication & Security

### 14.1 Environment Variable Pattern

Never hardcode credentials. Use environment variables:

```python
import os
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("Secure Server")

# Fail fast if required config is missing
API_KEY = os.environ.get("MY_API_KEY")
if not API_KEY:
    raise RuntimeError("MY_API_KEY environment variable is required")

@mcp.tool()
async def call_api(endpoint: str) -> str:
    """Call external API with authentication."""
    async with httpx.AsyncClient() as client:
        resp = await client.get(
            f"https://api.example.com/{endpoint}",
            headers={"Authorization": f"Bearer {API_KEY}"}
        )
        return resp.json()
```

### 14.2 Rate Limiting Pattern

```python
import time
from collections import defaultdict

# Simple in-memory rate limiter
call_times = defaultdict(list)
RATE_LIMIT = 10  # calls per minute

def check_rate_limit(tool_name: str):
    now = time.time()
    window = 60  # 1 minute
    
    # Remove calls outside the window
    call_times[tool_name] = [
        t for t in call_times[tool_name] if now - t < window
    ]
    
    if len(call_times[tool_name]) >= RATE_LIMIT:
        raise Exception(f"Rate limit exceeded for {tool_name}: {RATE_LIMIT}/min")
    
    call_times[tool_name].append(now)

@mcp.tool()
def rate_limited_search(query: str) -> list:
    """Search with rate limiting."""
    check_rate_limit("search")
    return do_search(query)
```

### 14.3 Input Sanitization

```python
import re

def sanitize_for_sql(value: str) -> str:
    """Remove SQL injection risks."""
    # Use parameterized queries instead of this in production!
    return re.sub(r"[;'\"\-\-]", "", value)

def sanitize_path(path: str, allowed_base: str) -> str:
    """Prevent path traversal attacks."""
    import os
    full_path = os.path.realpath(os.path.join(allowed_base, path))
    if not full_path.startswith(allowed_base):
        raise ValueError("Path traversal detected")
    return full_path

@mcp.tool()
def read_file(filename: str) -> str:
    """Read a file from the allowed directory."""
    safe_path = sanitize_path(filename, "/app/allowed_files")
    with open(safe_path) as f:
        return f.read()
```

---

## 15. Sampling — LLM-in-Server Patterns

**Sampling** allows your MCP server to request completions from the LLM. This is an advanced feature that makes the server "AI-powered."

### 15.1 When to Use Sampling

- Summarizing data before returning it
- Generating dynamic content based on context
- Multi-step reasoning where the server needs LLM judgment
- Building AI agents that chain LLM calls

### 15.2 Sampling Request

```python
from mcp.server import Server
from mcp.types import (
    CreateMessageRequest,
    CreateMessageResult,
    SamplingMessage,
    TextContent,
    Role
)

server = Server("Sampling Server")

@server.call_tool()
async def call_tool(name, arguments):
    if name == "summarize_and_analyze":
        raw_data = get_raw_data(arguments["source"])
        
        # Ask the LLM to summarize
        response: CreateMessageResult = await server.request_context.session.create_message(
            CreateMessageRequest(
                messages=[
                    SamplingMessage(
                        role=Role.user,
                        content=TextContent(
                            type="text",
                            text=f"Summarize this data in 3 bullet points:\n\n{raw_data}"
                        )
                    )
                ],
                maxTokens=500,
                systemPrompt="You are a data analysis assistant. Be concise."
            )
        )
        
        summary = response.content.text
        return [TextContent(type="text", text=f"Summary:\n{summary}")]
```

### 15.3 Sampling with FastMCP Context

```python
@mcp.tool()
async def smart_summary(topic: str, ctx: Context) -> str:
    """Generate an AI-powered summary of a topic from our knowledge base."""
    
    # First, fetch raw data
    raw = await fetch_knowledge_base(topic)
    
    # Then use sampling to process it intelligently
    result = await ctx.sample(
        f"Summarize the following in exactly 5 bullet points:\n\n{raw}",
        max_tokens=300
    )
    
    return result.text
```

> **Note:** Sampling requires the host (Claude Desktop) to support and allow it. The LLM in the host processes the request — your server is essentially asking the LLM to do work on your behalf.

---

## 16. Roots — File System Context

**Roots** tell the server which directories the client considers relevant. Useful for file-based tools that need to scope their operations.

### 16.1 Listing Roots

```python
@server.list_roots()
async def list_roots():
    # The client will call this to tell the server its workspace roots
    pass  # Roots are provided by the client, not the server

# Access roots in a tool
@server.call_tool()
async def call_tool(name, arguments):
    if name == "find_in_workspace":
        # Get roots from session
        roots = await server.request_context.session.list_roots()
        
        results = []
        for root in roots.roots:
            root_path = root.uri.replace("file://", "")
            # Search within this root
            results.extend(search_directory(root_path, arguments["query"]))
        
        return [TextContent(type="text", text=str(results))]
```

### 16.2 Roots with FastMCP

```python
from mcp.server.fastmcp import FastMCP, Context

mcp = FastMCP("File Server")

@mcp.tool()
async def list_workspace_files(ctx: Context) -> list:
    """List all files in the client's workspace roots."""
    import os
    
    # Get roots declared by the client
    roots_response = await ctx.request_context.session.list_roots()
    
    all_files = []
    for root in roots_response.roots:
        path = root.uri.replace("file://", "")
        if os.path.isdir(path):
            for f in os.listdir(path):
                all_files.append(f"{path}/{f}")
    
    return all_files
```

---

## 17. Logging & Observability

### 17.1 Server-Side Logging

```python
import logging
from mcp.server.fastmcp import FastMCP

# Standard Python logging works
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

mcp = FastMCP("Observable Server")

@mcp.tool()
async def tracked_operation(input_data: str, ctx: Context) -> str:
    """Operation with full observability."""
    
    logger.info(f"Operation started: input_length={len(input_data)}")
    
    # MCP protocol logging (visible to client/inspector)
    await ctx.info(f"Processing {len(input_data)} characters")
    
    try:
        result = process(input_data)
        logger.info(f"Operation succeeded: output_length={len(result)}")
        await ctx.info("Processing complete")
        return result
    except Exception as e:
        logger.error(f"Operation failed: {e}", exc_info=True)
        await ctx.error(f"Processing failed: {e}")
        raise
```

### 17.2 Structured Logging

```python
import json
from datetime import datetime

def log_tool_call(tool_name: str, args: dict, result=None, error=None):
    entry = {
        "timestamp": datetime.now().isoformat(),
        "tool": tool_name,
        "args": {k: str(v)[:100] for k, v in args.items()},  # truncate
        "success": error is None,
        "error": str(error) if error else None
    }
    print(json.dumps(entry), flush=True)  # Goes to stderr if using stdio

@mcp.tool()
def instrumented_tool(query: str) -> str:
    """Tool with structured logging."""
    try:
        result = do_work(query)
        log_tool_call("instrumented_tool", {"query": query}, result=result)
        return result
    except Exception as e:
        log_tool_call("instrumented_tool", {"query": query}, error=e)
        raise
```

---

## 18. Testing MCP Servers

### 18.1 Unit Testing Tools Directly

```python
# test_server.py
import pytest
from todo_server import add_todo, list_todos, complete_todo, todos

@pytest.fixture(autouse=True)
def clear_todos():
    """Clear todos before each test."""
    todos.clear()
    yield

def test_add_todo():
    result = add_todo("Buy milk", priority="high")
    assert result["title"] == "Buy milk"
    assert result["priority"] == "high"
    assert result["completed"] == False
    assert "id" in result

def test_list_todos_filtered():
    add_todo("Task 1", priority="high")
    add_todo("Task 2", priority="low")
    
    high_priority = list_todos(filter_priority="high")
    assert len(high_priority) == 1
    assert high_priority[0]["title"] == "Task 1"

def test_complete_todo():
    todo = add_todo("Test task")
    result = complete_todo(todo["id"])
    assert result["completed"] == True

def test_complete_nonexistent():
    with pytest.raises(ValueError, match="not found"):
        complete_todo(9999)
```

### 18.2 Integration Testing with MCP Client

```python
# test_integration.py
import pytest
import asyncio
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

@pytest.fixture
async def mcp_session():
    """Fixture providing a live MCP session."""
    params = StdioServerParameters(command="python", args=["server.py"])
    async with stdio_client(params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()
            yield session

@pytest.mark.asyncio
async def test_tool_list(mcp_session):
    tools = await mcp_session.list_tools()
    tool_names = [t.name for t in tools.tools]
    assert "add_todo" in tool_names
    assert "list_todos" in tool_names

@pytest.mark.asyncio
async def test_add_and_list(mcp_session):
    # Add a todo
    add_result = await mcp_session.call_tool(
        "add_todo",
        {"title": "Integration test task", "priority": "medium"}
    )
    assert add_result.isError is not True
    
    # List todos
    list_result = await mcp_session.call_tool("list_todos", {})
    import json
    todos = json.loads(list_result.content[0].text)
    assert any(t["title"] == "Integration test task" for t in todos)
```

### 18.3 Using MCP Inspector for Manual Testing

```bash
# Start inspector with your server
npx @modelcontextprotocol/inspector python server.py

# With environment variables
MY_API_KEY=test123 npx @modelcontextprotocol/inspector python server.py

# With arguments
npx @modelcontextprotocol/inspector python server.py --db-path ./test.db
```

The Inspector provides:
- **Tools tab:** List and call all tools with a form UI
- **Resources tab:** Browse and read resources
- **Prompts tab:** View and invoke prompts
- **History tab:** Full request/response log

---

## 19. Rarely Used but Powerful Concepts

### 19.1 Tool Annotations (Hints for Hosts)

Annotations tell the host how to treat a tool — doesn't affect the LLM directly:

```python
from mcp.server.fastmcp import FastMCP
from mcp.types import Tool

# Low-level: add annotations to tool
tool = Tool(
    name="delete_all_data",
    description="Delete all data from the system",
    inputSchema={"type": "object", "properties": {}},
    annotations={
        "destructive": True,        # Host should warn before calling
        "readOnly": False,          # Modifies data
        "idempotent": False,        # Can't be safely retried
        "openWorld": False,         # Has predictable scope
        "requiresConfirmation": True  # Custom: host should confirm
    }
)
```

### 19.2 Progress Tokens

For long operations, track progress with client-provided tokens:

```python
@server.call_tool()
async def call_tool(name, arguments):
    if name == "batch_process":
        # Get progress token from meta
        meta = server.request_context.meta
        progress_token = meta.progressToken if meta else None
        
        items = arguments["items"]
        for i, item in enumerate(items):
            result = process_item(item)
            
            if progress_token:
                await server.request_context.session.send_progress_notification(
                    progressToken=progress_token,
                    progress=i + 1,
                    total=len(items)
                )
        
        return [TextContent(type="text", text="Batch complete")]
```

### 19.3 Resource Templates (Dynamic Resource Discovery)

```python
from mcp.types import ResourceTemplate

@server.list_resource_templates()
async def list_resource_templates():
    return [
        ResourceTemplate(
            uriTemplate="users://{user_id}/data/{data_type}",
            name="User Data",
            description="Access user data by type",
            mimeType="application/json"
        )
    ]
```

### 19.4 Completion/Autocomplete for Arguments

MCP supports argument autocompletion for prompts and resource templates:

```python
from mcp.types import Completion, CompletionArgument

@server.complete()
async def complete(ref, argument: CompletionArgument):
    # Provide autocomplete for prompt arguments
    if ref.name == "analyze_data" and argument.name == "dataset_name":
        # Return available dataset names
        datasets = get_available_datasets()
        matches = [d for d in datasets if d.startswith(argument.value)]
        return Completion(values=matches[:10], hasMore=len(matches) > 10)
```

### 19.5 Elicitation (Requesting User Input Mid-Tool)

A newer MCP feature: tools can pause and ask the user for more information:

```python
from mcp.types import ElicitRequest, ElicitResult

@server.call_tool()
async def call_tool(name, arguments):
    if name == "delete_records":
        # Pause and ask user to confirm
        result: ElicitResult = await server.request_context.session.elicit(
            ElicitRequest(
                message="Are you sure you want to delete all records? This cannot be undone.",
                requestedSchema={
                    "type": "object",
                    "properties": {
                        "confirm": {
                            "type": "boolean",
                            "description": "Type true to confirm deletion"
                        }
                    },
                    "required": ["confirm"]
                }
            )
        )
        
        if result.action == "accept" and result.content.get("confirm"):
            delete_all_records()
            return [TextContent(type="text", text="All records deleted")]
        else:
            return [TextContent(type="text", text="Deletion cancelled")]
```

### 19.6 Custom Content Types

Beyond text and images, MCP supports embedded resources:

```python
from mcp.types import EmbeddedResource, TextResourceContents

@server.call_tool()
async def call_tool(name, arguments):
    if name == "get_document_with_attachments":
        return [
            TextContent(type="text", text="Here is the document:"),
            EmbeddedResource(
                type="resource",
                resource=TextResourceContents(
                    uri="docs://report-2025.txt",
                    mimeType="text/plain",
                    text="Full report content here..."
                )
            )
        ]
```

### 19.7 Multiple Servers in One Process

```python
# Run two FastMCP servers in one process
from mcp.server.fastmcp import FastMCP

server_a = FastMCP("Server A")
server_b = FastMCP("Server B")

@server_a.tool()
def tool_a() -> str:
    return "From A"

@server_b.tool()
def tool_b() -> str:
    return "From B"

# Mount B under A as a sub-application
server_a.mount("/b", server_b)
```

---

## 20. Common Architectures & Workflow Patterns

### 20.1 The Wrapper Architecture

Expose an existing API as MCP tools:

```
External API ← HTTP → MCP Server → MCP Client → LLM
```

```python
# Pattern: API Wrapper Server
mcp = FastMCP("Stripe Wrapper")

STRIPE_KEY = os.environ["STRIPE_SECRET_KEY"]

@mcp.tool()
async def list_customers(limit: int = 10) -> list:
    """List Stripe customers."""
    async with httpx.AsyncClient() as client:
        resp = await client.get(
            "https://api.stripe.com/v1/customers",
            auth=(STRIPE_KEY, ""),
            params={"limit": limit}
        )
        return resp.json()["data"]
```

### 20.2 The Proxy Architecture

Route tool calls to different backends:

```python
mcp = FastMCP("Smart Router")

@mcp.tool()
async def query(question: str) -> str:
    """Route query to appropriate backend."""
    
    # Simple keyword routing
    if any(w in question.lower() for w in ["sql", "database", "table"]):
        return await _query_database(question)
    elif any(w in question.lower() for w in ["file", "document", "read"]):
        return await _query_filesystem(question)
    else:
        return await _query_knowledge_base(question)
```

### 20.3 The Aggregator Pattern

Combine multiple data sources in one tool:

```python
import asyncio

@mcp.tool()
async def get_customer_360(customer_id: str) -> dict:
    """
    Get complete 360-degree view of a customer by aggregating
    data from CRM, billing, support, and analytics systems.
    """
    # Fetch from all systems concurrently
    crm, billing, support, analytics = await asyncio.gather(
        fetch_crm_data(customer_id),
        fetch_billing_data(customer_id),
        fetch_support_tickets(customer_id),
        fetch_usage_analytics(customer_id)
    )
    
    return {
        "profile": crm,
        "billing": billing,
        "support_history": support,
        "usage": analytics,
        "health_score": calculate_health(crm, billing, support, analytics)
    }
```

### 20.4 The Agent Loop Pattern

Build agentic workflows where tools call each other:

```python
@mcp.tool()
async def research_and_summarize(topic: str, ctx: Context) -> str:
    """
    Performs multi-step research: searches, reads pages, then summarizes.
    """
    await ctx.info("Step 1: Searching for sources...")
    
    # Step 1: Get search results
    search_results = await web_search(topic)
    urls = extract_urls(search_results)
    
    await ctx.info(f"Step 2: Reading {len(urls)} pages...")
    await ctx.report_progress(1, 3)
    
    # Step 2: Fetch each page
    contents = []
    for url in urls[:3]:
        content = await fetch_url(url)
        contents.append(content)
    
    await ctx.report_progress(2, 3)
    await ctx.info("Step 3: Synthesizing information...")
    
    # Step 3: Combine and return
    combined = "\n\n---\n\n".join(contents)
    summary = f"Research on '{topic}':\n\n{combined[:5000]}"
    
    await ctx.report_progress(3, 3)
    return summary
```

### 20.5 The Cache-Aside Pattern

```python
import json, hashlib
from datetime import datetime, timedelta

_cache = {}  # Use Redis in production

def cache_key(func_name: str, **kwargs) -> str:
    data = json.dumps({"fn": func_name, **kwargs}, sort_keys=True)
    return hashlib.md5(data.encode()).hexdigest()

def cached(ttl_seconds: int = 300):
    """Decorator for caching tool results."""
    def decorator(func):
        async def wrapper(*args, **kwargs):
            key = cache_key(func.__name__, **kwargs)
            
            if key in _cache:
                value, expires = _cache[key]
                if datetime.now() < expires:
                    return value
            
            result = await func(*args, **kwargs)
            _cache[key] = (result, datetime.now() + timedelta(seconds=ttl_seconds))
            return result
        
        return wrapper
    return decorator

@mcp.tool()
@cached(ttl_seconds=60)
async def get_stock_price(ticker: str) -> dict:
    """Get current stock price (cached for 60 seconds)."""
    return await fetch_stock_api(ticker)
```

---

## 21. MCP Registry & Distribution

### 21.1 Publishing Your Server

A distributable MCP server has:
1. A `pyproject.toml` or `package.json`
2. An entry point
3. Documented environment variables
4. A README

```toml
# pyproject.toml
[project]
name = "mcp-weather-server"
version = "1.0.0"
description = "MCP server for weather data"
dependencies = ["mcp>=1.0.0", "httpx>=0.27.0"]

[project.scripts]
mcp-weather = "weather_server:main"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

```python
# weather_server.py — add entry point
def main():
    mcp.run()

if __name__ == "__main__":
    main()
```

### 21.2 Installation Instructions Template

```markdown
## Installation

\`\`\`bash
pip install mcp-weather-server
\`\`\`

## Claude Desktop Config

\`\`\`json
{
  "mcpServers": {
    "weather": {
      "command": "mcp-weather",
      "env": {
        "OPENWEATHER_API_KEY": "your-key-here"
      }
    }
  }
}
\`\`\`
```

### 21.3 NPX-Installable Server (Node.js)

```json
// package.json
{
  "name": "@myorg/mcp-server-weather",
  "bin": {
    "mcp-server-weather": "./dist/index.js"
  }
}
```

```json
// Claude Desktop
{
  "mcpServers": {
    "weather": {
      "command": "npx",
      "args": ["-y", "@myorg/mcp-server-weather"],
      "env": { "API_KEY": "xxx" }
    }
  }
}
```

---

## 22. Reference Cheatsheet

### FastMCP Decorators

| Decorator | Purpose | Example |
|---|---|---|
| `@mcp.tool()` | Register a callable tool | `@mcp.tool() def search(q: str) -> list` |
| `@mcp.resource(uri)` | Expose data at a URI | `@mcp.resource("data://config")` |
| `@mcp.prompt()` | Register a prompt template | `@mcp.prompt() def review() -> str` |

### FastMCP Constructor Options

```python
mcp = FastMCP(
    name="My Server",              # Server name
    version="1.0.0",               # Server version
    instructions="Use this for...",# Instructions for the LLM
    dependencies=["httpx", "pydantic"],  # pip packages to declare
    lifespan=my_lifespan_context,  # Async context manager
)
```

### Context Methods

```python
await ctx.info(message)            # Log info
await ctx.warning(message)         # Log warning
await ctx.error(message)           # Log error
await ctx.debug(message)           # Log debug
await ctx.report_progress(n, total)# Report progress
await ctx.read_resource(uri)       # Read a server resource
ctx.request_context                # Raw request context
ctx.request_context.meta           # Request metadata
```

### Tool Return Types

```python
# String (most common)
return "Simple text result"

# Dict (auto-serialized to JSON)
return {"key": "value", "count": 42}

# List
return ["item1", "item2"]

# Explicit TextContent
return [TextContent(type="text", text="Result")]

# Image
return [ImageContent(type="image", data="base64...", mimeType="image/png")]
```

### Transport Options

```python
mcp.run()                              # stdio (default)
mcp.run(transport="stdio")             # explicit stdio
mcp.run(transport="sse", port=8000)    # SSE/HTTP
mcp.run(transport="http", port=8000)   # Streamable HTTP
```

### Lifecycle Hooks (Low-Level)

```python
@server.list_tools()          # Return available tools
@server.call_tool()           # Handle tool invocation
@server.list_resources()      # Return available resources
@server.read_resource()       # Return resource content
@server.list_prompts()        # Return available prompts
@server.get_prompt()          # Return prompt content
@server.list_roots()          # Return root URIs
@server.complete()            # Handle autocomplete
```

---

### Complete File Structure for a Production Server

```
my-mcp-server/
├── src/
│   ├── __init__.py
│   ├── server.py          # FastMCP setup, entry point
│   ├── tools/
│   │   ├── __init__.py
│   │   ├── search.py      # Search-related tools
│   │   ├── data.py        # Data manipulation tools
│   │   └── notify.py      # Notification tools
│   ├── resources/
│   │   ├── __init__.py
│   │   └── configs.py     # Resource definitions
│   ├── prompts/
│   │   └── templates.py   # Prompt templates
│   └── utils/
│       ├── auth.py        # Authentication helpers
│       ├── cache.py       # Caching layer
│       └── db.py          # Database connection
├── tests/
│   ├── test_tools.py
│   ├── test_resources.py
│   └── test_integration.py
├── pyproject.toml
├── .env.example
└── README.md
```

```python
# server.py — Main entry point
from mcp.server.fastmcp import FastMCP
from .tools.search import register_search_tools
from .tools.data import register_data_tools
from .resources.configs import register_resources
from .prompts.templates import register_prompts

mcp = FastMCP("Production Server")

# Register all tools/resources/prompts
register_search_tools(mcp)
register_data_tools(mcp)
register_resources(mcp)
register_prompts(mcp)

def main():
    mcp.run()

if __name__ == "__main__":
    main()
```

```python
# tools/search.py — Tool registration module
def register_search_tools(mcp):
    
    @mcp.tool()
    async def search(query: str, max: int = 10) -> list:
        """Search the knowledge base."""
        return await _do_search(query, max)
    
    @mcp.tool()
    async def search_by_tag(tag: str) -> list:
        """Search items by tag."""
        return await _search_tags(tag)
```

---

*This guide covers MCP from zero to production. The protocol is actively developed — check [modelcontextprotocol.io](https://modelcontextprotocol.io) and [github.com/modelcontextprotocol](https://github.com/modelcontextprotocol) for the latest spec updates, new transports, and community servers.*

---

**Key Resources:**
- Official Docs: https://modelcontextprotocol.io
- Python SDK: https://github.com/modelcontextprotocol/python-sdk
- Node.js SDK: https://github.com/modelcontextprotocol/typescript-sdk
- MCP Inspector: `npx @modelcontextprotocol/inspector`
- Community Servers: https://github.com/modelcontextprotocol/servers
