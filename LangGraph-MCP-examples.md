# Gemini AI + MCP Server with LangGraph and LangSmith

**Date:** August 29, 2026

## User

Could you show me examples that provide a mcp server to gemini-ai using langsmith or langgraph?

---

## Assistant

Yes. The cleanest architecture is:

**Gemini model → LangGraph agent → MCP client adapter → MCP server**, with **LangSmith** providing tracing/observability around the LangGraph execution.

LangGraph itself is model-provider agnostic, so Gemini can be the reasoning model while MCP tools are exposed to the agent. LangSmith then traces the model and tool calls.

A practical Python example looks like this:

```text
User
  │
  ▼
LangGraph Agent
  │
  ├── Gemini 2.5/3.x
  │      │
  │      └── decides which tool to call
  │
  └── MCP Client
          │
          ▼
      MCP Server
       ├── tool A
       ├── tool B
       └── tool C

       │
       └────────────► LangSmith tracing
```

## 1. Install dependencies

```bash
pip install \
    langgraph \
    langchain \
    langchain-google-genai \
    langchain-mcp-adapters \
    langsmith
```

Set the keys:

```bash
export GOOGLE_API_KEY="your-google-api-key"

export LANGSMITH_API_KEY="your-langsmith-api-key"
export LANGSMITH_TRACING=true
export LANGSMITH_PROJECT="gemini-mcp-demo"
```

Those LangSmith environment variables are the standard tracing setup.

---

## 2. Example MCP server

Suppose you have an MCP server providing two simple tools.

```python
# mcp_server.py

from mcp.server.fastmcp import FastMCP

mcp = FastMCP("example-server")


@mcp.tool()
def get_weather(city: str) -> str:
    """Get weather information for a city."""
    return f"The weather in {city} is sunny and 72°F."


@mcp.tool()
def add_numbers(a: int, b: int) -> int:
    """Add two numbers."""
    return a + b


if __name__ == "__main__":
    mcp.run(transport="stdio")
```

Conceptually, Gemini doesn't directly execute this Python code. LangGraph exposes these MCP tools to Gemini as model tools.

---

## 3. Connect LangGraph to MCP and use Gemini

Here's the important part:

```python
# agent.py

import asyncio

from langchain_google_genai import ChatGoogleGenerativeAI
from langchain_mcp_adapters.client import MultiServerMCPClient
from langgraph.prebuilt import create_react_agent


async def main():

    # ---------------------------------------
    # 1. Connect to MCP server
    # ---------------------------------------

    client = MultiServerMCPClient(
        {
            "example": {
                "command": "python",
                "args": ["mcp_server.py"],
                "transport": "stdio",
            }
        }
    )

    tools = await client.get_tools()

    print("Available MCP tools:")

    for tool in tools:
        print(f"  - {tool.name}")

    # ---------------------------------------
    # 2. Create Gemini model
    # ---------------------------------------

    model = ChatGoogleGenerativeAI(
        model="gemini-2.5-flash",
        temperature=0,
    )

    # ---------------------------------------
    # 3. Create LangGraph agent
    # ---------------------------------------

    agent = create_react_agent(
        model=model,
        tools=tools,
    )

    # ---------------------------------------
    # 4. Run agent
    # ---------------------------------------

    result = await agent.ainvoke(
        {
            "messages": [
                {
                    "role": "user",
                    "content": (
                        "What's the weather in Seattle? "
                        "Also calculate 123 + 456."
                    ),
                }
            ]
        }
    )

    print(result["messages"][-1].content)


if __name__ == "__main__":
    asyncio.run(main())
```

The important relationship is:

```python
tools = await client.get_tools()

agent = create_react_agent(
    model=gemini,
    tools=tools,
)
```

The MCP server becomes the **tool provider**, while Gemini remains the **LLM responsible for deciding when and how to invoke those tools**.

---

## 4. Using a remote HTTP MCP server

For production, you probably won't launch an MCP server with `stdio`. Instead, it may be something like:

```text
https://my-company.example.com/mcp
```

Then your configuration can look conceptually like:

```python
client = MultiServerMCPClient(
    {
        "internal-api": {
            "url": "https://my-company.example.com/mcp",
            "transport": "streamable_http",
        }
    }
)

tools = await client.get_tools()
```

And the Gemini/LangGraph portion stays essentially the same:

```python
model = ChatGoogleGenerativeAI(
    model="gemini-2.5-flash",
)

agent = create_react_agent(
    model=model,
    tools=tools,
)
```

This separation is useful because you can swap:

```text
Gemini
  ↓
Claude
  ↓
OpenAI
```

without changing your MCP server.

Likewise, you can add several MCP servers:

```python
client = MultiServerMCPClient(
    {
        "database": {
            "url": "https://database.example.com/mcp",
            "transport": "streamable_http",
        },

        "github": {
            "url": "https://github.example.com/mcp",
            "transport": "streamable_http",
        },

        "internal-api": {
            "url": "https://api.example.com/mcp",
            "transport": "streamable_http",
        },
    }
)

tools = await client.get_tools()
```

Gemini then sees tools originating from all three MCP servers.

---

## 5. Where LangSmith fits

LangSmith isn't normally the component that connects Gemini to MCP.

Think of the responsibilities as:

| Component | Responsibility |
|---|---|
| Gemini | reasoning / tool selection |
| LangGraph | agent orchestration |
| MCP adapter | MCP client |
| MCP server | exposes tools/resources |
| LangSmith | tracing, debugging, evaluation, deployment |

LangSmith supports tracing Gemini applications. LangGraph applications can also be traced using the normal LangSmith tracing configuration.

You'd get a trace conceptually similar to:

```text
agent
│
├── Gemini
│   └── tool_call: get_weather
│
├── MCP
│   └── get_weather("Seattle")
│
├── Gemini
│   └── tool_call: add_numbers
│
├── MCP
│   └── add_numbers(123, 456)
│
└── Gemini
    └── final response
```

---

## 6. LangSmith/LangGraph deployment

You can also deploy the entire graph:

```text
                    LangSmith
                 Agent Deployment
                       │
                 ┌─────▼─────┐
                 │ LangGraph │
                 └─────┬─────┘
                       │
               ┌───────▼────────┐
               │ Gemini         │
               │ Google AI API  │
               └────────────────┘
                       │
              Gemini tool calling
                       │
                 ┌─────▼─────┐
                 │ MCP Client│
                 └─────┬─────┘
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
    DB MCP          GitHub MCP    Internal MCP
```

LangGraph deployments can themselves be called remotely through `RemoteGraph`, and LangSmith deployments provide the production API/runtime around the graph.

There's also a newer deployment pattern where MCP servers can be declared in an `mcp.json`.

For example:

```json
{
  "mcpServers": {
    "docs": {
      "url": "https://docs.langchain.com/mcp",
      "transport": "http"
    }
  }
}
```

For that deployment approach, HTTP/SSE MCP servers are supported while local `stdio` servers aren't suitable because the deployed environment can't spawn the local MCP process.

For this use case, I'd start with:

**Gemini + `langchain-mcp-adapters` + LangGraph + LangSmith tracing**

It gives explicit control over MCP connections and makes it easy to move the graph into LangSmith deployment later.

---

## User

Can you make all history of this session to md file format so that I can copy and paste to my local file?

---

# Summary Architecture

```text
                       ┌─────────────────┐
                       │      User       │
                       └────────┬────────┘
                                │
                                ▼
                       ┌─────────────────┐
                       │    LangGraph    │
                       │      Agent      │
                       └────────┬────────┘
                                │
                    ┌───────────┴───────────┐
                    │                       │
                    ▼                       ▼
           ┌─────────────────┐     ┌─────────────────┐
           │ Google Gemini   │     │   MCP Client    │
           │      LLM        │     │    Adapter      │
           └─────────────────┘     └────────┬────────┘
                                            │
                         ┌──────────────────┼──────────────────┐
                         │                  │                  │
                         ▼                  ▼                  ▼
                    ┌─────────┐        ┌─────────┐       ┌─────────┐
                    │ DB MCP  │        │GitHub   │       │Internal │
                    │ Server  │        │MCP      │       │API MCP  │
                    └─────────┘        └─────────┘       └─────────┘

                              LangSmith
                    ──────────────────────────
                    Tracing / Evaluation /
                    Debugging / Deployment
```

## Recommended Stack

- **LLM:** Google Gemini
- **Agent orchestration:** LangGraph
- **MCP integration:** `langchain-mcp-adapters`
- **MCP protocol:** Streamable HTTP for production
- **Local development:** MCP `stdio`
- **Observability:** LangSmith
- **Deployment:** LangGraph/LangSmith deployment