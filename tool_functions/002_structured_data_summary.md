# Structured Data Extraction Using Claude Tool Schemas

## Introduction

The Claude API's tool use feature enables a powerful pattern: **structured data extraction**. By defining tool schemas without corresponding function implementations and using the `tool_choice` parameter, you can force Claude to output data in a specific, structured JSON format. This transforms Claude from generating free-form text into producing reliable, type-safe structured data.

### Key Applications

- Extracting structured information from unstructured text
- Parsing documents into consistent formats
- Converting natural language into JSON objects
- Ensuring consistent API response formats

---

## Conceptual Foundation

### Understanding Schema-Only Tools

In the 001_ examples, we saw tools that require both a schema AND a function implementation:

```python
# Traditional tool (from 001_)
get_current_datetime_schema = {...}  # Schema
def get_current_datetime(...):       # Function implementation
    return datetime.now().strftime(...)
```

However, for **structured extraction**, we only need the schema - no function required:

```python
# Schema-only tool (from 002_)
article_summary_schema = {...}  # Schema ONLY
# No function implementation needed!
```

**Why no function?** Because we're not executing anything. We're just telling Claude how to format its output.

---

## The `tool_choice` Parameter

The key to structured extraction is the `tool_choice` parameter, which forces Claude to use a specific tool.

### Implementation

Notice the `chat()` function in the 002_ notebooks includes `tool_choice`:

```python
def chat(
    messages,
    system=None,
    temperature=1.0,
    stop_sequences=[],
    tools=None,
    tool_choice=None,  # NEW parameter for forcing tool use
):
    params = {
        "model": model,
        "max_tokens": 1000,
        "messages": messages,
        "temperature": temperature,
        "stop_sequences": stop_sequences,
    }

    if tool_choice:
        params["tool_choice"] = tool_choice  # Add to API request
    
    if tools:
        params["tools"] = tools
    
    if system:
        params["system"] = system
    
    message = client.messages.create(**params)
    return message
```

### Usage

```python
# Force Claude to use a specific tool
tool_choice = {"type": "tool", "name": "article_summary"}
```

---

## Understanding `input_schema` Terminology

### Why Is It Called `input_schema`?

The field is named `input_schema` because the API was designed for traditional tool calling, where it defines **inputs to a function**. However, for structured extraction, this creates confusion:

| Usage Pattern | What `input_schema` Defines |
|---------------|----------------------------|
| **Traditional Tool** (001_) | Parameters your function receives (actual inputs) |
| **Structured Extraction** (002_) | Structure of data Claude generates (effectively outputs) |

**Key Point**: From Claude's perspective, these are "inputs" it provides when "calling" the tool. From your perspective in structured extraction, this is the **output structure** you extract.

Think of it as: **`input_schema` = The data structure Claude must generate**

---

## Complete Example: Article Summarization

This is the example from `002_structured_data_completed.ipynb`.

### Step 1: Define the Schema

```python
article_summary_schema = {
    "name": "article_summary",
    "description": """Creates a summary of an article with its key insights. 
    Use this tool when you need to generate a structured summary of an article, 
    research paper, or any textual content. The tool requires the article's title, 
    author name, and a list of the most important insights or takeaways from the 
    content. Each insight should be a concise statement capturing a significant 
    point from the article.""",
    "input_schema": {
        "type": "object",
        "properties": {
            "title": {
                "type": "string",
                "description": "The title of the article being summarized."
            },
            "author": {
                "type": "string",
                "description": "The name of the author who wrote the article."
            },
            "key_insights": {
                "type": "array",
                "items": {"type": "string"},
                "description": """A list of the most important takeaways or 
                insights from the article. Each insight should be a complete, 
                concise statement."""
            }
        },
        "required": ["title", "author", "key_insights"]
    }
}
```

### Step 2: Generate an Article (Optional)

First, let Claude generate some content to work with:

```python
messages = []
add_user_message(
    messages,
    """
    Write a one-paragraph scholarly article about computer science. 
    Include a title and author name.
    """
)
response = chat(messages)
article_text = text_from_message(response)
```

**Output:**
```
Advancements in Machine Learning Algorithms: Implications for Autonomous Systems

By Dr. Jonathan Mercer

Recent developments in machine learning algorithms have significantly expanded 
the capabilities of autonomous systems across diverse domains. Convolutional 
neural networks (CNNs) and transformer architectures have demonstrated remarkable 
efficacy in pattern recognition tasks, while reinforcement learning techniques 
continue to evolve, enabling systems to optimize decision-making processes through 
experience...
```

### Step 3: Extract Structured Data

Now force Claude to extract structured data using the schema:

```python
messages = []
add_user_message(messages, article_text)

response = chat(
    messages,
    tools=[article_summary_schema],
    tool_choice={"type": "tool", "name": "article_summary"}  # Force the tool
)

# Extract the structured data
structured_data = response.content[0].input
```

**Output:**
```python
{
    'title': 'Advancements in Machine Learning Algorithms: Implications for Autonomous Systems',
    'author': 'Dr. Jonathan Mercer',
    'key_insights': [
        'Convolutional neural networks (CNNs) and transformer architectures have shown remarkable effectiveness in pattern recognition tasks',
        'Reinforcement learning techniques are evolving to enable better decision-making processes through experience',
        'Algorithmic innovations combined with increased computational resources and larger datasets have accelerated progress in computer vision, NLP, and robotics',
        'Significant challenges remain in addressing algorithmic bias, interpretability, and ethical deployment in high-stakes environments',
        'The computer science community must balance innovation with responsible development practices to ensure autonomous systems are beneficial while mitigating risks to privacy, security, and social equity'
    ]
}
```

### How the Data Is Accessed

```python
# The structured data is in response.content[0].input
data = response.content[0].input

# Now you can access fields directly
title = data["title"]
author = data["author"]
insights = data["key_insights"]

# Use the structured data in your application
for insight in insights:
    print(f"- {insight}")
```

---

## How It Works

### The Response Structure

When you use `tool_choice` to force tool use, Claude's response contains a `ToolUseBlock`:

```python
response.content = [
    ToolUseBlock(
        id='toolu_01AbCdEfGhIj',
        name='article_summary',
        input={
            'title': '...',
            'author': '...',
            'key_insights': [...]
        },
        type='tool_use'
    )
]
```

**Important**: No function execution happens! You simply extract the data from `response.content[0].input`.

### The Key Difference

**Traditional Tool Use (001_):**
```
1. Claude calls tool with arguments
2. Your code EXECUTES the function
3. You return results to Claude
4. Claude continues conversation
```

**Structured Extraction (002_):**
```
1. You force Claude to "call" the tool
2. Claude generates structured data matching the schema
3. You EXTRACT the data directly (no execution)
4. You use the data in your application
```

---

## Schema Design Principles

Based on the `article_summary_schema` example:

### 1. Clear Descriptions

Provide detailed descriptions for both the tool and each property:

```python
"description": """Creates a summary of an article with its key insights. 
Use this tool when you need to generate a structured summary..."""
```

### 2. Appropriate Types

Use correct JSON Schema types:

```python
"title": {"type": "string"}         # Single value
"key_insights": {                   # Array of values
    "type": "array",
    "items": {"type": "string"}
}
```

### 3. Required Fields

Mark essential fields as required:

```python
"required": ["title", "author", "key_insights"]
```

### 4. Detailed Property Descriptions

Help Claude understand what each field should contain:

```python
"key_insights": {
    "type": "array",
    "items": {"type": "string"},
    "description": """A list of the most important takeaways or insights 
    from the article. Each insight should be a complete, concise statement."""
}
```

---

## Advantages Over Free-Form Text

### Without Structured Extraction

```python
# Generate free-form text
response = chat(messages)
text = text_from_message(response)

# Now you need to parse:
# - Where is the title? (First line? After #?)
# - Where is the author? (After "By"? After "Author:"?)
# - How are insights formatted? (Bullets? Numbers? Paragraphs?)
```

**Problems:**
- Inconsistent format between requests
- Brittle parsing code
- Hard to maintain
- Error-prone

### With Structured Extraction

```python
# Get structured data
response = chat(
    messages,
    tools=[article_summary_schema],
    tool_choice={"type": "tool", "name": "article_summary"}
)
data = response.content[0].input

# Direct, reliable access
title = data["title"]           # Always present, always a string
author = data["author"]         # Always present, always a string
insights = data["key_insights"] # Always present, always an array
```

**Benefits:**
- Consistent structure every time
- No parsing needed
- Type-safe access
- Easy to maintain

---

## Common Use Cases

The pattern demonstrated in the 002_ notebooks can be applied to many scenarios:

### Document Parsing
Extract titles, authors, sections, and summaries from documents.

### Entity Extraction
Identify and structure people, places, organizations from text.

### Data Normalization
Convert varying input formats into consistent structures.

### Form Filling
Extract information to populate structured forms.

### Content Classification
Categorize content with metadata (topic, sentiment, category).

---

## Best Practices

### 1. Schema Focus

Create focused schemas for specific tasks. The `article_summary_schema` focuses on just three fields: title, author, and insights.

### 2. Descriptive Names

Use clear, descriptive names:
- Good: `article_summary`, `key_insights`
- Avoid: `summarize`, `data`, `info`

### 3. Helpful Descriptions

Write descriptions that guide Claude:
```python
"description": """A list of the most important takeaways or insights 
from the article. Each insight should be a complete, concise statement."""
```

### 4. Required vs Optional

Only mark fields as required if they're truly essential. This gives Claude flexibility with varied inputs.

---

## Common Pitfalls

### Pitfall 1: Forgetting to Access `.input`

```python
# Wrong - trying to get text
text = response.content[0].text  # Error! No text in tool use response

# Correct - access structured data
data = response.content[0].input  # This is where the structured data is
```

### Pitfall 2: Not Using `tool_choice`

```python
# Wrong - Claude might not use the tool
response = chat(messages, tools=[article_summary_schema])

# Correct - Force Claude to use the tool
response = chat(
    messages,
    tools=[article_summary_schema],
    tool_choice={"type": "tool", "name": "article_summary"}
)
```

### Pitfall 3: Overly Vague Descriptions

```python
# Poor
"key_insights": {
    "type": "array",
    "description": "insights"
}

# Better
"key_insights": {
    "type": "array",
    "items": {"type": "string"},
    "description": """A list of the most important takeaways or insights 
    from the article. Each insight should be a complete, concise statement."""
}
```

---

## Summary

Structured data extraction using tool schemas is a powerful pattern demonstrated in the 002_ notebooks:

### Key Principles

1. **Schema Without Function**: Define only the schema, no function implementation needed
2. **Force Tool Use**: Use `tool_choice` parameter to ensure structured output
3. **Direct Extraction**: Access data from `response.content[0].input`
4. **Consistent Structure**: Every extraction follows the same format
5. **Type Safety**: JSON Schema provides type guarantees

### The Flow

```python
# 1. Define schema (no function)
article_summary_schema = {...}

# 2. Force Claude to use it
response = chat(
    messages,
    tools=[article_summary_schema],
    tool_choice={"type": "tool", "name": "article_summary"}
)

# 3. Extract structured data
data = response.content[0].input

# 4. Use directly in your application
title = data["title"]
```

This technique transforms Claude from a text generator into a reliable structured data extraction engine, perfect for applications requiring consistent, type-safe outputs.

