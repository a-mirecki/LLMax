# LLMax Instructions Reference

This document provides detailed reference for all core LLMax instructions.

## getInput

Prompts the user for input with optional validation.

### Syntax

```llmax
getInput(
  prompt: string,
  validate?: ValidationRule,
  default?: any,
  minLength?: number,
  maxLength?: number,
  options?: array,
  secret?: boolean
) -> outputVariable
```

### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `prompt` | string | Yes | The prompt shown to the user |
| `validate` | ValidationRule | No | Validation type or regex |
| `default` | any | No | Default value if input is empty |
| `minLength` | number | No | Minimum string length |
| `maxLength` | number | No | Maximum string length |
| `options` | array | No | Allowed values (enum-style) |
| `secret` | boolean | No | Mask input (for passwords) |

### Built-in Validation Rules

| Rule | Description | Example Valid Input |
|------|-------------|---------------------|
| `text` | Any non-empty string | "Hello" |
| `number` | Any numeric value | 42, -3.14 |
| `int` | Integer only | 42, -10 |
| `positiveInt` | Positive integer | 1, 100 |
| `positiveNumber` | Positive number | 0.5, 100 |
| `email` | Valid email format | "user@example.com" |
| `url` | Valid URL | "https://example.com" |
| `date` | ISO date (YYYY-MM-DD) | "2025-01-21" |
| `datetime` | ISO datetime | "2025-01-21T10:30:00Z" |
| `json` | Valid JSON string | '{"key": "value"}' |
| `/regex/` | Custom regex pattern | Depends on pattern |

### Examples

```llmax
# Basic text input
getInput(prompt: "What's your name?") -> name

# With validation
getInput(prompt: "Enter your email", validate: email) -> userEmail

# With default value
getInput(prompt: "Enter quantity", validate: positiveInt, default: 1) -> qty

# Custom regex validation
getInput(prompt: "Enter code", validate: /^[A-Z]{3}-[0-9]{4}$/) -> code

# Password input (masked)
getInput(prompt: "Enter API key", secret: true) -> apiKey

# Selection from options
getInput(
  prompt: "Select environment",
  options: ["development", "staging", "production"]
) -> env

# With length constraints
getInput(
  prompt: "Describe the issue",
  validate: text,
  minLength: 10,
  maxLength: 500
) -> description
```

---

## apiCall

Makes HTTP API requests.

### Syntax

```llmax
apiCall(
  url: string,
  method?: HttpMethod,
  headers?: object,
  query?: object,
  body?: any,
  auth?: AuthConfig,
  timeout?: duration,
  retry?: RetryConfig,
  followRedirects?: boolean
) -> outputVariable
```

### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `url` | string | Yes | The endpoint URL |
| `method` | HttpMethod | No | GET, POST, PUT, PATCH, DELETE (default: GET) |
| `headers` | object | No | HTTP headers |
| `query` | object | No | Query string parameters |
| `body` | any | No | Request body (auto JSON-encoded if object) |
| `auth` | AuthConfig | No | Authentication configuration |
| `timeout` | duration | No | Request timeout (default: 30s) |
| `retry` | RetryConfig | No | Retry configuration |
| `followRedirects` | boolean | No | Follow redirects (default: true) |

### HTTP Methods

- `GET` - Retrieve data
- `POST` - Create resource
- `PUT` - Replace resource
- `PATCH` - Update resource
- `DELETE` - Delete resource

### Authentication Types

```llmax
# Bearer token
auth: bearer("{token}")

# Basic auth
auth: basic("{username}", "{password}")

# API key in header
auth: apiKey(header: "X-API-Key", value: "{key}")

# API key in query
auth: apiKey(query: "api_key", value: "{key}")

# OAuth2 client credentials
auth: oauth2(
  tokenUrl: "https://auth.example.com/token",
  clientId: "{clientId}",
  clientSecret: "{clientSecret}",
  scope: "read write"
)
```

### Retry Configuration

```llmax
retry: {
  maxAttempts: 3,
  delay: 1s,
  backoff: exponential,  # or: fixed, linear
  retryOn: [NetworkError, TimeoutError, RateLimitError]
}
```

### Response Structure

The output variable contains:

```json
{
  "status": 200,
  "statusText": "OK",
  "headers": { "content-type": "application/json" },
  "data": { /* parsed response body */ }
}
```

### Examples

```llmax
# Simple GET request
apiCall(url: "https://api.example.com/users") -> users

# GET with query parameters
apiCall(
  url: "https://api.example.com/search",
  query: { q: "{searchTerm}", limit: 10 }
) -> results

# POST with JSON body
apiCall(
  url: "https://api.example.com/users",
  method: POST,
  headers: { "Content-Type": "application/json" },
  body: {
    name: "{userName}",
    email: "{userEmail}"
  }
) -> newUser

# With authentication
apiCall(
  url: "https://api.github.com/user/repos",
  auth: bearer("{GITHUB_TOKEN}"),
  timeout: 60s
) -> repos

# With retry logic
apiCall(
  url: "https://unstable-api.example.com/data",
  retry: {
    maxAttempts: 5,
    delay: 2s,
    backoff: exponential
  }
) -> data
```

---

## assign

Assigns values to variables or extracts data from objects.

### Syntax

```llmax
assign(
  value?: any,
  source?: variable,
  path?: string,
  expr?: string
) -> outputVariable
```

### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `value` | any | No* | Direct value to assign |
| `source` | variable | No* | Source variable for extraction |
| `path` | string | No | JSONPath expression |
| `expr` | string | No | Mathematical/string expression |

*One of `value`, `source` (with `path`), or `expr` is required.

### JSONPath Syntax

| Pattern | Description | Example |
|---------|-------------|---------|
| `$.field` | Direct field access | `$.name` |
| `$.a.b.c` | Nested field | `$.user.address.city` |
| `$[0]` | Array index | `$.items[0]` |
| `$[-1]` | Last array element | `$.items[-1]` |
| `$[0:3]` | Array slice | `$.items[0:3]` |
| `$[*]` | All array elements | `$.items[*].name` |
| `$..field` | Recursive descent | `$..id` |
| `$[?(...)]` | Filter expression | `$[?(@.active == true)]` |

### Expression Syntax

Expressions support:
- Arithmetic: `+`, `-`, `*`, `/`, `%`
- Comparison: `==`, `!=`, `>`, `<`, `>=`, `<=`
- Logical: `&&`, `||`, `!`
- String concatenation: `+`
- Parentheses for grouping

### Examples

```llmax
# Direct value assignment
assign(value: "Hello World") -> greeting
assign(value: 42) -> count
assign(value: { name: "John", age: 30 }) -> user

# Extract from JSON response
assign(source: {apiResponse}, path: "$.data") -> data
assign(source: {apiResponse}, path: "$.data.users[0].name") -> firstName
assign(source: {apiResponse}, path: "$.items[*].id") -> allIds

# Filter array
assign(
  source: {items},
  path: "$[?(@.status == 'active')]"
) -> activeItems

# Mathematical expression
assign(expr: "{price} * {quantity}") -> subtotal
assign(expr: "{subtotal} * (1 + {taxRate})") -> total
assign(expr: "{count} + 1") -> count

# String concatenation
assign(expr: "{firstName} + ' ' + {lastName}") -> fullName

# Construct new object
assign(value: {
  id: "{userId}",
  timestamp: "{$now}",
  data: {processedData}
}) -> record
```

---

## llmCall

Invokes an LLM with a prompt and optional structured output.

### Syntax

```llmax
llmCall(
  prompt: string,
  system?: string,
  schema?: JSONSchema,
  model?: string,
  temperature?: number,
  maxTokens?: number,
  stopSequences?: array,
  context?: array
) -> outputVariable
```

### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `prompt` | string | Yes | The prompt to send |
| `system` | string | No | System message/context |
| `schema` | JSONSchema | No | Structured output schema |
| `model` | string | No | Model identifier (executor default) |
| `temperature` | number | No | Sampling temperature (0.0-2.0) |
| `maxTokens` | number | No | Maximum response tokens |
| `stopSequences` | array | No | Stop generation sequences |
| `context` | array | No | Additional context documents |

### Structured Output Schema

When `schema` is provided, the LLM response is validated and parsed:

```llmax
llmCall(
  prompt: "Extract information from: {text}",
  schema: {
    type: "object",
    properties: {
      title: { type: "string" },
      summary: { type: "string", maxLength: 200 },
      tags: { type: "array", items: { type: "string" } },
      sentiment: {
        type: "string",
        enum: ["positive", "negative", "neutral"]
      }
    },
    required: ["title", "sentiment"]
  }
) -> extracted
```

### Examples

```llmax
# Simple prompt
llmCall(prompt: "What is the capital of France?") -> answer

# With system message
llmCall(
  prompt: "Analyze this code for bugs: {code}",
  system: "You are an expert code reviewer. Be concise and specific."
) -> review

# Creative generation
llmCall(
  prompt: "Write a tagline for a coffee shop named '{shopName}'",
  temperature: 0.9,
  maxTokens: 50
) -> tagline

# Structured extraction
llmCall(
  prompt: "Parse this receipt: {receiptText}",
  schema: {
    type: "object",
    properties: {
      vendor: { type: "string" },
      date: { type: "string", format: "date" },
      total: { type: "number" },
      items: {
        type: "array",
        items: {
          type: "object",
          properties: {
            name: { type: "string" },
            price: { type: "number" }
          }
        }
      }
    },
    required: ["vendor", "total"]
  }
) -> receipt

# Multi-turn with context
llmCall(
  prompt: "What's the next step based on this analysis?",
  context: [
    { role: "user", content: "{previousQuestion}" },
    { role: "assistant", content: "{previousAnswer}" }
  ]
) -> nextStep
```

---

## webSearch

Searches the web and returns results.

### Syntax

```llmax
webSearch(
  query: string,
  maxResults?: number,
  freshness?: string,
  site?: string,
  extract?: string
) -> outputVariable
```

### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `query` | string | Yes | Search query |
| `maxResults` | number | No | Maximum results (default: 10) |
| `freshness` | string | No | Time filter: day, week, month, year |
| `site` | string | No | Limit to specific domain |
| `extract` | string | No | "links", "summary", "full" |

### Response Structure

```json
{
  "results": [
    {
      "title": "Page Title",
      "url": "https://example.com/page",
      "snippet": "Brief description...",
      "content": "Full content if extract=full"
    }
  ],
  "totalResults": 100
}
```

### Examples

```llmax
# Basic search
webSearch(query: "latest AI news") -> news

# Limited results with freshness
webSearch(
  query: "{topic} research papers",
  maxResults: 5,
  freshness: "month"
) -> papers

# Site-specific search
webSearch(
  query: "React hooks tutorial",
  site: "reactjs.org"
) -> docs

# Get full content
webSearch(
  query: "{productName} reviews",
  maxResults: 3,
  extract: "full"
) -> reviews
```

---

## output

Outputs results from the pipeline.

### Syntax

```llmax
output(
  value: any,
  format?: string,
  label?: string
)
```

### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `value` | any | Yes | Value to output |
| `format` | string | No | Format: text, json, markdown, html |
| `label` | string | No | Output label/name |

### Examples

```llmax
# Simple text output
output(value: "Process completed!")

# JSON output
output(value: {results}, format: json)

# Markdown formatted
output(value: {report}, format: markdown)

# Labeled output
output(value: {summary}, format: text, label: "Executive Summary")

# Multiple outputs
output(value: {data}, format: json, label: "Raw Data")
output(value: {analysis}, format: markdown, label: "Analysis")
```

---

## log

Logs messages for debugging.

### Syntax

```llmax
log(
  message: string,
  data?: any,
  level?: string
)
```

### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `message` | string | Yes | Log message |
| `data` | any | No | Additional data to log |
| `level` | string | No | debug, info, warn, error (default: info) |

### Examples

```llmax
# Simple log
log(message: "Starting process")

# With data
log(message: "API Response", data: {response}, level: debug)

# Warning
log(message: "Retrying request", level: warn)

# Error
log(message: "Failed to process: {$error.message}", level: error)

# With variable interpolation
log(message: "Processing item {$index} of {totalCount}")
```

---

## Summary Table

| Instruction | Purpose | Key Parameters |
|-------------|---------|----------------|
| `getInput` | User input | prompt, validate |
| `apiCall` | HTTP requests | url, method, body, auth |
| `assign` | Variable assignment | value, source, path, expr |
| `llmCall` | LLM invocation | prompt, schema, temperature |
| `webSearch` | Web search | query, maxResults |
| `output` | Pipeline output | value, format |
| `log` | Debug logging | message, level |
