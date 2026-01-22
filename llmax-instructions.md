# LLMax Execution Instructions for LLM Agents

This document provides complete instructions for executing `.llmax` pipeline files as an LLM agent. LLMax is an LLM-agnostic protocol for defining executable agentic workflows.

---

## Table of Contents

1. [File Structure](#1-file-structure)
2. [Parsing Rules](#2-parsing-rules)
3. [Variable System](#3-variable-system)
4. [Instruction Execution](#4-instruction-execution)
5. [Control Flow](#5-control-flow)
6. [LLM Calls with JSON Schema](#6-llm-calls-with-json-schema)
7. [Error Handling](#7-error-handling)
8. [Execution Order](#8-execution-order)
9. [Complete Example](#9-complete-example)

---

## 1. File Structure

Every `.llmax` file follows this structure:

```llmax
#!llmax/1.0

@name: "Pipeline Name"
@version: "1.0.0"
@description: "Optional description"
@author: "Optional author"

@vars {
  API_KEY: env("API_KEY")
  BASE_URL: "https://api.example.com"
  config: {
    timeout: 30,
    retries: 3
  }
}

---

@pipeline main {
  # Instructions go here
}
```

### Required Elements

| Element | Required | Description |
|---------|----------|-------------|
| `#!llmax/1.0` | Yes | Version shebang, must be first line |
| `@name` | Yes | Pipeline name |
| `@version` | Yes | Semantic version |
| `@pipeline main` | Yes | Main entry point |

### Optional Elements

| Element | Description |
|---------|-------------|
| `@description` | Human-readable description |
| `@author` | Author information |
| `@vars` | Global variable definitions |
| `---` | Section separator (visual only) |

---

## 2. Parsing Rules

### Comments
```llmax
# Single line comment
// Also valid single line comment
```

### String Literals
```llmax
"Double quoted string"
'Single quoted string'
"String with \"escaped\" quotes"
"Multi-line strings
are allowed"
```

### Numbers
```llmax
42        # Integer
3.14      # Float
-10       # Negative
1e6       # Scientific notation
```

### Booleans and Null
```llmax
true
false
null
```

### Objects
```llmax
{
  name: "John",
  age: 30,
  nested: {
    key: "value"
  }
}
```

### Arrays
```llmax
[1, 2, 3]
["a", "b", "c"]
[{ id: 1 }, { id: 2 }]
```

### Duration Values
```llmax
30s     # 30 seconds
5m      # 5 minutes
2h      # 2 hours
1d      # 1 day
```

---

## 3. Variable System

### Variable Declaration

Variables are declared in `@vars` block or created via instruction output:

```llmax
@vars {
  # Static value
  API_URL: "https://api.example.com"

  # Environment variable
  API_KEY: env("API_KEY")

  # Constant (cannot be reassigned)
  const MAX_RETRIES: 3

  # Object
  config: {
    timeout: 30,
    debug: true
  }

  # Array
  allowed_types: ["text", "json", "xml"]
}
```

### Variable Reference Syntax

Use curly braces `{variableName}` to reference variables:

```llmax
# Simple reference
{username}

# Nested path access
{response.data.user.name}

# Array index
{items[0]}
{items[-1]}              # Last element

# Default fallback (if null/undefined)
{config.timeout ?? 30}

# String interpolation
"Hello, {user.name}! You have {count} messages."
```

### JSONPath for Complex Extraction

```llmax
$.field                    # Direct field access
$.parent.child.value       # Nested access
$[0]                       # First array element
$[-1]                      # Last array element
$[0:3]                     # Array slice (indices 0-2)
$[*]                       # All array elements
$.items[*].name            # All 'name' fields from items array
$..id                      # Recursive descent - all 'id' fields
$[?(@.status == 'active')] # Filter by condition
```

### Reserved Variables

| Variable | Description | Context |
|----------|-------------|---------|
| `$index` | Current iteration index (0-based) | Loop |
| `$item` | Current item being iterated | Loop |
| `$result` | Result of last instruction | Global |
| `$error` | Current error object | Catch block |
| `$env` | Environment variables accessor | Global |
| `$now` | Current ISO timestamp | Global |

---

## 4. Instruction Execution

### 4.1 getInput

Prompts user for input with validation.

```llmax
getInput(
  prompt: "Enter your name:",
  validate: text,           # text|number|email|url|date|json|/regex/
  default: "Anonymous",     # Default if empty
  minLength: 1,
  maxLength: 100,
  options: ["a", "b", "c"], # Multiple choice
  secret: false             # Hide input (for passwords)
) -> userName
```

**Execution:**
1. Display prompt to user
2. Collect input
3. Validate against rules
4. If invalid, re-prompt with error message
5. Store result in output variable

**Output:** The validated input value (string, number, etc.)

---

### 4.2 apiCall

Makes HTTP API requests.

```llmax
apiCall(
  url: "https://api.example.com/users/{userId}",
  method: GET,              # GET|POST|PUT|PATCH|DELETE
  headers: {
    "Authorization": "Bearer {API_KEY}",
    "Content-Type": "application/json"
  },
  query: {
    limit: 10,
    offset: 0
  },
  body: {
    name: "{userName}",
    email: "{userEmail}"
  },
  timeout: 30s,
  retry: {
    maxAttempts: 3,
    delay: 1s,
    backoff: "exponential"
  }
) -> response
```

**Execution:**
1. Interpolate all variables in url, headers, query, body
2. Construct HTTP request
3. Send request with timeout
4. On failure, retry according to retry config
5. Parse response

**Output Structure:**
```json
{
  "status": 200,
  "statusText": "OK",
  "headers": {
    "content-type": "application/json",
    "x-request-id": "abc123"
  },
  "data": { ... },
  "error": null
}
```

On error:
```json
{
  "status": 500,
  "statusText": "Internal Server Error",
  "headers": { ... },
  "data": null,
  "error": "Connection timeout"
}
```

---

### 4.3 assign

Assigns values or extracts data.

```llmax
# Direct assignment
assign(value: "Hello World") -> greeting

# From variable
assign(value: {response.data}) -> userData

# JSONPath extraction
assign(
  source: {response},
  path: "$.data.users[*].name"
) -> userNames

# Expression evaluation
assign(expr: "{price} * {quantity} * (1 + {taxRate})") -> total

# Object construction
assign(value: {
  id: {user.id},
  fullName: "{user.firstName} {user.lastName}",
  timestamp: {$now}
}) -> userRecord
```

**Execution:**
1. If `value` provided: evaluate and assign directly
2. If `source` + `path`: extract using JSONPath
3. If `expr`: evaluate mathematical/string expression
4. Store result in output variable

---

### 4.4 llmCall

Invokes an LLM with prompt and optional structured output.

```llmax
llmCall(
  prompt: "Analyze this text: {inputText}",
  system: "You are a helpful assistant.",
  model: "default",
  temperature: 0.7,
  maxTokens: 1000,
  schema: {
    type: "object",
    properties: {
      sentiment: {
        type: "string",
        enum: ["positive", "negative", "neutral"]
      },
      summary: {
        type: "string",
        maxLength: 200
      },
      topics: {
        type: "array",
        items: { type: "string" }
      }
    },
    required: ["sentiment", "summary"]
  }
) -> analysis
```

**Execution:**
1. Interpolate variables in prompt and system
2. If `schema` provided, instruct LLM to return valid JSON matching schema
3. Send to LLM API
4. If schema provided, parse and validate JSON response
5. Store result in output variable

**Output:**
- With schema: Parsed JSON object matching schema
- Without schema: Raw text string

See [Section 6](#6-llm-calls-with-json-schema) for detailed schema instructions.

---

### 4.5 webSearch

Searches the web.

```llmax
webSearch(
  query: "latest {topic} news",
  maxResults: 10,
  freshness: "week",        # day|week|month|year
  site: "example.com",      # Limit to domain
  extract: "summary"        # links|summary|full
) -> searchResults
```

**Execution:**
1. Interpolate variables in query
2. Execute web search
3. Filter/limit results
4. Extract content based on `extract` setting

**Output Structure:**
```json
{
  "results": [
    {
      "title": "Article Title",
      "url": "https://...",
      "snippet": "Brief excerpt...",
      "content": "Full content if extract=full"
    }
  ],
  "totalResults": 1000
}
```

---

### 4.6 fileRead

Reads file content.

```llmax
fileRead(
  path: "./data/config.json",
  encoding: "utf-8",        # utf-8|base64|binary
  parseAs: "json"           # text|json|lines
) -> fileContent
```

**Output:**
- `parseAs: text` → String
- `parseAs: json` → Parsed object
- `parseAs: lines` → Array of strings

---

### 4.7 fileWrite

Writes content to file.

```llmax
fileWrite(
  path: "./output/result.json",
  content: {resultData},
  encoding: "utf-8",
  mode: "overwrite"         # overwrite|append
)
```

---

### 4.8 output

Outputs final result.

```llmax
output(
  value: {finalResult},
  format: "json",           # text|json|markdown|html
  label: "Analysis Result"
)
```

**Execution:**
1. Format value according to format type
2. Display/return to user with optional label

---

### 4.9 log

Debug logging (does not affect flow).

```llmax
log(
  message: "Processing user {userId}",
  data: {userData},
  level: "info"             # debug|info|warn|error
)
```

---

### 4.10 javascript

Executes custom JavaScript code.

```llmax
javascript {
  code: `
    const items = {input_data};
    const filtered = items.filter(i => i.active);
    const mapped = filtered.map(i => ({
      id: i.id,
      name: i.name.toUpperCase()
    }));
    return mapped;
  `
} -> processedItems
```

**Execution:**
1. Interpolate variables in code
2. Execute JavaScript in sandboxed environment
3. Return value becomes output variable

**Important:** Code MUST include a `return` statement.

---

## 5. Control Flow

### 5.1 Conditional (if/elif/else)

```llmax
@if ({status} == "success") {
  output(value: "Operation successful!")
} @elif ({status} == "pending") {
  log(message: "Still processing...")
  # Wait and retry logic
} @else {
  output(value: "Operation failed: {error}")
}
```

**Comparison Operators:**

| Operator | Description | Example |
|----------|-------------|---------|
| `==` | Equal | `{a} == "test"` |
| `!=` | Not equal | `{a} != null` |
| `>` | Greater than | `{count} > 10` |
| `<` | Less than | `{count} < 100` |
| `>=` | Greater or equal | `{score} >= 80` |
| `<=` | Less or equal | `{score} <= 100` |
| `&&` | Logical AND | `{a} > 0 && {b} > 0` |
| `||` | Logical OR | `{a} == null || {a} == ""` |
| `!` | Logical NOT | `!{isDisabled}` |

**Special Operators:**

```llmax
# Contains (for strings and arrays)
@if ({text} contains "error") { ... }
@if ({tags} contains "urgent") { ... }

# Regex match
@if ({email} matches /^[\w.-]+@[\w.-]+\.\w+$/) { ... }

# Type check
@if ({value} is string) { ... }
@if ({data} is array) { ... }
```

---

### 5.2 For-Each Loop

```llmax
@foreach (user in {users}) {
  log(message: "Processing: {user.name}")

  apiCall(
    url: "{API_URL}/users/{user.id}/profile",
    method: GET
  ) -> profile

  assign(value: {
    id: {user.id},
    name: {user.name},
    profile: {profile.data}
  }) -> enrichedUser
}
```

**Available in loop:**
- `{user}` - Current item (or whatever name you specify)
- `{$index}` - Current index (0-based)
- `{$item}` - Alias for current item

---

### 5.3 Range Loop

```llmax
@for (i in range(1, 10)) {
  log(message: "Page {i}")
  apiCall(url: "{API_URL}/items?page={i}") -> page
}

# With step
@for (i in range(0, 100, 10)) {
  # 0, 10, 20, 30, ...
}
```

---

### 5.4 While Loop

```llmax
assign(value: true) -> hasMore
assign(value: 1) -> page

@while ({hasMore} == true) {
  apiCall(url: "{API_URL}/items?page={page}") -> response

  # Process items...

  assign(value: {response.data.hasNextPage}) -> hasMore
  assign(expr: "{page} + 1") -> page
}
```

---

### 5.5 Loop Control

```llmax
@foreach (item in {items}) {
  # Skip invalid items
  @if ({item.status} == "invalid") {
    @continue
  }

  # Stop if we found what we need
  @if ({item.id} == {targetId}) {
    assign(value: {item}) -> foundItem
    @break
  }

  # Process item...
}
```

---

### 5.6 Parallel Execution

```llmax
@parallel {
  @branch fetchUsers {
    apiCall(url: "{API_URL}/users") -> users
  }

  @branch fetchProducts {
    apiCall(url: "{API_URL}/products") -> products
  }

  @branch fetchOrders {
    apiCall(url: "{API_URL}/orders") -> orders
  }
}

# All results available after @parallel block completes
log(message: "Loaded {users.data.length} users")
log(message: "Loaded {products.data.length} products")
```

**Execution:**
1. Start all branches concurrently
2. Wait for ALL branches to complete
3. All output variables available after block

---

### 5.7 Switch/Case

```llmax
@switch ({action}) {
  @case "create" {
    apiCall(url: "{API_URL}/items", method: POST, body: {data}) -> result
  }

  @case "update" {
    apiCall(url: "{API_URL}/items/{id}", method: PUT, body: {data}) -> result
  }

  @case "delete" {
    apiCall(url: "{API_URL}/items/{id}", method: DELETE) -> result
  }

  @default {
    log(message: "Unknown action: {action}", level: "warn")
    assign(value: null) -> result
  }
}
```

---

## 6. LLM Calls with JSON Schema

When you need structured output from an LLM, provide a JSON Schema:

### Basic Schema

```llmax
llmCall(
  prompt: "Extract person info from: {text}",
  schema: {
    type: "object",
    properties: {
      name: { type: "string" },
      age: { type: "number" },
      email: { type: "string" }
    },
    required: ["name"]
  }
) -> person
```

**Expected Output:**
```json
{
  "name": "John Doe",
  "age": 30,
  "email": "john@example.com"
}
```

### Complex Nested Schema

```llmax
llmCall(
  prompt: "Analyze this article: {article}",
  schema: {
    type: "object",
    properties: {
      title: { type: "string" },
      summary: {
        type: "string",
        maxLength: 200
      },
      sentiment: {
        type: "string",
        enum: ["positive", "negative", "neutral", "mixed"]
      },
      topics: {
        type: "array",
        items: { type: "string" }
      },
      entities: {
        type: "array",
        items: {
          type: "object",
          properties: {
            name: { type: "string" },
            type: {
              type: "string",
              enum: ["person", "organization", "location", "product"]
            },
            relevance: { type: "number" }
          },
          required: ["name", "type"]
        }
      },
      metadata: {
        type: "object",
        properties: {
          wordCount: { type: "integer" },
          readingTime: { type: "string" },
          language: { type: "string" }
        }
      }
    },
    required: ["title", "summary", "sentiment"]
  }
) -> analysis
```

**Expected Output:**
```json
{
  "title": "Tech Industry Sees Major Growth",
  "summary": "The technology sector reported significant gains this quarter...",
  "sentiment": "positive",
  "topics": ["technology", "business", "finance"],
  "entities": [
    { "name": "Apple Inc", "type": "organization", "relevance": 0.9 },
    { "name": "Tim Cook", "type": "person", "relevance": 0.7 }
  ],
  "metadata": {
    "wordCount": 1500,
    "readingTime": "6 min",
    "language": "en"
  }
}
```

### Schema Type Reference

| Type | Description | Constraints |
|------|-------------|-------------|
| `string` | Text value | `maxLength`, `minLength`, `pattern`, `format`, `enum` |
| `number` | Decimal number | `minimum`, `maximum`, `exclusiveMinimum`, `exclusiveMaximum` |
| `integer` | Whole number | Same as number |
| `boolean` | true/false | - |
| `array` | List of items | `items`, `minItems`, `maxItems`, `uniqueItems` |
| `object` | Key-value pairs | `properties`, `required`, `additionalProperties` |
| `null` | Null value | - |

### String Formats

```llmax
{ type: "string", format: "date" }        # "2024-01-15"
{ type: "string", format: "date-time" }   # "2024-01-15T10:30:00Z"
{ type: "string", format: "time" }        # "10:30:00"
{ type: "string", format: "email" }       # "user@example.com"
{ type: "string", format: "uri" }         # "https://example.com"
{ type: "string", format: "uuid" }        # "550e8400-e29b-41d4-a716-446655440000"
```

### LLM Instructions for Schema Compliance

When executing an `llmCall` with a schema, instruct the LLM:

```
You must respond with valid JSON that matches this schema:
{schema}

Rules:
1. Output ONLY the JSON object, no additional text
2. Include all required fields
3. Use correct data types (string, number, boolean, array, object)
4. For enum fields, use only the allowed values
5. Respect maxLength and other constraints
6. For arrays, ensure items match the items schema
```

---

## 7. Error Handling

### Try-Catch-Finally

```llmax
@try {
  apiCall(url: "{API_URL}/data") -> response

  @if ({response.status} != 200) {
    @throw ValidationError("API returned {response.status}")
  }

  assign(value: {response.data}) -> result

} @catch (NetworkError) {
  log(message: "Network error: {$error.message}", level: "error")
  assign(value: null) -> result

} @catch (TimeoutError) {
  log(message: "Request timed out", level: "error")
  assign(value: null) -> result

} @catch {
  # Catch all other errors
  log(message: "Unexpected error: {$error.message}", level: "error")
  @throw  # Re-throw the error

} @finally {
  log(message: "Request attempt completed")
}
```

### Error Types

| Error Type | Description |
|------------|-------------|
| `NetworkError` | Network/connectivity issues |
| `TimeoutError` | Operation exceeded timeout |
| `ValidationError` | Input validation failed |
| `AuthError` | Authentication/authorization failed |
| `NotFoundError` | Resource not found (404) |
| `RateLimitError` | Rate limit exceeded (429) |
| `ParseError` | JSON/data parsing failed |
| `SchemaError` | Output doesn't match schema |

### Error Object Properties

```llmax
{$error.type}      # Error type name
{$error.message}   # Human-readable message
{$error.code}      # Error code (if available)
{$error.details}   # Additional details object
```

### Assertions

```llmax
@assert ({response.status} == 200) "API call failed with status {response.status}"
@assert ({items}.length > 0) "No items returned"
@assert ({user.email} matches /^[\w.-]+@[\w.-]+\.\w+$/) "Invalid email format"
```

If assertion fails, throws `AssertionError` with the provided message.

---

## 8. Execution Order

### Sequential Execution

Instructions execute top-to-bottom within a block:

```llmax
@pipeline main {
  getInput(prompt: "Name:") -> name           # 1. Execute first
  getInput(prompt: "Email:") -> email         # 2. Execute second

  apiCall(url: "...", body: {                 # 3. Execute third
    name: {name},                             #    (uses results from 1 & 2)
    email: {email}
  }) -> response

  output(value: {response.data})              # 4. Execute last
}
```

### Variable Scope

1. **Global scope**: Variables in `@vars` block
2. **Pipeline scope**: Variables created within pipeline
3. **Block scope**: Loop variables, catch block `$error`

```llmax
@vars {
  GLOBAL_VAR: "available everywhere"
}

@pipeline main {
  assign(value: "pipeline var") -> pipelineVar

  @foreach (item in {items}) {
    # {item} and {$index} only available inside loop
    # {GLOBAL_VAR} and {pipelineVar} still accessible
  }

  # {item} not available here
}
```

### Execution State

The executor maintains:

1. **Variable store**: All current variable values
2. **Call stack**: Current execution position
3. **Error state**: Current error (in catch blocks)

---

## 9. Complete Example

```llmax
#!llmax/1.0

@name: "Research Assistant"
@version: "1.0.0"
@description: "Researches a topic and generates a summary report"

@vars {
  API_KEY: env("OPENAI_API_KEY")
  MAX_SOURCES: 5
}

---

@pipeline main {
  # Get research topic from user
  getInput(
    prompt: "What topic would you like me to research?",
    validate: text,
    minLength: 3
  ) -> topic

  log(message: "Researching: {topic}")

  # Search for information
  webSearch(
    query: "{topic} latest information",
    maxResults: {MAX_SOURCES},
    freshness: "month"
  ) -> searchResults

  # Check if we got results
  @if ({searchResults.results}.length == 0) {
    output(value: "No results found for '{topic}'")
    @return
  }

  # Process each result
  assign(value: []) -> sources

  @foreach (result in {searchResults.results}) {
    log(message: "Processing: {result.title}")

    assign(value: {
      title: {result.title},
      url: {result.url},
      snippet: {result.snippet}
    }) -> source

    # Add to sources array
    assign(expr: "{sources}.concat([{source}])") -> sources
  }

  # Generate analysis using LLM
  llmCall(
    prompt: "Based on these sources about '{topic}', provide a comprehensive analysis:\n\n{sources}",
    system: "You are a research analyst. Provide factual, well-organized analysis.",
    temperature: 0.3,
    schema: {
      type: "object",
      properties: {
        summary: {
          type: "string",
          description: "Executive summary (2-3 sentences)"
        },
        keyFindings: {
          type: "array",
          items: { type: "string" },
          description: "List of key findings"
        },
        sentiment: {
          type: "string",
          enum: ["positive", "negative", "neutral", "mixed"]
        },
        confidence: {
          type: "number",
          minimum: 0,
          maximum: 1,
          description: "Confidence score"
        },
        suggestedFollowUp: {
          type: "array",
          items: { type: "string" },
          description: "Suggested follow-up questions"
        }
      },
      required: ["summary", "keyFindings", "sentiment"]
    }
  ) -> analysis

  # Build final report
  assign(value: {
    topic: {topic},
    generatedAt: {$now},
    sourceCount: {sources}.length,
    analysis: {analysis},
    sources: {sources}
  }) -> report

  # Output the report
  output(
    value: {report},
    format: "json",
    label: "Research Report"
  )

  log(message: "Research complete!", level: "info")
}
```

### Expected Execution Flow

1. **getInput** → User enters "artificial intelligence trends"
2. **log** → "Researching: artificial intelligence trends"
3. **webSearch** → Returns 5 search results
4. **@if** → Check passes (results found)
5. **@foreach** → Loop 5 times, building sources array
6. **llmCall** → Sends prompt with sources, receives structured JSON:
   ```json
   {
     "summary": "AI continues rapid advancement with focus on...",
     "keyFindings": [
       "Large language models dominate research",
       "Enterprise adoption accelerating",
       "Ethical concerns increasing"
     ],
     "sentiment": "mixed",
     "confidence": 0.85,
     "suggestedFollowUp": [
       "What are the main ethical concerns?",
       "Which industries are adopting AI fastest?"
     ]
   }
   ```
7. **assign** → Build final report object
8. **output** → Display formatted JSON report
9. **log** → "Research complete!"

---

## Appendix: Quick Reference

### Instruction Summary

| Instruction | Purpose | Output |
|-------------|---------|--------|
| `getInput` | Get user input | Input value |
| `apiCall` | HTTP request | Response object |
| `assign` | Set/transform value | Assigned value |
| `llmCall` | AI completion | Text or JSON |
| `webSearch` | Web search | Results array |
| `fileRead` | Read file | File content |
| `fileWrite` | Write file | - |
| `output` | Display result | - |
| `log` | Debug log | - |
| `javascript` | Custom code | Return value |

### Control Flow Summary

| Construct | Syntax |
|-----------|--------|
| Conditional | `@if (...) { } @elif (...) { } @else { }` |
| For-each | `@foreach (item in {array}) { }` |
| Range | `@for (i in range(start, end)) { }` |
| While | `@while (condition) { }` |
| Parallel | `@parallel { @branch name { } }` |
| Switch | `@switch (value) { @case "x" { } @default { } }` |
| Try-catch | `@try { } @catch (Error) { } @finally { }` |
| Break/Continue | `@break`, `@continue` |
| Return | `@return {value}` |
| Assert | `@assert (condition) "message"` |
| Throw | `@throw ErrorType("message")` |

---

*LLMax v1.0 - LLM-Agnostic Pipeline Protocol*
