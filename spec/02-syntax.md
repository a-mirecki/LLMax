# LLMax Syntax Reference

## File Structure

A `.llmax` file consists of:
1. Shebang (version declaration)
2. Metadata block
3. Variables block (optional)
4. Pipeline definitions

### Basic Structure

```llmax
#!llmax/1.0

@name: "Pipeline Name"
@version: "1.0.0"
@description: "What this pipeline does"
@author: "author@example.com"

@vars {
  API_KEY: env("API_KEY")
  BASE_URL: "https://api.example.com"
}

---

@pipeline main {
  # Instructions go here
}
```

## Shebang

Every `.llmax` file MUST start with a shebang declaring the protocol version:

```llmax
#!llmax/1.0
```

## Metadata

Metadata directives start with `@` and provide information about the pipeline:

| Directive | Required | Description |
|-----------|----------|-------------|
| `@name` | Yes | Human-readable pipeline name |
| `@version` | No | Semantic version (default: "1.0.0") |
| `@description` | No | Pipeline description |
| `@author` | No | Author contact |
| `@requires` | No | Required capabilities |

### Example

```llmax
@name: "Data Processing Pipeline"
@version: "2.1.0"
@description: "Processes customer data and generates reports"
@author: "team@company.com"
@requires: {
  tools: [file_read, file_write, bash],
  web_access: true
}
```

## Variables

### Global Variables Block

Define variables available throughout the pipeline:

```llmax
@vars {
  # Environment variable reference
  API_KEY: env("API_KEY")

  # Static value
  BASE_URL: "https://api.example.com"

  # Numeric value
  MAX_RETRIES: 3

  # Object value
  DEFAULT_HEADERS: {
    "Content-Type": "application/json",
    "Accept": "application/json"
  }
}
```

### Variable Types

| Type | Syntax | Example |
|------|--------|---------|
| String | `"text"` or `'text'` | `"Hello World"` |
| Number | Digits | `42`, `3.14`, `-10` |
| Boolean | `true` / `false` | `true` |
| Null | `null` | `null` |
| Object | `{ key: value }` | `{ name: "John", age: 30 }` |
| Array | `[item, ...]` | `[1, 2, 3]` |
| Environment | `env("NAME")` | `env("API_KEY")` |
| Duration | Number + unit | `30s`, `5m`, `2h` |

### Variable References

Reference variables using curly braces:

```llmax
# Simple reference
"{variableName}"

# Nested path
"{response.data.items[0].name}"

# With default fallback
"{config.timeout ?? 30}"
```

## Instructions

Instructions follow this pattern:

```
instructionName(param1: value1, param2: value2) -> outputVariable
```

### Syntax Rules

1. **Instruction name**: lowercase, alphanumeric
2. **Parameters**: Named with `param: value` syntax
3. **Output**: Optional, uses `-> varName` syntax
4. **Multi-line**: Use indentation for readability

### Single Line

```llmax
getInput(prompt: "Enter name", validate: text) -> userName
```

### Multi-line

```llmax
apiCall(
  url: "{BASE_URL}/users/{userId}",
  method: GET,
  headers: {
    "Authorization": "Bearer {API_KEY}"
  },
  timeout: 30s
) -> userResponse
```

## Core Instructions

| Instruction | Description |
|-------------|-------------|
| `getInput()` | Prompt user for input with validation |
| `apiCall()` | Make HTTP API calls |
| `assign()` | Assign values or extract from JSON |
| `llmCall()` | Invoke LLM with prompt |
| `webSearch()` | Search the web |
| `output()` | Output results |
| `log()` | Debug logging |

See [05-instructions.md](./05-instructions.md) for detailed reference.

## Control Flow

### Conditionals

```llmax
@if ({status} == "success") {
  output(value: "Operation succeeded!")
} @elif ({status} == "pending") {
  output(value: "Still processing...")
} @else {
  output(value: "Operation failed")
}
```

### Comparison Operators

| Operator | Description |
|----------|-------------|
| `==` | Equal |
| `!=` | Not equal |
| `>` | Greater than |
| `<` | Less than |
| `>=` | Greater or equal |
| `<=` | Less or equal |
| `&&` | Logical AND |
| `\|\|` | Logical OR |
| `!` | Logical NOT |
| `contains()` | String/array contains |
| `matches` | Regex match |

### Loops

#### For-each Loop

```llmax
@foreach (item in {items}) {
  log(message: "Processing: {item.name}")
  apiCall(url: "{API}/process/{item.id}") -> result
}
```

#### Range Loop

```llmax
@for (i in range(1, 10)) {
  log(message: "Iteration {i}")
}
```

#### While Loop

```llmax
@while ({hasMore} == true) {
  apiCall(url: "{nextPageUrl}") -> page
  assign(source: {page}, path: "$.nextPage") -> nextPageUrl
  assign(expr: "{nextPageUrl} != null") -> hasMore
}
```

#### Loop Control

```llmax
@foreach (item in {items}) {
  @if ({item.skip}) {
    @continue
  }
  @if ({item.stop}) {
    @break
  }
  # Process item
}
```

### Parallel Execution

```llmax
@parallel {
  @branch fetchUsers {
    apiCall(url: "{API}/users") -> users
  }
  @branch fetchProducts {
    apiCall(url: "{API}/products") -> products
  }
  @branch fetchOrders {
    apiCall(url: "{API}/orders") -> orders
  }
}
# All results available: {users}, {products}, {orders}
```

### Switch/Case

```llmax
@switch ({action}) {
  @case "create" {
    apiCall(url: "{API}/create", method: POST) -> result
  }
  @case "update" {
    apiCall(url: "{API}/update", method: PUT) -> result
  }
  @case "delete" {
    apiCall(url: "{API}/delete", method: DELETE) -> result
  }
  @default {
    log(message: "Unknown action", level: warn)
  }
}
```

## Error Handling

### Try-Catch-Finally

```llmax
@try {
  apiCall(url: "{unstableApi}") -> response
} @catch (NetworkError) {
  log(message: "Network error: {$error.message}", level: error)
  assign(value: null) -> response
} @catch {
  log(message: "Unexpected error: {$error}", level: error)
  @throw
} @finally {
  log(message: "API call attempt completed")
}
```

### Built-in Error Types

| Error | Description |
|-------|-------------|
| `NetworkError` | Network/connectivity issues |
| `TimeoutError` | Operation timeout |
| `ValidationError` | Input validation failed |
| `AuthError` | Authentication failed |
| `NotFoundError` | Resource not found |
| `RateLimitError` | Rate limit exceeded |
| `ParseError` | JSON/data parsing error |
| `SchemaError` | Schema validation error |

### Assertions

```llmax
@assert ({response.status} == 200) "API returned error"
@assert ({items}.length > 0) "No items found"
```

## Comments

```llmax
# This is a single-line comment

@pipeline main {
  # Comments can appear anywhere
  getInput(prompt: "Name?") -> name  # Inline comment
}
```

## Section Separators

Use `---` to visually separate sections:

```llmax
@name: "My Pipeline"

---

@vars {
  KEY: env("KEY")
}

---

@pipeline main {
  # ...
}
```

## Pipelines and Functions

### Named Pipelines

```llmax
@pipeline main {
  # Entry point
}

@pipeline cleanup {
  # Can be called from main
}
```

### Custom Functions

```llmax
@function calculateTotal(items: array, taxRate: number) -> number {
  assign(value: 0) -> total
  @foreach (item in {items}) {
    assign(expr: "{total} + {item.price}") -> total
  }
  assign(expr: "{total} * (1 + {taxRate})") -> total
  @return {total}
}

@pipeline main {
  calculateTotal(items: {cartItems}, taxRate: 0.08) -> finalTotal
}
```

## Imports

```llmax
# Import entire module
@import "./utils.llmax" as utils

# Import specific functions
@import { formatDate, parseJson } from "./helpers.llmax"

# Usage
utils.formatDate(date: {timestamp}) -> formatted
```

## JSON Path Expressions

LLMax uses JSONPath for data extraction:

```llmax
# Basic path
"$.data.name"

# Array access
"$.items[0]"
"$.items[-1]"          # Last item
"$.items[0:3]"         # Slice

# Wildcard
"$.items[*].name"      # All names

# Filter
"$.items[?(@.status == 'active')]"
"$.items[?(@.price > 100)]"

# Recursive descent
"$..name"              # All 'name' fields at any depth
```

## Reserved Variables

| Variable | Description |
|----------|-------------|
| `$index` | Current loop index |
| `$item` | Current loop item |
| `$result` | Last instruction result |
| `$error` | Current error (in catch) |
| `$env` | Environment accessor |
| `$now` | Current timestamp |

## Whitespace and Formatting

- Indentation: 2 spaces recommended
- Line length: 120 characters soft limit
- Empty lines: Allowed for readability
- Trailing whitespace: Ignored

## Complete Example

```llmax
#!llmax/1.0

@name: "Customer Order Processor"
@version: "1.0.0"
@description: "Processes customer orders and sends notifications"

@vars {
  API_KEY: env("ORDER_API_KEY")
  API_BASE: "https://api.orders.com/v1"
  NOTIFICATION_EMAIL: "orders@company.com"
}

---

@pipeline main {
  # Get order ID from user
  getInput(
    prompt: "Enter order ID to process",
    validate: /^ORD-[0-9]+$/
  ) -> orderId

  # Fetch order details
  @try {
    apiCall(
      url: "{API_BASE}/orders/{orderId}",
      headers: { "Authorization": "Bearer {API_KEY}" },
      timeout: 30s
    ) -> order
  } @catch (NotFoundError) {
    output(value: "Order {orderId} not found", format: text)
    @return
  }

  # Process based on order status
  @switch ({order.status}) {
    @case "pending" {
      processOrder(order: {order}) -> result
    }
    @case "shipped" {
      output(value: "Order already shipped")
    }
    @default {
      output(value: "Cannot process order with status: {order.status}")
    }
  }
}

@function processOrder(order: object) -> object {
  # Validate inventory
  apiCall(
    url: "{API_BASE}/inventory/check",
    method: POST,
    body: { items: {order.items} }
  ) -> inventory

  @if ({inventory.available} == false) {
    @throw ValidationError("Insufficient inventory")
  }

  # Update order status
  apiCall(
    url: "{API_BASE}/orders/{order.id}",
    method: PATCH,
    body: { status: "processing" }
  ) -> updated

  # Generate confirmation
  llmCall(
    prompt: "Generate a friendly order confirmation for order {order.id} with total ${order.total}",
    maxTokens: 200
  ) -> confirmation

  output(value: {confirmation}, format: markdown)
  @return {updated}
}
```
