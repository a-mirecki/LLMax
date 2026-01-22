# LLMax Protocol Specification

This directory contains the official specification for the LLMax Protocol - an LLM-agnostic language for building and executing agentic pipelines.

## Overview

LLMax is a domain-specific language (DSL) designed to be:
- **Human-readable**: Easy to write and understand
- **LLM-executable**: Any capable LLM can interpret and execute `.llmax` files
- **Visual-first**: Designed to work with node-based visual editors
- **Extensible**: Support for custom functions, plugins, and imports

## Specification Documents

| Document                                   | Description |
|--------------------------------------------|-------------|
| [01-overview.md](./01-overview.md)         | Introduction, goals, and design principles |
| [02-syntax.md](./02-syntax.md)             | Complete syntax reference |
| [03-instructions.md](./05-instructions.md) | Core instruction reference |

## Quick Example

```llmax
#!llmax/1.0

@name: "Hello World"
@version: "1.0.0"

@pipeline main {
  getInput(prompt: "What's your name?", validate: text) -> name

  llmCall(
    prompt: "Write a friendly greeting for {name}"
  ) -> greeting

  output(value: {greeting})
}
```

## File Extension

LLMax files use the `.llmax` extension.

## Version

Current specification version: **1.0**

## License

This specification is licensed under Apache 2.0. See [LICENSE](./LICENSE) for details.
