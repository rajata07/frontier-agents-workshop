# Scenario 1: Comprehensive Solution Guide

## Table of Contents
1. [Overview](#overview)
2. [Learning Objectives](#learning-objectives)
3. [Problem Statement](#problem-statement)
4. [Solution Architecture](#solution-architecture)
5. [Solution 1: Basic Agent with Function Calling](#solution-1-basic-agent-with-function-calling)
6. [Solution 2: Agent with Thread Management](#solution-2-agent-with-thread-management)
7. [Solution 3: Agent with MCP Server Integration](#solution-3-agent-with-mcp-server-integration)
8. [MCP Servers](#mcp-servers)
9. [Implementation Guide](#implementation-guide)
10. [Testing and Validation](#testing-and-validation)
11. [Troubleshooting](#troubleshooting)

## Overview

Scenario 1 introduces you to building your first intelligent agent using the Microsoft Agent Framework. This scenario demonstrates how to create an agent that can answer questions about time and weather while maintaining conversational memory across multiple turns. You'll learn the fundamental concepts of agent development, including tool integration, conversational state management, and external service connections through the Model Context Protocol (MCP).

### Why This Matters

Real-world AI agents need to:
- **Use tools and functions** to access external information beyond their training data
- **Maintain conversational context** to provide coherent, multi-turn interactions
- **Integrate with external services** to perform actions and retrieve dynamic data
- **Remember user preferences** and information shared during conversations

This scenario teaches these essential skills through a practical example: building a time and weather assistant.

## Learning Objectives

By completing this scenario, you will learn how to:

1. **Define and configure agents** using the Microsoft Agent Framework
2. **Connect Python functions as tools** that agents can call
3. **Manage conversational state** across multiple user interactions
4. **Integrate MCP servers** to extend agent capabilities
5. **Debug and trace agent execution** using the Agent Framework Dev UI
6. **Handle multi-step reasoning** where agents use multiple tools to answer questions

## Problem Statement

### The Challenge

Build an agent that can:
- Determine the user's location from conversation
- Answer questions about the current time at the user's location
- Provide weather information for the user's location
- Remember the user's location across multiple questions
- Update the location when the user moves

### Example Conversation

```
User: "I am currently in London"
Agent: "Got it! You're in London. How can I help you?"

User: "What is the weather now here?"
Agent: "The weather in London is currently cloudy with a high of 18°C."

User: "What time is it for me right now?"
Agent: "It's currently 3:45 PM in London."

User: "I moved to Berlin, what is the weather like today?"
Agent: "You've moved to Berlin! The weather there is sunny with a high of 22°C."

User: "Can you remind me where I said I am based?"
Agent: "You mentioned you're currently in Berlin."
```

## Solution Architecture

The solution uses a three-layer architecture:

```
┌─────────────────────────────────────────────────────┐
│                    User Interface                    │
│              (Console / Dev UI / AG-UI)              │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│                   Agent Layer                        │
│  ┌──────────────────────────────────────────────┐   │
│  │         ChatAgent with Instructions          │   │
│  ├──────────────────────────────────────────────┤   │
│  │  • Language Model (GPT-4, etc.)             │   │
│  │  • Conversation Thread Management           │   │
│  │  • Tool/Function Calling                    │   │
│  └──────────────────────────────────────────────┘   │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│                   Tools Layer                        │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────┐ │
│  │   Python     │  │  User Info   │  │  Weather  │ │
│  │  Functions   │  │ MCP Server   │  │MCP Server │ │
│  └──────────────┘  └──────────────┘  └───────────┘ │
└─────────────────────────────────────────────────────┘
```

### Key Components

1. **ChatAgent**: The core agent that processes user queries and orchestrates tool calls
2. **Language Model**: The AI model (e.g., GPT-4) that powers the agent's reasoning
3. **AgentThread**: Manages conversational state and message history
4. **Tools**: Functions and MCP servers that the agent can call
5. **Dev UI**: Web-based interface for debugging and monitoring agent behavior

## Solution 1: Basic Agent with Function Calling

### Concept

The simplest approach is to create an agent with a Python function that it can call as a tool. This demonstrates the core function-calling capability without the complexity of threads or external servers.

### Key Features

- Direct function integration
- Automatic function calling by the language model
- Simple, single-turn interactions

### Code Example: `basic-agent.py`

```python
from agent_framework import ChatAgent
from typing import Annotated
from pydantic import Field
import asyncio
from random import randint

# Define a tool function
def get_weather_at_location(
    location: Annotated[str, Field(description="The location to get the weather for.")],
) -> str:
    """Get the realtime weather for a given location."""
    conditions = ["sunny", "cloudy", "rainy", "stormy"]
    return f"The weather in {location} is {conditions[randint(0, 3)]} with a high of {randint(10, 30)}°C."

async def main() -> None:
    # Create a chat client
    client = create_chat_client("gpt-4")
    
    # Create the agent with the tool
    agent = ChatAgent(
        chat_client=client,
        instructions="You are a helpful weather agent.",
        tools=get_weather_at_location,  # Attach the function as a tool
    )
    
    # Ask a question
    message = "What's the weather in Amsterdam and in Paris?"
    print(f"User: {message}")
    
    # Get response (agent will call the function automatically)
    response = await agent.run(message)
    print(f"Assistant: {response}")

asyncio.run(main())
```

### How It Works

1. **Function Definition**: The `get_weather_at_location` function is defined with:
   - Type hints (using `Annotated` and `Field`) that describe parameters
   - A docstring that explains what the function does
   - Return type annotation

2. **Tool Registration**: The function is passed to `ChatAgent` via the `tools` parameter

3. **Automatic Calling**: When the user asks about weather, the language model:
   - Recognizes it needs weather information
   - Identifies the `get_weather_at_location` function
   - Calls it with appropriate arguments (e.g., "Amsterdam", "Paris")
   - Incorporates the results into its response

### Limitations

- **No memory**: Each call is independent; the agent doesn't remember previous interactions
- **Single turn**: Cannot handle follow-up questions that reference earlier context
- **Stateless**: Cannot track user location across multiple queries

## Solution 2: Agent with Thread Management

### Concept

Add conversation thread management to enable multi-turn conversations where the agent remembers previous exchanges.

### Key Features

- Conversational memory across multiple interactions
- Context preservation
- Follow-up question handling
- Stateful conversations

### Code Example: `agent-thread.py`

```python
from agent_framework import AgentThread, ChatAgent, ChatMessageStore
import asyncio

def get_weather(location: Annotated[str, Field(description="The location")]) -> str:
    """Get the weather for a given location."""
    conditions = ["sunny", "cloudy", "rainy", "stormy"]
    return f"The weather in {location} is {conditions[randint(0, 3)]} with a high of {randint(10, 30)}°C."

async def main() -> None:
    client = create_chat_client("gpt-4")
    
    agent = ChatAgent(
        chat_client=client,
        instructions="You are a helpful weather agent.",
        tools=get_weather,
    )
    
    # Create a thread to maintain conversation history
    thread = agent.get_new_thread()
    
    # First conversation
    query1 = "What's the weather like in Tokyo?"
    print(f"User: {query1}")
    result1 = await agent.run(query1, thread=thread)
    print(f"Agent: {result1.text}")
    
    # Second conversation - agent remembers Tokyo from previous message
    query2 = "How about London?"
    print(f"\nUser: {query2}")
    result2 = await agent.run(query2, thread=thread)
    print(f"Agent: {result2.text}")
    
    # Third conversation - agent can reference both cities
    query3 = "Which of the cities I asked about has better weather?"
    print(f"\nUser: {query3}")
    result3 = await agent.run(query3, thread=thread)
    print(f"Agent: {result3.text}")

asyncio.run(main())
```

### How It Works

1. **Thread Creation**: `agent.get_new_thread()` creates an `AgentThread` object that stores conversation history

2. **Message Storage**: Each interaction is stored in the thread:
   - User messages
   - Agent responses
   - Tool calls and results

3. **Context Passing**: When you call `agent.run(query, thread=thread)`:
   - The entire conversation history is included in the context
   - The model can reference previous messages
   - The agent "remembers" what was discussed

4. **Persistent State**: The thread object can be reused across multiple `agent.run()` calls

### Thread Management Patterns

#### Pattern 1: Automatic Thread Creation
```python
# No thread provided - creates a new thread each time (no memory)
result1 = await agent.run("What's the weather in Seattle?")
result2 = await agent.run("What was the last city?")  # Won't remember Seattle
```

#### Pattern 2: Thread Persistence
```python
# Same thread across calls - maintains memory
thread = agent.get_new_thread()
result1 = await agent.run("What's the weather in Seattle?", thread=thread)
result2 = await agent.run("What was the last city?", thread=thread)  # Remembers Seattle
```

#### Pattern 3: Thread Restoration
```python
# Save and restore thread state
thread = agent.get_new_thread()
await agent.run("I'm in London", thread=thread)

# Get message history
messages = await thread.message_store.list_messages()

# Create new thread from existing messages
restored_thread = AgentThread(message_store=ChatMessageStore(messages))
await agent.run("What's my location?", thread=restored_thread)  # Remembers London
```

### Benefits

- **Contextual responses**: Agent can reference previous messages
- **Natural conversations**: Users can ask follow-up questions without repeating context
- **User preference tracking**: Agent remembers user's location, preferences, etc.

## Solution 3: Agent with MCP Server Integration

### Concept

Integrate with external MCP (Model Context Protocol) servers to access specialized capabilities like user information management and weather data. This demonstrates how to connect agents to external services.

### Key Features

- External service integration via MCP
- Separation of concerns (agent logic vs. data sources)
- Reusable MCP servers across multiple agents
- Standard protocol for tool exposure

### Code Example: `agents-using-mcp.py`

```python
from agent_framework import ChatAgent, MCPStreamableHTTPTool
import asyncio

async def main() -> None:
    client = create_chat_client("gpt-4")
    
    # Connect to weather MCP server
    weather_tool = MCPStreamableHTTPTool(
        name="Weather Server", 
        url="http://localhost:8001/mcp"
    )
    
    # Create agent with MCP tool
    async with ChatAgent(
        chat_client=client,
        name="WeatherAgent",
        instructions="You are a helpful assistant that provides weather information.",
        tools=weather_tool,
    ) as agent:
        # Create thread for conversation memory
        thread = agent.get_new_thread()
        
        # First query
        query1 = "What is the weather in Amsterdam?"
        print(f"User: {query1}")
        result1 = await agent.run(query1, thread=thread)
        print(f"Agent: {result1.text}")
        
        # Follow-up query
        query2 = "How about in New York?"
        print(f"\nUser: {query2}")
        result2 = await agent.run(query2, thread=thread)
        print(f"Agent: {result2.text}")

asyncio.run(main())
```

### How It Works

1. **MCP Server Connection**: 
   - `MCPStreamableHTTPTool` creates a connection to an MCP server via HTTP
   - The MCP server exposes tools/functions that the agent can discover and call
   - Communication uses the standardized MCP protocol

2. **Tool Discovery**: 
   - The agent queries the MCP server to discover available tools
   - Each tool includes metadata: name, description, parameters
   - The language model uses this metadata to decide when to call tools

3. **Function Execution**:
   - When the model decides to call a tool, it sends a request to the MCP server
   - The MCP server executes the function and returns results
   - The agent incorporates results into its response

4. **Multi-Server Support**:
   ```python
   # Connect to multiple MCP servers
   user_tool = MCPStreamableHTTPTool(
       name="User Server",
       url="http://localhost:8002/mcp"
   )
   weather_tool = MCPStreamableHTTPTool(
       name="Weather Server",
       url="http://localhost:8001/mcp"
   )
   
   agent = ChatAgent(
       chat_client=client,
       tools=[user_tool, weather_tool],  # Multiple tools
   )
   ```

### MCP Protocol Benefits

- **Standardization**: Common protocol for tool exposure
- **Language agnostic**: MCP servers can be written in any language
- **Security**: Can implement approval workflows for sensitive operations
- **Discoverability**: Tools are self-describing via metadata
- **Scalability**: Servers can be deployed independently and scaled

## MCP Servers

### User Information MCP Server

**Location**: `src/mcp-server/02-user-server/server-mcp-sse-user.py`

**Purpose**: Manages user information including current location and time zone data.

#### Available Tools

1. **`get_current_user()`**
   - Returns the username of the current user
   - No parameters required
   - Returns: String (username)

2. **`get_current_location(username: str)`**
   - Gets the current timezone location for a given user
   - Parameters:
     - `username`: The user's name
   - Returns: String (timezone in format like "Europe/Berlin")

3. **`get_current_time(location: str)`**
   - Gets current time in a specific location
   - Parameters:
     - `location`: Timezone location (e.g., "America/New_York", "Europe/London")
   - Returns: String (formatted time like "03:45:30 PM")

4. **`move(username: str, newlocation: str)`**
   - Updates a user's location
   - Parameters:
     - `username`: The user's name
     - `newlocation`: New timezone location
   - Returns: Boolean (success/failure)

#### User Data Structure

```python
users = {
    "Dennis": {
        "name": "Dennis",
        "location": "Europe/Berlin",
    },
    "John": {
        "name": "John",
        "location": "America/New_York",
    },
}
```

#### Running the Server

```bash
cd src/mcp-server/02-user-server
python server-mcp-sse-user.py
```

Server runs on: `http://localhost:8002`
MCP endpoint: `http://localhost:8002/mcp`

### Weather MCP Server

**Location**: `src/mcp-server/04-weather-server/server-mcp-sse-weather.py`

**Purpose**: Provides weather information based on location and time of day.

#### Available Tools

1. **`list_supported_locations()`**
   - Lists all supported locations
   - No parameters
   - Returns: List of strings (location names)
   - Supported locations: Seattle, New York, London, Berlin, Tokyo, Sydney

2. **`get_weather_at_location(location: str)`**
   - Gets weather for a single location
   - Parameters:
     - `location`: City name (e.g., "London", "Tokyo")
   - Returns: String with weather description including:
     - Current local time
     - Time of day category (morning/afternoon/evening/night)
     - Weather conditions

3. **`get_weather_for_multiple_locations(locations: list[str])`**
   - Gets weather for multiple locations at once
   - Parameters:
     - `locations`: List of city names
   - Returns: List of weather descriptions

#### Weather Logic

The server provides different weather descriptions based on time of day:

- **Morning (5-11 AM)**: "Cool and clear with a light breeze."
- **Afternoon (12-5 PM)**: "Mild temperatures with scattered clouds and good visibility."
- **Evening (6-9 PM)**: "Calm conditions with a gentle breeze and fading light."
- **Night (10 PM-4 AM)**: "Quiet, mostly clear skies and cooler air."

#### Supported Locations and Time Zones

```python
LOCATIONS = {
    "Seattle": "America/Los_Angeles",
    "New York": "America/New_York",
    "London": "Europe/London",
    "Berlin": "Europe/Berlin",
    "Tokyo": "Asia/Tokyo",
    "Sydney": "Australia/Sydney",
}
```

#### Running the Server

```bash
cd src/mcp-server/04-weather-server
python server-mcp-sse-weather.py
```

Server runs on: `http://localhost:8001`
MCP endpoint: `http://localhost:8001/mcp`

## Implementation Guide

### Step-by-Step Implementation

#### Step 1: Environment Setup

1. **Install dependencies**:
   ```bash
   pip install -r requirements.txt
   ```

2. **Configure environment variables**:
   Create a `.env` file based on `.env.example`:
   ```bash
   GITHUB_TOKEN="your-github-pat"
   # Or for Azure OpenAI:
   AZURE_OPENAI_API_KEY="your-key"
   AZURE_OPENAI_ENDPOINT="https://your-resource.openai.azure.com/openai/v1/"
   ```

#### Step 2: Start MCP Servers

Open two terminal windows:

**Terminal 1 - Weather Server**:
```bash
cd src/mcp-server/04-weather-server
python server-mcp-sse-weather.py
```

**Terminal 2 - User Server**:
```bash
cd src/mcp-server/02-user-server
python server-mcp-sse-user.py
```

Verify servers are running:
- Weather: http://localhost:8001/mcp
- User: http://localhost:8002/mcp

#### Step 3: Create Your Agent

Create a new file `src/scenarios/01-hello-world-agent/my_agent.py`:

```python
import sys
from pathlib import Path
sys.path.insert(0, str(Path(__file__).parent.parent.parent.parent))

from samples.shared.model_client import create_chat_client
import os
import asyncio
from agent_framework import ChatAgent, MCPStreamableHTTPTool
from dotenv import load_dotenv

load_dotenv()

async def main():
    # Create chat client
    model_name = os.environ.get("SMALL_DEPLOYMENT_MODEL_NAME")
    client = create_chat_client(model_name)
    
    # Connect to MCP servers
    user_tool = MCPStreamableHTTPTool(
        name="User Server",
        url="http://localhost:8002/mcp"
    )
    
    weather_tool = MCPStreamableHTTPTool(
        name="Weather Server",
        url="http://localhost:8001/mcp"
    )
    
    # Create agent
    async with ChatAgent(
        chat_client=client,
        name="TimeWeatherAgent",
        instructions="""You are a helpful assistant that helps users with time and weather information.
        When a user tells you their location, remember it for future questions.
        Use the available tools to get accurate time and weather information.""",
        tools=[user_tool, weather_tool],
    ) as agent:
        # Create conversation thread
        thread = agent.get_new_thread()
        
        # Test queries
        queries = [
            "I am currently in London",
            "What is the weather now here?",
            "What time is it for me right now?",
            "I moved to Berlin, what is the weather like today?",
            "Can you remind me where I said I am based?"
        ]
        
        for query in queries:
            print(f"\nUser: {query}")
            result = await agent.run(query, thread=thread)
            print(f"Agent: {result.text}")

if __name__ == "__main__":
    asyncio.run(main())
```

#### Step 4: Run Your Agent

```bash
cd src/scenarios/01-hello-world-agent
python my_agent.py
```

#### Step 5: Use the Dev UI

Start the Agent Framework Dev UI for debugging:

```bash
agent-framework-devui
```

Open http://localhost:5000 in your browser to:
- View conversation threads
- Inspect tool calls
- Monitor token usage
- View traces and metrics
- Debug agent behavior

### Advanced Features

#### Custom Instructions

Improve agent behavior with detailed instructions:

```python
instructions = """You are a time and weather assistant with the following capabilities:

1. Location Management:
   - Remember the user's location when they tell you
   - Update location when they move
   - Ask for clarification if location is ambiguous

2. Time Information:
   - Provide current time in the user's timezone
   - Use the get_current_time tool with proper timezone format
   - Handle timezone conversions if asked

3. Weather Information:
   - Provide current weather conditions
   - Use get_weather_at_location for single locations
   - Use get_weather_for_multiple_locations for comparisons
   - Consider time of day when describing weather

4. Conversation Style:
   - Be friendly and helpful
   - Acknowledge when you remember information from earlier messages
   - Provide context in your responses"""
```

#### Error Handling

```python
async def run_agent_with_error_handling():
    try:
        async with ChatAgent(...) as agent:
            thread = agent.get_new_thread()
            result = await agent.run(query, thread=thread)
            return result
    except Exception as e:
        print(f"Error: {e}")
        # Handle gracefully
        return "I encountered an error. Please try again."
```

#### Streaming Responses

For real-time output:

```python
async for chunk in agent.run_stream(query, thread=thread):
    if chunk.text:
        print(chunk.text, end="", flush=True)
print()  # New line after streaming completes
```

## Testing and Validation

### Test Scenarios

#### Test 1: Location Memory
```python
queries = [
    "I am in Tokyo",
    "What's the weather here?",  # Should use Tokyo
    "What time is it?",  # Should use Tokyo timezone
]
```

**Expected**: Agent remembers Tokyo for subsequent queries.

#### Test 2: Location Updates
```python
queries = [
    "I am in London",
    "What's the weather?",  # London weather
    "I moved to Berlin",
    "What's the weather now?",  # Berlin weather
    "Where am I?",  # Should say Berlin
]
```

**Expected**: Agent correctly updates and remembers new location.

#### Test 3: Multiple Tool Calls
```python
query = "What's the time and weather in New York?"
```

**Expected**: Agent calls both time and weather tools, combines results.

#### Test 4: Unsupported Location
```python
query = "What's the weather in Antarctica?"
```

**Expected**: Agent handles gracefully, explains limitation.

### Debugging with Dev UI

1. **View Tool Calls**:
   - Check which tools were called
   - Inspect arguments passed to each tool
   - Verify return values

2. **Examine Thread State**:
   - Review message history
   - Check if context is properly maintained
   - Look for missing information

3. **Monitor Performance**:
   - Track response times
   - Check token usage
   - Identify bottlenecks

4. **Trace Execution**:
   - Follow the complete execution flow
   - See decision-making process
   - Understand tool selection logic

### Manual Testing Checklist

- [ ] Agent correctly extracts location from user messages
- [ ] Agent remembers location across multiple turns
- [ ] Time queries use correct timezone
- [ ] Weather queries return location-appropriate data
- [ ] Agent updates location when user moves
- [ ] Agent handles ambiguous locations appropriately
- [ ] Follow-up questions work without repeating context
- [ ] Agent provides helpful responses when tools fail
- [ ] Thread state persists across multiple interactions
- [ ] MCP servers respond correctly to all tool calls

## Troubleshooting

### Common Issues

#### Issue 1: Agent Doesn't Call Tools

**Symptoms**: Agent provides generic responses instead of calling weather/time tools.

**Solutions**:
- Check that tools are properly registered: `tools=[user_tool, weather_tool]`
- Verify MCP servers are running: `curl http://localhost:8001/mcp`
- Improve instructions to explicitly mention tool usage
- Check language model supports function calling (GPT-3.5+)

#### Issue 2: Agent Forgets Context

**Symptoms**: Agent doesn't remember previous messages.

**Solutions**:
- Ensure you're passing the same `thread` object to all `agent.run()` calls
- Check thread creation: `thread = agent.get_new_thread()`
- Verify thread parameter: `await agent.run(query, thread=thread)`
- Consider using `store=True` to explicitly save messages

#### Issue 3: MCP Server Connection Failures

**Symptoms**: Errors about connection refused or timeout.

**Solutions**:
- Verify MCP servers are running: `ps aux | grep server-mcp`
- Check ports 8001 and 8002 are not in use by other processes
- Ensure correct URLs in tool configuration
- Review server logs for errors
- Test endpoints manually: `curl http://localhost:8001/mcp`

#### Issue 4: Incorrect Time/Weather Data

**Symptoms**: Agent returns wrong time or unexpected weather.

**Solutions**:
- Verify timezone format (use IANA format: "America/New_York")
- Check user location data in user server
- Confirm supported locations: only 6 cities in weather server
- Review time bucket logic (morning/afternoon/evening/night)

#### Issue 5: Environment Variable Issues

**Symptoms**: Model initialization fails or authentication errors.

**Solutions**:
- Check `.env` file exists and has correct values
- Verify `GITHUB_TOKEN` or `AZURE_OPENAI_API_KEY` is set
- Ensure model deployment names match `.env` configuration
- Use `python -c "from dotenv import load_dotenv; load_dotenv(); import os; print(os.environ.get('GITHUB_TOKEN'))"` to test

### Debugging Tips

1. **Enable Verbose Logging**:
   ```python
   import logging
   logging.basicConfig(level=logging.DEBUG)
   ```

2. **Print Thread Messages**:
   ```python
   messages = await thread.message_store.list_messages()
   for msg in messages:
       print(f"{msg.role}: {msg.text}")
   ```

3. **Test MCP Servers Independently**:
   ```bash
   # Test weather server
   curl -X POST http://localhost:8001/mcp \
     -H "Content-Type: application/json" \
     -d '{"method": "tools/call", "params": {"name": "list_supported_locations"}}'
   ```

4. **Simplify for Debugging**:
   - Start with one MCP server
   - Test with one query at a time
   - Use print statements to trace execution
   - Remove thread management temporarily to isolate issues

### Getting Help

If you're still stuck:

1. **Check Server Logs**: Review output from MCP server terminals
2. **Review Sample Code**: Compare your code with working samples
3. **Test Each Component**: Isolate the problem (agent vs. MCP vs. thread)
4. **Use Dev UI**: Inspect traces and activities for detailed debugging
5. **Consult Documentation**: https://learn.microsoft.com/en-us/agent-framework/

## Summary

You've learned three progressive approaches to building agents:

1. **Basic Agent**: Simple function calling for stateless interactions
2. **Thread Management**: Adding conversational memory for multi-turn dialogs
3. **MCP Integration**: Connecting to external services for scalable, reusable capabilities

These patterns form the foundation for building more complex multi-agent systems in later scenarios.

### Key Takeaways

- **Tools extend agent capabilities** beyond the language model's knowledge
- **Threads enable memory** for natural multi-turn conversations
- **MCP provides standardization** for connecting to external services
- **Instructions guide behavior** and improve agent reliability
- **Dev UI aids debugging** through visualization and tracing

### Next Steps

- **Scenario 2**: Build a custom UI for your agent using AG-UI protocol
- **Scenario 3**: Connect multiple agents using A2A protocol
- **Scenario 4**: Orchestrate workflows across multiple agents
- Continue exploring more advanced multi-agent patterns!
