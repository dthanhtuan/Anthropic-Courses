# Claude API Tools: Complete Implementation Guide

## Introduction

The Anthropic Claude API provides a **tool use** capability (also referred to as function calling in other LLM APIs) that enables Claude to interact with external functions and APIs. This feature allows you to extend Claude's capabilities beyond text generation to include actions like data retrieval, computations, API calls, and structured data extraction.

## Core Concepts

### Tool Use Architecture

Tools in the Claude API are functions that you define and make available to Claude during a conversation. The architecture follows a request-response pattern:

**Client-side responsibilities:**
- Define tool schemas (specifications)
- Implement corresponding functions
- Execute tools when Claude requests them
- Return execution results to Claude

**Claude's responsibilities:**
- Analyze user requests to determine if tools are needed
- Select appropriate tools from available options
- Generate valid arguments based on the schema
- Process tool results and formulate responses

### Request-Response Flow

1. **Schema Definition**: Define tool schemas using JSON Schema format
2. **API Request**: Send messages with tool definitions to Claude
3. **Tool Selection**: Claude analyzes the request and decides whether to use tools
4. **Argument Generation**: Claude generates arguments conforming to the schema
5. **Tool Execution**: Your application executes the requested tools
6. **Result Transmission**: Tool outputs are sent back to Claude via user messages
7. **Response Generation**: Claude processes results and continues the conversation

---

## Tool Schema Definition

Tool schemas are JSON objects that describe a tool's purpose, parameters, and validation rules. They follow the JSON Schema specification and enable Claude to understand how to interact with your tools.

### Schema Structure

```python
tool_schema = {
    "name": "tool_name",
    "description": "Clear, comprehensive description of the tool's purpose and behavior",
    "input_schema": {
        "type": "object",
        "properties": {
            "parameter_name": {
                "type": "parameter_type",
                "description": "Detailed parameter description"
            }
        },
        "required": ["list", "of", "required", "parameters"]
    }
}
```

### Practical Example: DateTime Tool

```python
from anthropic.types import ToolParam
from datetime import datetime

get_current_datetime_schema = ToolParam({
    "name": "get_current_datetime",
    "description": """Returns the current date and time formatted according to a 
    specified format string. This tool provides access to the system's current 
    timestamp, useful for timestamping events, calculating time differences, or 
    displaying time information to users.""",
    "input_schema": {
        "type": "object",
        "properties": {
            "date_format": {
                "type": "string",
                "description": """Format string using Python's strftime format codes.
                Examples: '%Y-%m-%d' for date only, '%H:%M:%S' for time only,
                '%Y-%m-%d %H:%M:%S' for complete timestamp (default).""",
                "default": "%Y-%m-%d %H:%M:%S"
            }
        },
        "required": []
    }
})
```

### Schema Design Principles

**1. Comprehensive Descriptions**
Write detailed descriptions for both the tool and each parameter. Claude uses these to understand when and how to use the tool.

**2. Accurate Type Specifications**
Use appropriate JSON Schema types: `string`, `number`, `integer`, `boolean`, `array`, `object`, `null`.

**3. Required Field Management**
Mark parameters as required only when they are truly mandatory. Optional parameters provide flexibility.

**4. Default Value Documentation**
Document default values in parameter descriptions to help Claude understand fallback behavior.

**5. Validation Constraints**
Use JSON Schema validation keywords (`minimum`, `maximum`, `enum`, `pattern`) to constrain inputs.

---

## Tool Function Implementation

After defining schemas, implement the corresponding Python functions that will execute when Claude requests tool usage.

### Implementation Example

```python
from datetime import datetime

def get_current_datetime(date_format="%Y-%m-%d %H:%M:%S"):
    """
    Returns the current system datetime as a formatted string.
    
    Args:
        date_format: Python strftime format string
        
    Returns:
        Formatted datetime string
        
    Raises:
        ValueError: If date_format is empty
    """
    if not date_format:
        raise ValueError("date_format cannot be empty")
    return datetime.now().strftime(date_format)
```

### Tool Execution Router

Create a routing function to dispatch tool calls to appropriate implementations:

```python
def run_tool(tool_name: str, tool_input: dict):
    """
    Routes tool execution requests to the appropriate function.
    
    Args:
        tool_name: Name of the tool to execute
        tool_input: Dictionary of arguments for the tool
        
    Returns:
        Tool execution result
        
    Raises:
        ValueError: If tool_name is not recognized
    """
    tool_functions = {
        "get_current_datetime": get_current_datetime,
        "add_duration_to_datetime": add_duration_to_datetime,
        "set_reminder": set_reminder
    }
    
    if tool_name not in tool_functions:
        raise ValueError(f"Unknown tool: {tool_name}")
        
    return tool_functions[tool_name](**tool_input)
```

---

## API Integration

### Including Tools in API Requests

```python
from anthropic import Anthropic

client = Anthropic()

response = client.messages.create(
    model="claude-3-5-sonnet-20241022",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "What is the current time?"}
    ],
    tools=[get_current_datetime_schema, add_duration_to_datetime_schema]
)
```

### Detecting Tool Usage

Claude's response will contain `tool_use` blocks when it determines tools are needed:

```python
# Response structure when Claude uses a tool
response.content = [
    TextBlock(
        text="I'll check the current time for you.",
        type="text"
    ),
    ToolUseBlock(
        id="toolu_01A2B3C4D5E6F7G8H9I0J1K2",
        name="get_current_datetime",
        input={"date_format": "%H:%M:%S"},
        type="tool_use"
    )
]
```

### Executing Tools

Extract tool use blocks and execute the requested functions:

```python
import json
from typing import List, Dict

def run_tools(message) -> List[Dict]:
    """
    Executes all tool requests in a message and returns formatted results.
    
    Args:
        message: Claude API response message object
        
    Returns:
        List of tool_result blocks ready to send back to Claude
    """
    tool_requests = [
        block for block in message.content 
        if block.type == "tool_use"
    ]
    tool_result_blocks = []
    
    for tool_request in tool_requests:
        try:
            tool_output = run_tool(tool_request.name, tool_request.input)
            tool_result_block = {
                "type": "tool_result",
                "tool_use_id": tool_request.id,
                "content": json.dumps(tool_output),
                "is_error": False
            }
        except Exception as e:
            tool_result_block = {
                "type": "tool_result",
                "tool_use_id": tool_request.id,
                "content": f"Error: {str(e)}",
                "is_error": True
            }
        
        tool_result_blocks.append(tool_result_block)
    
    return tool_result_blocks
```

### Returning Results to Claude

Tool execution results must be sent back as user messages:

```python
# Add Claude's response (including tool use) to message history
messages.append({
    "role": "assistant",
    "content": response.content
})

# Execute tools and get results
tool_results = run_tools(response)

# Send tool results back as user message
messages.append({
    "role": "user",
    "content": tool_results
})

# Continue conversation with tool results
next_response = client.messages.create(
    model="claude-3-5-sonnet-20241022",
    max_tokens=1024,
    messages=messages,
    tools=[get_current_datetime_schema, add_duration_to_datetime_schema]
)
```

---

## Conversation Loop Implementation

Implement a conversation loop to handle multi-turn interactions with tool usage:

```python
def run_conversation(messages: List[Dict], tools: List[Dict]) -> List[Dict]:
    """
    Manages a conversation loop that handles tool execution.
    
    Args:
        messages: List of conversation messages
        tools: List of available tool schemas
        
    Returns:
        Complete message history including tool interactions
    """
    while True:
        # Request response from Claude
        response = client.messages.create(
            model="claude-3-5-sonnet-20241022",
            max_tokens=1024,
            messages=messages,
            tools=tools
        )
        
        # Add Claude's response to message history
        messages.append({
            "role": "assistant",
            "content": response.content
        })
        
        # Extract and display text content
        text_content = [
            block.text for block in response.content 
            if block.type == "text"
        ]
        if text_content:
            print("\n".join(text_content))
        
        # Check if conversation is complete
        if response.stop_reason != "tool_use":
            break
        
        # Execute requested tools
        tool_results = run_tools(response)
        
        # Send tool results back to Claude
        messages.append({
            "role": "user",
            "content": tool_results
        })
    
    return messages
```

### Usage Example

```python
messages = [{
    "role": "user",
    "content": "What is the current time in HH:MM format?"
}]

tools = [
    get_current_datetime_schema,
    add_duration_to_datetime_schema,
    set_reminder_schema
]

final_messages = run_conversation(messages, tools)
```

---

## Advanced Pattern: Batch Tool Execution

Batch tools enable Claude to invoke multiple tools simultaneously, improving efficiency for operations that can be parallelized.

### Batch Tool Schema

```python
batch_tool_schema = {
    "name": "batch_tool",
    "description": "Executes multiple tool invocations concurrently",
    "input_schema": {
        "type": "object",
        "properties": {
            "invocations": {
                "type": "array",
                "description": "Array of tool invocation specifications",
                "items": {
                    "type": "object",
                    "properties": {
                        "name": {
                            "type": "string",
                            "description": "Name of the tool to invoke"
                        },
                        "arguments": {
                            "type": "string",
                            "description": "Tool arguments encoded as JSON string"
                        }
                    },
                    "required": ["name", "arguments"]
                }
            }
        },
        "required": ["invocations"]
    }
}
```

### Batch Tool Implementation

```python
import json
from typing import List, Dict, Any

def run_batch(invocations: List[Dict[str, str]]) -> List[Dict[str, Any]]:
    """
    Executes multiple tool invocations and returns aggregated results.
    
    Args:
        invocations: List of tool invocation specifications, each containing
                    'name' and 'arguments' (JSON string)
    
    Returns:
        List of execution results with tool names and outputs
    """
    batch_output = []
    
    for invocation in invocations:
        tool_name = invocation["name"]
        tool_args = json.loads(invocation["arguments"])
        
        try:
            tool_output = run_tool(tool_name, tool_args)
            batch_output.append({
                "tool_name": tool_name,
                "output": tool_output,
                "error": None
            })
        except Exception as e:
            batch_output.append({
                "tool_name": tool_name,
                "output": None,
                "error": str(e)
            })
    
    return batch_output
```

### Use Case

Batch tools are useful when multiple independent operations need to be performed:

```python
# Claude can request:
# - Current datetime
# - Add duration to a date
# - Set multiple reminders
# All in a single tool invocation, reducing round trips
```

---

## Message Flow Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│ USER REQUEST                                                     │
│ "What is the current time in HH:MM format?"                     │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│ API REQUEST (with tool schemas)                                 │
│ messages: [{"role": "user", "content": "..."}]                  │
│ tools: [get_current_datetime_schema, ...]                      │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│ CLAUDE RESPONSE                                                  │
│ content: [                                                       │
│   TextBlock("I'll check the time"),                            │
│   ToolUseBlock(name="get_current_datetime",                    │
│                input={"date_format": "%H:%M"})                 │
│ ]                                                                │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│ TOOL EXECUTION (Application Code)                               │
│ result = get_current_datetime(date_format="%H:%M")             │
│ # Returns: "14:30"                                              │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│ TOOL RESULT MESSAGE (User Role)                                 │
│ {                                                                │
│   "type": "tool_result",                                        │
│   "tool_use_id": "toolu_...",                                   │
│   "content": "\"14:30\""                                         │
│ }                                                                │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│ CLAUDE FINAL RESPONSE                                            │
│ content: [                                                       │
│   TextBlock("The current time is 14:30.")                      │
│ ]                                                                │
└─────────────────────────────────────────────────────────────────┘
```

---

## Implementation Guidelines

### Tool Result Message Format

**Critical**: Tool results must be sent as user messages. This is required by the API design because tool execution happens externally to Claude.

```python
# Correct: Tool results in user message
messages.append({
    "role": "user",
    "content": tool_results
})

# Incorrect: Tool results should not be in assistant message
# This will cause an API error
```

### Result Serialization

Tool outputs should be serialized to JSON for consistent formatting:

```python
tool_result = {
    "type": "tool_result",
    "tool_use_id": tool_request.id,
    "content": json.dumps(tool_output),  # Serialize to JSON string
    "is_error": False
}
```

### Error Handling

Implement robust error handling for tool execution:

```python
try:
    tool_output = run_tool(tool_request.name, tool_request.input)
    is_error = False
    content = json.dumps(tool_output)
except Exception as e:
    is_error = True
    content = f"Error executing {tool_request.name}: {str(e)}"
    # Log error for debugging
    logger.error(f"Tool execution failed: {e}", exc_info=True)
```

### Stop Reason Interpretation

The `stop_reason` field indicates why Claude stopped generating:

- `"tool_use"`: Claude wants to use a tool; continue the loop
- `"end_turn"`: Natural conversation end; exit the loop
- `"max_tokens"`: Token limit reached; may need to increase max_tokens
- `"stop_sequence"`: Custom stop sequence encountered

---

## Reference Implementations

### Tool Examples from Course Materials

**get_current_datetime**
- Purpose: Returns current system timestamp
- Parameters: Optional format string
- Use case: Timestamping, time-based operations

**add_duration_to_datetime**
- Purpose: Performs datetime arithmetic
- Parameters: DateTime string, duration value, time unit, format
- Use case: Date calculations, scheduling

**set_reminder**
- Purpose: Schedules user notifications
- Parameters: Content message, timestamp
- Use case: Task management, time-based alerts

**batch_tool**
- Purpose: Executes multiple tools concurrently
- Parameters: Array of tool invocations
- Use case: Reducing round-trip latency for independent operations

---

## Quick Reference Guide

| Phase | Action | Implementation Pattern |
|-------|--------|------------------------|
| **Define** | Create schema | `{"name": "...", "description": "...", "input_schema": {...}}` |
| **Implement** | Write function | `def tool_name(param1, param2=default): ...` |
| **Register** | Add to API call | `tools=[schema1, schema2, ...]` |
| **Detect** | Check response | `if block.type == "tool_use": ...` |
| **Execute** | Run function | `output = run_tool(block.name, block.input)` |
| **Return** | Send as user msg | `{"type": "tool_result", "tool_use_id": ..., "content": ...}` |
| **Loop** | Continue until done | `while response.stop_reason == "tool_use": ...` |

---

## Conclusion

Tools extend Claude's capabilities from pure text generation to interactive, action-oriented applications. Key takeaways:

1. **Schema-driven**: Tool definitions use JSON Schema for type safety and validation
2. **Client-side execution**: Your application executes tools and returns results
3. **Conversational integration**: Tools fit naturally into multi-turn conversations
4. **Error resilient**: Proper error handling ensures robust applications
5. **Flexible patterns**: Support for single tools, batch operations, and complex workflows

This foundation enables building production applications that leverage Claude's language understanding with custom functionality, API integrations, and structured data processing.

