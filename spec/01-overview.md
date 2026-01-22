# LLMax Protocol Overview

## Introduction

LLMax is a protocol for defining agentic pipelines that can be executed by any capable Large Language Model (LLM). It provides a human-readable, text-based format that bridges the gap between visual workflow builders and LLM execution.

## Goals

### 1. LLM Agnostic
LLMax files should be executable by any LLM that supports:
- Tool/function calling
- Basic reasoning
- Following structured instructions

This includes Claude Code, GitHub Copilot, ChatGPT, and others.

### 2. Human Readable
The syntax prioritizes readability. A developer should be able to:
- Understand a pipeline at a glance
- Write pipelines by hand if needed
- Debug issues by reading the file

### 3. Visual Editor Compatible
LLMax is designed to work seamlessly with node-based visual editors:
- Each instruction maps to a visual node
- Connections between nodes map to variable references
- The visual representation and text format are bidirectionally convertible

### 4. Extensible
The protocol supports:
- Custom functions and reusable components
- Plugin system for additional capabilities
- Import/export for modular pipelines

## Design Principles

### Line-by-Line Instructions
Each instruction exists on its own line (or spans multiple lines with clear structure). This makes it easy for LLMs to parse and execute step-by-step.

### Explicit Data Flow
Variables are explicitly declared and passed between instructions. There's no hidden state - all data flow is visible.

### Fail-Safe Defaults
- Instructions have sensible defaults
- Error handling is built-in
- Graceful degradation when capabilities are missing

### Security First
- Credentials are never embedded in files
- Secrets are referenced by name and resolved at runtime
- Sensitive data is never logged or exposed

## Architecture

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  Visual Editor  │────▶│  .llmax File    │────▶│  LLM Executor   │
│   (Dashboard)   │◀────│   (Protocol)    │     │ (Claude/GPT/...)│
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

1. **Visual Editor**: Users create pipelines by connecting nodes
2. **.llmax File**: The visual representation is serialized to a text file
3. **LLM Executor**: Any capable LLM reads and executes the file

## Use Cases

### Automation Pipelines
- Data processing workflows
- Report generation
- Content creation pipelines

### AI Agents
- Research assistants
- Customer support bots
- Code generation agents

### Integration Workflows
- API orchestration
- Data transformation
- Multi-service coordination

## Conformance Levels

LLM implementations can conform at different levels:

### Level 1: Basic
- Parse and validate .llmax files
- Execute sequential instructions
- Basic variable substitution
- Error reporting

### Level 2: Standard
- All Level 1 features
- Control flow (conditionals, loops)
- Parallel execution
- Retry logic

### Level 3: Advanced
- All Level 2 features
- State checkpointing
- Circuit breakers
- Full observability

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025-01 | Initial specification |
