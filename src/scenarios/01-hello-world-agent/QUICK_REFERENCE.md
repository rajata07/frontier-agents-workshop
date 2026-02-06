# Scenario 1 - Quick Reference Card

## 🚀 Quick Start

### 1. Start MCP Servers
```bash
# Terminal 1 - Weather Server (Port 8001)
cd src/mcp-server/04-weather-server
python server-mcp-sse-weather.py

# Terminal 2 - User Server (Port 8002)
cd src/mcp-server/02-user-server
python server-mcp-sse-user.py
```

### 2. Run Sample Agent
```bash
cd samples/simple-agents
python agents-using-mcp.py
```

### 3. Start Dev UI (Optional)
```bash
agent-framework-devui
# Open http://localhost:5000
```

## 📋 Three Solution Approaches

| Approach | File | Key Features | Use Case |
|----------|------|--------------|----------|
| **Solution 1: Basic** | `basic-agent.py` | Simple function calling | Single-turn queries |
| **Solution 2: Thread** | `agent-thread.py` | Conversation memory | Multi-turn dialogs |
| **Solution 3: MCP** | `agents-using-mcp.py` | External services | Scalable architecture |

## 🛠️ MCP Servers

### Weather Server (Port 8001)
**Endpoint**: `http://localhost:8001/mcp`

**Tools**:
- `list_supported_locations()` → List of cities
- `get_weather_at_location(location: str)` → Weather description
- `get_weather_for_multiple_locations(locations: list)` → Multiple results

**Supported Cities**: Seattle, New York, London, Berlin, Tokyo, Sydney

### User Server (Port 8002)
**Endpoint**: `http://localhost:8002/mcp`

**Tools**:
- `get_current_user()` → Username
- `get_current_location(username: str)` → Timezone
- `get_current_time(location: str)` → Current time
- `move(username: str, newlocation: str)` → Boolean

## 💻 Code Templates

### Basic Agent Setup
```python
from agent_framework import ChatAgent
from samples.shared.model_client import create_chat_client

client = create_chat_client("gpt-4")
agent = ChatAgent(
    chat_client=client,
    instructions="You are a helpful assistant.",
    tools=my_function,  # Python function as tool
)
result = await agent.run("User query here")
```

### Agent with Thread
```python
agent = ChatAgent(chat_client=client, tools=my_function)
thread = agent.get_new_thread()  # Create thread

# Multiple queries with memory
result1 = await agent.run(query1, thread=thread)
result2 = await agent.run(query2, thread=thread)  # Remembers query1
```

### Agent with MCP
```python
from agent_framework import MCPStreamableHTTPTool

weather_tool = MCPStreamableHTTPTool(
    name="Weather Server",
    url="http://localhost:8001/mcp"
)

async with ChatAgent(
    chat_client=client,
    tools=weather_tool,  # MCP server as tool
) as agent:
    result = await agent.run(query)
```

### Multiple MCP Servers
```python
tools = [
    MCPStreamableHTTPTool(name="User", url="http://localhost:8002/mcp"),
    MCPStreamableHTTPTool(name="Weather", url="http://localhost:8001/mcp"),
]

agent = ChatAgent(chat_client=client, tools=tools)
```

## 🧪 Test Queries

```python
queries = [
    "I am currently in London",
    "What is the weather now here?",
    "What time is it for me right now?",
    "I moved to Berlin, what is the weather like today?",
    "Can you remind me where I said I am based?",
    "Compare weather in Tokyo and Sydney",
]
```

## 🐛 Common Issues & Fixes

| Problem | Solution |
|---------|----------|
| Agent doesn't call tools | Check `tools` parameter is set |
| Agent forgets context | Use same `thread` object for all calls |
| MCP connection fails | Verify servers running on ports 8001, 8002 |
| Wrong timezone | Use IANA format: `America/New_York` |
| Model auth error | Check `.env` has `GITHUB_TOKEN` or Azure keys |

## 📊 Decision Tree

```
Need to build an agent?
│
├─ Single query, no memory needed?
│  └─ Use Solution 1: Basic Agent
│
├─ Multi-turn conversation?
│  └─ Use Solution 2: Agent with Thread
│
└─ Need external services?
   └─ Use Solution 3: Agent with MCP
```

## 🔍 Debugging Checklist

- [ ] MCP servers running? → `curl http://localhost:8001/mcp`
- [ ] Environment variables set? → Check `.env` file
- [ ] Thread being reused? → Same thread object in all calls
- [ ] Tools registered? → Check `tools=` parameter
- [ ] Instructions clear? → Add explicit tool usage guidance
- [ ] Dev UI accessible? → http://localhost:5000

## 📚 Key Concepts

### Function as Tool
```python
def my_tool(param: Annotated[str, Field(description="...")]) -> str:
    """Tool description for the model."""
    return "result"
```
**Requirements**: Type hints, docstring, return type

### Thread Management
- **Create**: `thread = agent.get_new_thread()`
- **Use**: `await agent.run(query, thread=thread)`
- **Persist**: Pass same thread object to all calls

### MCP Integration
- **Protocol**: Standardized tool exposure
- **Connection**: HTTP endpoint (`/mcp`)
- **Discovery**: Automatic tool detection
- **Execution**: Remote function calls

## 🎯 Learning Outcomes

After completing Scenario 1, you should be able to:
- ✅ Create ChatAgent instances
- ✅ Register tools (functions and MCP)
- ✅ Manage conversation threads
- ✅ Connect to MCP servers
- ✅ Debug with Dev UI
- ✅ Handle multi-turn conversations

## 📖 Full Documentation

For detailed explanations, architecture diagrams, and comprehensive examples, see:
- **[SOLUTION_GUIDE.md](./SOLUTION_GUIDE.md)** - Complete solution guide with all details
- **[README.md](./README.md)** - Scenario overview and tasks

## 🔗 Useful Links

- [Microsoft Agent Framework Docs](https://learn.microsoft.com/en-us/agent-framework/)
- [Agent Framework GitHub](https://github.com/microsoft/agent-framework)
- [MCP Protocol Specification](https://modelcontextprotocol.io/)
- [Dev UI Documentation](https://pypi.org/project/agent-framework-devui/)

---

💡 **Tip**: Start with Solution 1 to understand basics, then add thread management (Solution 2), and finally integrate MCP (Solution 3) for production-ready agents.
