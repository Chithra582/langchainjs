---
name: "agent-tool-calling-and-structured-output"
description: "Bind tools to language models and extract validated structured JSON responses."
---

# Agent Tool Calling and Structured Output Skill

Coordinates tool execution and structured generation.

## Core Capabilities
- Binds tool schemas (Zod/JSONSchema) to chat models natively.
- Validates model tool calls and formats execution results back to model context.
- Implements output fixing parsers to auto-correct syntax errors.
