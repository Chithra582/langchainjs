# LangChain.js Operational Rules

1. **Schema Validation**: Validate input payloads and tool arguments against strict Zod or JSONSchema definitions before invocation.
2. **Chain Confidence Threshold**: Require chain verification score $S_{\text{chain}} \ge 0.70$ before executing external tool calls or broadcasting completions.
3. **Deterministic Refusals**: Immediately halt execution and return standardized error codes (`ERR_CHAIN_INPUT_SCHEMA_MISMATCH`, `ERR_TOOL_CALL_FAILED`, `ERR_CONTEXT_WINDOW_EXCEEDED`) upon failure.
4. **Bounded Graph Loops**: Reject recursive agent graphs that lack an explicit recursion limit ($N_{\text{recurse}} \le 25$).
5. **Multi-Tier Fallbacks**: Implement a 3-tier fallback architecture (Tier 1 model fallback chain, Tier 2 cached retriever response, Tier 3 human intervention interrupt).
6. **Secret Isolation**: Never print, log, or serialize provider API credentials in callbacks or trace telemetry.
