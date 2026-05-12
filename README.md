# Complete MCP (Model Context Protocol) Hands-On Mastery Guide

## Beginner → Advanced | Theory + Hands-On + Architectures + Workflows

---

# Table of Contents

1. Introduction to MCP
2. Why MCP Exists
3. MCP vs REST APIs
4. MCP Architecture
5. Hosts, Clients, Servers
6. MCP Workflow
7. Transport Layers
8. Installing MCP
9. Running Existing MCP Servers
10. FastMCP Deep Dive
11. Creating Your First MCP Server
12. Decorators in MCP
13. Functions, Parameters & Type Hints
14. Resources
15. Prompts
16. Structured Outputs
17. Authentication & Security
18. Error Handling
19. Logging & Observability
20. MCP + FastAPI
21. MCP + LangChain
22. MCP + LangGraph
23. MCP + RAG
24. Healthcare MCP Project
25. Advanced MCP Concepts
26. Rarely Used MCP Features
27. Deployment
28. Production Best Practices
29. Common Errors
30. Complete Learning Roadmap

---

# 1. Introduction to MCP

MCP stands for:

> Model Context Protocol

MCP is a standardized protocol that allows AI applications to communicate with external systems like:

- APIs
- Databases
- Files
- GitHub
- Slack
- Cloud services
- Vector databases
- Internal enterprise tools

Think of MCP as:

```text
USB-C for AI applications
```

Just like USB-C standardizes hardware connectivity, MCP standardizes AI connectivity.

---

# 2. Why MCP Exists

Before MCP:

```text
Every AI app needed custom integrations.
```

Problems:

- Repeated development
- No standard tool format
- Different auth mechanisms
- Different schemas
- Hard maintenance

MCP solves this through:

- Standardized communication
- Reusable tool interfaces
- Dynamic tool discovery
- Reusable prompts/resources
- Better AI interoperability

---

# 3. MCP vs REST APIs

## REST API

REST is:

```text
Frontend ↔ Backend communication
```

Example:

```http
GET /users
POST /orders
```

The frontend explicitly decides:

- Which endpoint to call
- What payload to send

---

## MCP

MCP is:

```text
AI ↔ Tool communication
```

The AI dynamically decides:

- Which tool to call
- What parameters to use
- When to use tools

---

# 4. MCP Architecture

```text
User
 ↓
AI Application / Host
 ↓
MCP Client
 ↓
MCP Server
 ↓
External Systems
```

---

# 5. Hosts, Clients, Servers

## MCP Host

The application users interact with.

Examples:

- Claude Desktop
- Cursor
- VS Code
- AI IDEs
- Custom LangGraph systems

---

## MCP Client

Responsible for communication between:

```text
Host ↔ MCP Server
```

Responsibilities:

- Tool discovery
- Transport communication
- Schema validation

---

## MCP Server

The actual integration layer.

Exposes:

- Tools
- Resources
- Prompts

---

# 6. MCP Workflow

```text
User asks question
      ↓
AI analyzes available tools
      ↓
AI selects appropriate tool
      ↓
MCP client invokes tool
      ↓
MCP server executes tool
      ↓
Response returned
      ↓
AI generates final answer
```

---

# 7. Transport Layers

Transport = communication mechanism.

## stdio

Most common for local development.

Uses:

- stdin
- stdout

Good for:

- Local MCP servers
- Learning MCP
- IDE integrations

---

## HTTP Transport

Used for remote servers.

Useful for:

- Cloud deployments
- Enterprise systems
- Distributed architectures

---

## SSE (Server Sent Events)

Useful for:

- Streaming
- Progress updates
- Long-running tasks

---

# 8. Installing MCP

```bash
pip install mcp
```

Verify:

```bash
pip show mcp
```

---

# 9. Running Existing MCP Servers

Best way to learn MCP.

## Filesystem MCP Server

Capabilities:

- Read files
- Search files
- Analyze repositories

Use cases:

- README generation
- Code understanding
- Architecture explanation

---

## GitHub MCP Server

Capabilities:

- Analyze repositories
- Read issues
- Create PRs

---

## Database MCP Servers

Examples:

- PostgreSQL MCP
- SQLite MCP
- MongoDB MCP

Use cases:

- SQL analysis
- Data retrieval
- Analytics

---

# 10. FastMCP Deep Dive

Import:

```python
from mcp.server.fastmcp import FastMCP
```

Create server:

```python
mcp = FastMCP("my-server")
```

FastMCP simplifies:

- Tool registration
- Schema creation
- Validation
- MCP communication

---

# 11. Creating Your First MCP Server

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("calculator-server")


@mcp.tool()
def add(a: int, b: int) -> int:
    \"\"\"Add two numbers.\"\"\"
    return a + b


if __name__ == "__main__":
    mcp.run()
```

---

# 12. Decorators in MCP

Decorators are critical in MCP.

## What is a Decorator?

A decorator modifies function behavior.

---

## Without Decorator

```python
def greet():
    return "Hello"
```

---

## With Decorator

```python
@mcp.tool()
def greet():
    return "Hello"
```

Now the function becomes:

```text
An MCP Tool
```

---

# 13. @mcp.tool() Deep Dive

Most important MCP decorator.

Purpose:

- Registers function as MCP tool
- Generates schemas
- Enables AI discovery

---

## Example

```python
@mcp.tool()
def get_weather(city: str):
    \"\"\"Get weather information.\"\"\"
    return f"Weather for {city}"
```

---

## Internally MCP Does

- Tool registration
- Parameter schema generation
- Type validation
- Tool metadata creation

---

# 14. Functions & Parameters

## Basic Parameters

```python
@mcp.tool()
def greet(name: str):
    return f"Hello {name}"
```

---

## Multiple Parameters

```python
@mcp.tool()
def calculate(a: int, b: int, operation: str):
    pass
```

---

## Optional Parameters

```python
from typing import Optional

@mcp.tool()
def search(query: str, limit: Optional[int] = 5):
    pass
```

---

# 15. Type Hinting

Type hints help AI understand:

- Input types
- Output types
- Validation rules

---

## Example

```python
def add(a: int, b: int) -> int:
```

This means:

- a → integer
- b → integer
- returns integer

---

# 16. Resources

Resources provide readable context.

## Example

```python
@mcp.resource("notes://architecture")
def get_architecture():
    return "Architecture notes"
```

Use cases:

- Documentation
- Policies
- Knowledge bases

---

# 17. Prompts

Reusable prompt templates.

## Example

```python
@mcp.prompt()
def summarize_prompt():
    return "Summarize the provided content."
```

---

# 18. Structured Outputs

Always prefer structured outputs.

Good:

```python
return {
    "covered": True,
    "copay": 1500
}
```

Bad:

```python
return "Covered with copay"
```

---

# 19. Authentication

Production systems require authentication.

Methods:

- API Keys
- JWT
- OAuth

Example:

```python
import os

API_KEY = os.getenv("API_KEY")
```

---

# 20. Security Best Practices

Never execute raw commands.

Bad:

```python
os.system(command)
```

Good:

- Whitelisted operations
- Input validation
- Least privilege access

---

# 21. Error Handling

```python
@mcp.tool()
def divide(a: int, b: int):

    if b == 0:
        return {"error": "Division by zero"}

    return a / b
```

---

# 22. Logging

```python
import logging

logging.basicConfig(level=logging.INFO)
```

Production logging is mandatory.

---

# 23. MCP + FastAPI

Architecture:

```text
Frontend
   ↓
FastAPI
   ↓
MCP Tools
   ↓
External Systems
```

Use MCP inside FastAPI endpoints.

---

# 24. MCP + LangChain

Architecture:

```text
LangChain Agent
      ↓
MCP Client
      ↓
MCP Server
```

Benefits:

- Dynamic tool calling
- Reusable integrations

---

# 25. MCP + LangGraph

Perfect for Agentic AI.

Architecture:

```text
LangGraph Node
      ↓
MCP Tool
      ↓
Response
      ↓
Next Node
```

---

# 26. MCP + RAG

Workflow:

```text
User Query
   ↓
MCP Tool
   ↓
Vector Search
   ↓
Relevant Chunks
   ↓
LLM Response
```

---

# 27. Healthcare MCP Project

## Use Case

Healthcare Policy & Copay Assistant

Features:

- Coverage checking
- Copay estimation
- Policy RAG search
- Eligibility validation

---

## Architecture

```text
Frontend
   ↓
FastAPI
   ↓
LangGraph
   ↓
Healthcare MCP Server
   ├── Coverage Tool
   ├── Copay Tool
   ├── RAG Tool
   └── Explanation Tool
```

---

# 28. Advanced MCP Concepts

## Sampling

Allows model-generated content during workflows.

---

## Roots

Restricts accessible resource boundaries.

---

## Elicitation

Server asks user for missing information.

Example:

```text
Please provide policy ID.
```

---

## Progress Notifications

Useful for:

- PDF ingestion
- Vector indexing
- Long-running tasks

---

# 29. Rarely Used Features

- Dynamic tool registration
- Capability negotiation
- Resource subscriptions
- Stateful sessions

---

# 30. Deployment

## Local

```bash
python server.py
```

---

## Docker

```dockerfile
FROM python:3.11

WORKDIR /app

COPY . .

RUN pip install -r requirements.txt

CMD ["python", "server.py"]
```

---

# 31. Best Practices

- Use type hints
- Add docstrings
- Return structured outputs
- Add validation
- Keep tools small
- Add logging
- Handle errors gracefully

---

# 32. Common Errors

## Tool Not Discovered

Cause:

```text
Missing @mcp.tool()
```

---

## Validation Errors

Cause:

```text
Wrong parameter types
```

---

## Import Errors

Fix:

```bash
pip install mcp
```

---

# 33. Recommended Learning Path

## Beginner

Learn:

- MCP basics
- Decorators
- Tools
- stdio transport

Projects:

- Calculator MCP
- Weather MCP

---

## Intermediate

Learn:

- Resources
- Prompts
- Databases
- Authentication

Projects:

- GitHub Assistant
- Resume Analyzer

---

## Advanced

Learn:

- LangGraph integration
- RAG
- Streaming
- Multi-server systems

Projects:

- Healthcare AI Platform
- Enterprise MCP System

---

# 34. Recommended Stack

Frontend:

- React + Vite

Backend:

- FastAPI

AI:

- LangChain
- LangGraph
- Groq

Embeddings:

- sentence-transformers/all-MiniLM-L6-v2

Database:

- MongoDB Atlas Vector Search

Protocol:

- MCP

---

# 35. Final Advice

Best way to master MCP:

1. Use existing MCP servers
2. Build small tools
3. Add APIs
4. Add databases
5. Add LangGraph
6. Add RAG
7. Deploy production systems
