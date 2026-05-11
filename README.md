# Notebook on MCP server

**ChatGPT Apps SDK Tutorial:** Getting Started with Your First MCP Server using Python, FastMCP, and FastAPI

- [Watch on YouTube](https://www.youtube.com/watch?v=YAxA-7ZSn-0)

**Integration with FastAPI**: The tutorial explains how to combine a traditional FastAPI REST application with a FastMCP server. This allows developers to serve both a web interface and an MCP connector simultaneously.

**Shared lifespan**: An advanced section on managing the 'lifespan' of both applications to allow for efficient startup/shutdown processes, which is essential if your app interacts with databases or machine learning models.

```python
from contextlib import asynccontextmanager
api = FastAPI()
mcp = FastMCP.from_fastapi(api)
mcp_app = mcp.http_app(path='/mcp')
```

```python
def answer_to_ml_model_everything(x: float):
    return x * 42


ml_models = {}
```

### Step 1

```python
**Create an individual lifespans:**
@asynccontextmanager
async def fastapi_lifespan(app: FastAPI):
    # run something before the app fully starts
    ml_models["ml_model"] = answer_to_ml_model_everything
    yield
    # run something after the app ends
```

```python
# tool -> mcp
@mcp.tool()
def add(a:int, b:int) -> int:
    """
    Add two numbers
    """
    return (a + b) * 3823
```

### Step 2

**Define a master async context manager:**

```python
@asynccontextmanager
async def global_lifespan(app:FastAPI):
    async with fastapi_lifespan(app):
        async with mcp_app.lifespan(app):
            yield
```

### Step 3

**Pass the global life span in step 2 to the `lifespan` parameter of the FastAPI app**

```python
app = FastAPI(
    title="Hungry Py MCP App",
    routes = [
        *mcp_app.routes,
        *api.routes,
    ],
    lifespan=global_lifespan
)

```

**Steps to merge lifespans:**

1. **Define individual lifespans**: Create separate async context manager functions for both your FastAPI app and your FastMCP server
2. **Create a global lifespan manager**: Define a master async context manager (often called a `global_lifespan` or `combined_lifespan`). Inside this manager, use `async with` blocks to nest the individual lifespans together, then `yield` to allow the application to start
3. **Apply to the app**: Pass this combined lifespan function into the `lifespan` parameter of your final FastAPI application instantiation ß

[Full code example](https://github.com/codingforentrepreneurs/chatgpt-apps-sdk-hello-world)

**Key Concept:**

**Nested Execution**: By placing the FastAPI lifespan at the highest level and the FastMCP lifespan nested within it, you ensure that the startup logic for one triggers the other in sequence, creating a reliable shared environment for your tools and routes

# MCP Elicitation

Elicitation enables MCP servers to request specific information directly from users during tool execution, enhancing interactivity and improving the overall user experience.

- [MCP Elicitation Tutorial — Building Interactive Tools with FastMCP](https://www.youtube.com/watch?v=BYDr3jIuybU)
- [Example](https://github.com/zazencodes/zazencodes-season-2/tree/main/src/mcp-elicitation/book-buy-demo-mcp)

**User Flow**

1. **Server Initiation**: During the runtime of a tool call, the server identifies a need for additional input and initiates an elicitation.create request. This pauses the tool's execution while the server waits for the requested information.

2. **Human Interaction Phase**: The MCP client (e.g., VS Code or the Model Context Protocol Inspector) receives the request and presents it to the user. The user provides the necessary input or confirmation through the client interface.

3. **Completion**: The client packages the user's response—which includes an action field (e.g., "accept") and the provided content—and sends it back to the server. The server then resumes processing using the new data.

**Key Implementation Details:**

- **Async Execution**: In FastMCP, this is implemented as an awaitable function, meaning the script holds execution on the elicitation line until the response is received.
- **Schema Handling**: Servers can request specific data formats, such as strings or literals, to ensure the input matches the expected requirements.
- **Error Prevention**: For complex workflows, it is crucial to manage state so tools aren't triggered prematurely.

- [FastMCP Elicitation Documentation](https://gofastmcp.com/clients/elicitation)

# Building apps

## FastMCP 3.0 features

### Concepts

1. **Generative UI** means the LLM writes the UI code at runtime. Instead of calling a pre-built tool with a fixed interface, the model writes Prefab Python code tailored to the current data and request. The user watches the UI build up in real time as the model generates code. [Learn more](https://gofastmcp.com/apps/generative)

2. Prefab UI is the component library behind all FastMCP app features. You describe layouts, charts, tables, and forms in Python, and Prefab compiles them to interactive UIs that render in the host's conversation. [Learn more](https://gofastmcp.com/apps/prefab)

3. **Prefab app** can call server tools — there's nothing stopping you from using `CallTool("tool_name")` in a regular `@mcp.tool(app=True)` Managed tool binding, visibility, and composition for apps with heavy server interaction. [Learn more](https://gofastmcp.com/apps/interactive-apps)

4. **OAuth Authentication:** Authenticate your FastMCP client via OAuth 2.1 [Learn more](https://gofastmcp.com/clients/auth/oauth)

5. **Bearer Token Authentication:** Authenticate your FastMCP client with a Bearer token.[Learn more](https://gofastmcp.com/clients/auth/bearer)

# Installing FastMCP

fastmcp.json configuration files declare dependencies alongside the server definition. When you install from a config file `fastmcp.json`, dependencies are picked up automatically:

```bash
fastmcp install claude-desktop fastmcp.json
fastmcp install claude-desktop  # auto-detects fastmcp.json in current directory
```

**Why do you need it?**

`fastmcp install` performs two essential tasks:

1. **Registers the server** with an MCP client application, enabling the client to launch it automatically

2. **Manages dependencies** in an isolated environment — each MCP client runs servers separately, so all dependencies must be explicitly declared rather than relying on locally installed packages [Learn more](https://gofastmcp.com/cli/install-mcp)

## Common Patterns

**Basic install with auto-detected server instance**

```bash
fastmcp install claude-desktop server.py
```

**Install with environment file**

```bash
fastmcp install cursor server.py --env-file .env
```

## Generating MCP JSON

The `mcp-json` target generates standard MCP configuration JSON instead of installing into a specific client. This is useful for clients that FastMCP doesn't directly support, for CI/CD environments, or for sharing server configs:

```bash
fastmcp install mcp-json server.py
```

Example output:

```json
{
  "server-name": {
    "command": "uv",
    "args": [
      "run",
      "--with",
      "fastmcp",
      "fastmcp",
      "run",
      "/path/to/server.py"
    ],
    "env": {
      "API_KEY": "value"
    }
  }
}
```

Use `--copy` to send it to your clipboard instead of stdout.

### Generating Stdio Commands (CLI)

**Basic stdio command:**

```bash
fastmcp install stdio server.py
# Output: uv run --with fastmcp fastmcp run /absolute/path/to/server.py
```

**With fastmcp.json pattern:**

```bash
fastmcp install stdio fastmcp.json
# Output: uv run --with fastmcp --with pillow --with 'qrcode[pil]>=8.0' fastmcp run /path/to/server.py
```

# Migration Guide to V3

Migration instructions for upgrading between FastMCP versions

[This guide covers breaking changes and migration steps when upgrading FastMCP](https://gofastmcp.com/getting-started/upgrading/from-fastmcp-2) most importantly deprecated features and kwargs
