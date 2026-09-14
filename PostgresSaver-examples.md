# LangGraph PostgreSQL Checkpoint Persistence

## Session History

This document contains the conversation about using LangGraph checkpoint persistence with PostgreSQL, including `PersistentDict`, `PostgresSaver`, `AsyncPostgresSaver`, and `PostgresStore`.

---

## User

Could you provide examples for langgraph's checkpoint.PersistentDict for Postgres DB?

---

## Assistant

Yes. One important distinction first: **`PersistentDict` is not the Postgres persistence mechanism in LangGraph**. It lives in `langgraph.checkpoint.memory` and is essentially a disk-backed dictionary used by the memory checkpointer implementation.

For production PostgreSQL persistence, LangGraph provides:

- `PostgresSaver`
- `AsyncPostgresSaver`

So if your goal is something like:

```text
LangGraph agent
      ↓
checkpoint state
      ↓
PostgreSQL
      ↓
resume the same thread later
```

then `PostgresSaver` is what you want.

### 1. Install

```bash
pip install langgraph langgraph-checkpoint-postgres "psycopg[binary]"
```

The Postgres checkpoint package uses Psycopg 3.

### 2. Minimal `PostgresSaver` example

Suppose your database is:

```text
postgresql://myuser:mypassword@localhost:5432/langgraph
```

You can create the checkpointer like this:

```python
from langgraph.checkpoint.postgres import PostgresSaver

DB_URI = "postgresql://myuser:mypassword@localhost:5432/langgraph"

with PostgresSaver.from_conn_string(DB_URI) as checkpointer:
    # Run once when initializing the database.
    checkpointer.setup()

    # use checkpointer with your graph
```

`setup()` creates/migrates the tables LangGraph needs.

LangGraph recommends running it during deployment/database initialization rather than on every application startup.

---

## 3. Complete LangGraph example

Suppose we have a small graph whose state survives application restarts.

```python
from typing import TypedDict

from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.postgres import PostgresSaver


class State(TypedDict):
    counter: int


def increment(state: State):
    return {
        "counter": state["counter"] + 1
    }


builder = StateGraph(State)

builder.add_node("increment", increment)

builder.add_edge(START, "increment")
builder.add_edge("increment", END)


DB_URI = (
    "postgresql://myuser:mypassword"
    "@localhost:5432/langgraph"
)


with PostgresSaver.from_conn_string(DB_URI) as checkpointer:

    # Only necessary the first time / during migrations.
    checkpointer.setup()

    graph = builder.compile(
        checkpointer=checkpointer
    )

    config = {
        "configurable": {
            "thread_id": "user-123"
        }
    }

    result = graph.invoke(
        {"counter": 0},
        config=config,
    )

    print(result)
```

Result:

```text
{'counter': 1}
```

The important part is:

```python
config = {
    "configurable": {
        "thread_id": "user-123"
    }
}
```

The `thread_id` identifies the persistent execution/conversation.

Checkpoints belonging to that thread are stored in Postgres.

---

## 4. Resume the same thread

Later—even after restarting your Python process—you can use the same `thread_id`.

```python
with PostgresSaver.from_conn_string(DB_URI) as checkpointer:

    graph = builder.compile(
        checkpointer=checkpointer
    )

    config = {
        "configurable": {
            "thread_id": "user-123"
        }
    }

    state = graph.get_state(config)

    print(state.values)
```

For example:

```text
{'counter': 1}
```

You can then continue execution using that same thread.

This is the major difference from:

```python
from langgraph.checkpoint.memory import InMemorySaver

checkpointer = InMemorySaver()
```

`InMemorySaver` loses its checkpoints when the process exits, whereas `PostgresSaver` persists them.

---

## 5. Chat-agent example

This is probably closer to the typical real-world use case.

```python
import os

from langchain.chat_models import init_chat_model
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import MessagesState
from langgraph.checkpoint.postgres import PostgresSaver


model = init_chat_model("openai:gpt-5")


def chatbot(state: MessagesState):
    response = model.invoke(state["messages"])

    return {
        "messages": [response]
    }


builder = StateGraph(MessagesState)

builder.add_node("chatbot", chatbot)

builder.add_edge(START, "chatbot")
builder.add_edge("chatbot", END)


DB_URI = os.environ["DATABASE_URL"]


with PostgresSaver.from_conn_string(DB_URI) as checkpointer:

    graph = builder.compile(
        checkpointer=checkpointer
    )

    config = {
        "configurable": {
            "thread_id": "conversation-123"
        }
    }

    # First message
    result = graph.invoke(
        {
            "messages": [
                {
                    "role": "user",
                    "content": "My name is Austin.",
                }
            ]
        },
        config,
    )

    # Second message
    result = graph.invoke(
        {
            "messages": [
                {
                    "role": "user",
                    "content": "What is my name?",
                }
            ]
        },
        config,
    )

    print(result["messages"][-1].content)
```

Because both invocations use:

```python
thread_id = "conversation-123"
```

LangGraph retrieves the previous checkpoint from PostgreSQL and the model sees the previous messages.

---

## 6. Using an existing Psycopg connection

If your application already manages PostgreSQL connections, you don't have to use `from_conn_string()`.

LangGraph specifically requires `autocommit=True` and `row_factory=dict_row` when manually supplying the connection.

```python
import psycopg

from psycopg.rows import dict_row
from langgraph.checkpoint.postgres import PostgresSaver


conn = psycopg.connect(
    DB_URI,
    autocommit=True,
    row_factory=dict_row,
)

checkpointer = PostgresSaver(conn)

graph = builder.compile(
    checkpointer=checkpointer
)
```

In particular, avoid:

```python
conn = psycopg.connect(DB_URI)
```

because the default row factory produces tuples, while `PostgresSaver` expects dictionary-style row access.

---

## 7. Async version

For an async Python service—FastAPI, for example—I'd generally use `AsyncPostgresSaver`.

```python
from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver


DB_URI = "postgresql://myuser:mypassword@localhost:5432/langgraph"


async def run_graph():

    async with AsyncPostgresSaver.from_conn_string(
        DB_URI
    ) as checkpointer:

        graph = builder.compile(
            checkpointer=checkpointer
        )

        config = {
            "configurable": {
                "thread_id": "conversation-123"
            }
        }

        result = await graph.ainvoke(
            {
                "messages": [
                    {
                        "role": "user",
                        "content": "Hello!",
                    }
                ]
            },
            config,
        )

        return result
```

LangGraph provides the async implementation directly.

---

## 8. Connection pool example

For a production web service, using a pool is usually preferable to creating one DB connection per request.

```python
from psycopg_pool import ConnectionPool
from langgraph.checkpoint.postgres import PostgresSaver


DB_URI = "postgresql://myuser:mypassword@localhost:5432/langgraph"


pool = ConnectionPool(
    conninfo=DB_URI,
    min_size=2,
    max_size=10,
    kwargs={
        "autocommit": True,
    },
)


checkpointer = PostgresSaver(pool)

graph = builder.compile(
    checkpointer=checkpointer
)
```

LangGraph's implementation supports `ConnectionPool` directly.

For an async FastAPI application I'd instead use:

```text
AsyncConnectionPool
+
AsyncPostgresSaver
```

---

## 9. Inspecting checkpoints directly

You can retrieve the latest checkpoint:

```python
config = {
    "configurable": {
        "thread_id": "conversation-123"
    }
}

checkpoint = checkpointer.get(config)

print(checkpoint)
```

Or list checkpoint history:

```python
checkpoints = list(
    checkpointer.list(config)
)

for checkpoint in checkpoints:
    print(checkpoint)
```

Checkpoint history is one advantage of the regular `PostgresSaver`.

LangGraph also provides `ShallowPostgresSaver`, which keeps only the most recent checkpoint and therefore doesn't support time-travel/history in the same way.

---

## 10. `PersistentDict` vs `PostgresSaver` vs `PostgresStore`

There's another distinction that's easy to miss:

| Component | Purpose |
|---|---|
| `PersistentDict` | Local/disk-backed dictionary implementation |
| `InMemorySaver` | Development/testing checkpoint storage |
| `PostgresSaver` | Persistent **graph/thread checkpoints** |
| `PostgresStore` | Persistent **application/agent memory across threads** |

For example, `PostgresStore` is useful for something like:

```text
user-123
│
├── conversation-A
├── conversation-B
│
└── persistent preferences
    ├── theme = dark
    └── language = English
```

while `PostgresSaver` handles the state/history of conversation A or B.

LangGraph's Postgres store API uses operations such as `put()` and `get()` for durable cross-thread data.

A production architecture could therefore look like:

```text
                    PostgreSQL
                        │
             ┌──────────┴──────────┐
             │                     │
      PostgresSaver          PostgresStore
             │                     │
       thread state           long-term memory
             │                     │
    conversation-123          user-123
      checkpoints            preferences/data
```

If what you specifically need is **a `PersistentDict`-style key/value API backed by Postgres**, rather than LangGraph checkpointing, that's a different design; `PostgresStore` may actually be the closer built-in abstraction.

---

## User

Can you provide this chat history as md file format? So that I can download or pase to local md file?

---

## Assistant

Yes. Below is a Markdown version you can copy directly into a local `.md` file.

### LangGraph PostgreSQL Checkpoint Persistence

One important distinction:

**`PersistentDict` is not the PostgreSQL persistence mechanism in LangGraph.**

`PersistentDict` is essentially a disk-backed dictionary used by LangGraph's checkpoint implementation.

For production PostgreSQL persistence, LangGraph provides:

- `PostgresSaver`
- `AsyncPostgresSaver`

If the goal is:

```text
LangGraph Agent
      ↓
Checkpoint State
      ↓
PostgreSQL
      ↓
Resume same thread later
```

then `PostgresSaver` is the appropriate solution.

### Installation

```bash
pip install langgraph langgraph-checkpoint-postgres "psycopg[binary]"
```

### Minimal `PostgresSaver`

```python
from langgraph.checkpoint.postgres import PostgresSaver

DB_URI = "postgresql://myuser:mypassword@localhost:5432/langgraph"

with PostgresSaver.from_conn_string(DB_URI) as checkpointer:
    checkpointer.setup()
```

### Persistent thread

```python
config = {
    "configurable": {
        "thread_id": "conversation-123"
    }
}
```

The `thread_id` identifies a persistent LangGraph execution/conversation.

Checkpoints belonging to that thread are stored in PostgreSQL.

### Existing Psycopg connection

```python
import psycopg

from psycopg.rows import dict_row
from langgraph.checkpoint.postgres import PostgresSaver


conn = psycopg.connect(
    DB_URI,
    autocommit=True,
    row_factory=dict_row,
)

checkpointer = PostgresSaver(conn)
```

### Async

```python
from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver
```

For async applications such as FastAPI, `AsyncPostgresSaver` is generally preferable.

### Connection Pool

```python
from psycopg_pool import ConnectionPool
from langgraph.checkpoint.postgres import PostgresSaver


pool = ConnectionPool(
    conninfo=DB_URI,
    min_size=2,
    max_size=10,
    kwargs={
        "autocommit": True,
    },
)

checkpointer = PostgresSaver(pool)
```

### Component comparison

| Component | Purpose |
|---|---|
| `PersistentDict` | Local/disk-backed dictionary |
| `InMemorySaver` | Development/testing checkpoints |
| `PostgresSaver` | Persistent graph/thread checkpoints |
| `PostgresStore` | Persistent application/agent memory across threads |

A typical production architecture is:

```text
LangGraph
   │
   ├── PostgresSaver
   │      └── Conversation / thread checkpoints
   │
   └── PostgresStore
          └── Long-term agent / user memory
```

---

## User

Can you make all history of this session to md file format so that I can copy and paste to my local file?

---

## End of Session Export

Suggested filename:

```text
langgraph-postgres-checkpoint-session.md
```