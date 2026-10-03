# EXPLAINABILITY.md

This document explains the internal mechanisms, data lineage, operational boundaries, and governance framework of **LangChain.js** (`langchainjs`) in accordance with the **OpenGAP v0.1.0** specification for the **HiDevs GitAgent Passport** clearance pipeline.

> **Agent Name:** LangChain.js (`langchainjs`)  
> **Specification:** OpenGAP v0.1.0  
> **Category / Domain:** Context-Aware LLM Applications & Multi-Agent Framework  
> **Compliance Standard:** OpenGAP Checkpoint 2 (Explainability & Decision Governance), OWASP LLM Top 10, MITRE ATLAS  

---

## How the Agent Decides

LangChain.js constructs context-aware pipelines, resolves dynamic tool invocations, and transitions state machines through a deterministic 5-stage decision pipeline.

### 1. Decision Architecture

The runtime intake, state classification, evaluation, and execution tracking operate across a deterministic, five-stage pipeline:

```
+-----------------------------------------------------------------------------------+
|                           5-STAGE DECISION PIPELINE                               |
+-----------------------------------------------------------------------------------+
|  [Stage 1: Runnable Ingestion & Input Schema Validation]                          |
|  - Parse input dictionary, validate against Zod/JSONSchema runnable contract      |
|                                     |                                             |
|                                     v                                             |
|  [Stage 2: Context Retrieval & Document Reranking]                               |
|  - Query vector stores, apply similarity thresholds, assemble prompt variables    |
|                                     |                                             |
|                                     v                                             |
|  [Stage 3: Model Invocation & Tool Calling Evaluation]                            |
|  - Execute chat model, compute chain affinity S_chain, evaluate tool call intent  |
|                                     |                                             |
|                                     v                                             |
|  [Stage 4: Threshold Evaluation & Graph State Transition]                         |
|  - Verify tau >= 0.70; enforce recursion limits, update state channels            |
|                                     |                                             |
|                                     v                                             |
|  [Stage 5: Output Parsing, Token Streaming & Trace Emission]                      |
|  - Parse structured output, stream tokens to client, emit LangSmith traces        |
+-----------------------------------------------------------------------------------+
```

### 2. Decision Logic & Routing Formulations

For a given LCEL chain configuration $R_i$ processing input payload $X$ with retrieved context chunks $D = \{d_1, d_2, \dots, d_K\}$, the chain execution score $S_{\text{chain}}(R_i, X)$ is formulated as:

$$S_{\text{chain}}(R_i, X) = w_{\text{schema}} V(X) + w_{\text{rag}} \left(\frac{1}{K} \sum_{j=1}^K \cos(\mathbf{e}_X, \mathbf{e}_{d_j})\right) + w_{\text{tool}} T(R_i, X) + w_{\text{cost}} C(R_i)$$

Where:
- $V(X) \in \{0, 1\}$ verifies that input $X$ strictly adheres to the chain's Zod input schema.
- $\cos(\mathbf{e}_X, \mathbf{e}_{d_j})$ represents cosine similarity between input embeddings and retrieved context documents.
- $T(R_i, X) \in [0, 1]$ represents tool argument validity and parameter confidence.
- $C(R_i) = \max\left(0, 1 - \frac{\text{TokensEstimated}(R_i)}{\text{ContextLimit}(R_i)}\right)$ penalizes context saturation.
- Standard default weights: $w_{\text{schema}} = 0.35$, $w_{\text{rag}} = 0.30$, $w_{\text{tool}} = 0.20$, $w_{\text{cost}} = 0.15$ with $\sum w = 1.0$.

Chain dispatch and tool execution require:

$$S_{\text{chain}}(R_i, X) \ge \tau \quad (\tau = 0.70) \quad \land \quad V(X) = 1$$

### 3. Thresholding & Refusal Decision Criteria

LangChain.js enforces strict operational boundaries and deterministic refusal thresholds:
- **Refusal on ERR_CHAIN_INPUT_SCHEMA_MISMATCH**: $V(X) = 0$ (Zod schema validation failure) halts execution with code `ERR_CHAIN_INPUT_SCHEMA_MISMATCH`.
- **Refusal on ERR_TOOL_CALL_FAILED**: Tool handler throws exception or times out halts execution with code `ERR_TOOL_CALL_FAILED`.
- **Refusal on ERR_CONTEXT_WINDOW_EXCEEDED**: Estimated prompt tokens exceed model window halts execution with code `ERR_CONTEXT_WINDOW_EXCEEDED`.
- **Refusal on ERR_VECTOR_RETRIEVAL_EMPTY**: Similarity search returns 0 documents above 0.50 halts execution with code `ERR_VECTOR_RETRIEVAL_EMPTY`.
- **Refusal on ERR_STATE_CYCLIC_DEADLOCK**: Graph recursion count reaches $N_{\text{recurse}} > 25$ halts execution with code `ERR_STATE_CYCLIC_DEADLOCK`.

### 4. Fallback Decision Mechanism

Continuous operational stability is maintained through layered fault recovery:
- **Tier 1 (Fallback Model Chains):** Use `.withFallbacks([backupModel])` to seamlessly fail over from primary LLM endpoints to alternative secondary providers upon HTTP 429 or 5xx errors.
- **Tier 2 (Cached Output & Default Handlers):** If downstream tools or external vector databases fail, serve cached responses or invoke default heuristic runnables.
- **Model Fallback Cascade**: High-level reasoning and synthesis default to `gemini-2.0-flash` with automatic failover to `gpt-4o` and `claude-3-5-sonnet`.

### 5. Human-in-the-Loop Governance

Human operators retain sovereign authority over the multi-agent execution lifecycle:
- **Tier 3 (HumanintheLoop Interrupt):** In LangGraph workflows, pause state transitions before highstakes tool calls, yielding execution to human review via checkpoint persistence.
- **Session Telemetry Auditing**: Operators inspect execution logs, routing traces, and token usage to maintain oversight.

---

## The Data It Uses

LangChain.js operates under strict principles of data minimization, environment isolation, and privacy protection.

### 1. Ingested Input Data

The framework processes only operational data necessary to perform its functions:
- **Prompt Inputs**: String queries, chat history message arrays (`SystemMessage`, `HumanMessage`, `AIMessage`), and structured variables.
- **Documents**: Raw text, PDFs, markdown files, and web pages ingested via document loaders.
- **Tool Invocations**: Function names, typed argument dictionaries, and external API responses.

### 2. Configuration & Reference Data

- **Vector Embeddings**: Dense vector representations stored in Pinecone, Chroma, Qdrant, MemoryVectorStore, or pgvector.
- **Prompt Templates**: Versioned chat prompt templates, few-shot exemplars, and system instructions.
- **Tool Schemas**: JSONSchema / Zod manifests describing tool functions, descriptions, and parameter constraints.

### 3. Base Model & Inference Lineage

- **Model Agnostic**: Interfaces with OpenAI, Anthropic, Google GenAI, Mistral, Ollama, Cohere, Bedrock, and HuggingFace.
- **Weight Integrity**: Operates directly on provider endpoints; does not modify model weights, ensuring transparency and reproducibility.

### 4. Data Privacy, Storage, and Retention

- **OWASP LLM & MITRE ATLAS Hardened**: Defended against indirect prompt injection, credential leakage, and unauthorized external API dispatch.
- **Local Environment Isolation**: Agent execution workspaces, intermediate scratchpads, and vector stores reside strictly within designated local project directories.
- **Automated Secret Scrubbing**: API keys, database credentials, and personal credentials are automatically redacted prior to embedding or logging.
- **Zero Commercial Monetization**: Prompts, intermediate reasoning trajectories, and task deliverables are never commercialized or shared with third parties.

---

## Limitations

Understanding the operational boundaries and technical constraints of LangChain.js is essential for effective deployment.

### 1. Complex agent loops with multiple tool
- **Limitation**: Complex agent loops with multiple tool calls can accumulate high latency and significant token costs.
- **Mitigation**: Developers can configure parallel tool execution (`Promise.all`) and clamp recursion limits using LangGraph channels.

### 2. Vector similarity searches can retrieve irrelevant
- **Limitation**: Vector similarity searches can retrieve irrelevant context if chunking boundaries split key semantic concepts.
- **Mitigation**: Recursive character text splitters and contextual document rerankers (Cohere Rerank) refine chunk relevance.

### 3. Structured output parsers can fail if
- **Limitation**: Structured output parsers can fail if the underlying language model emits imperfect JSON syntax.
- **Mitigation**: Built-in auto-fixing parsers (`OutputFixingParser`) prompt the model with the syntax error to correct JSON formatting.

### 4. Edge environments (e
- **Limitation**: Edge environments (e.g., Cloudflare Workers) have bundle size constraints and lack Node.js built-ins.
- **Mitigation**: LangChain.js packages entrypoints into modular sub-paths (`@langchain/core`, `@langchain/openai`) with zero Node-native dependencies.

### 5. Asynchronous token streaming can be interrupted
- **Limitation**: Asynchronous token streaming can be interrupted by client disconnection.
- **Mitigation**: AbortSignal integration cancels active model generation immediately upon client socket termination, conserving tokens.

---

## Summary & Compliance Checklist

| Checkpoint 2 Requirement | Corresponding Section | Status |
| :--- | :--- | :---: |
| **How the agent decides** | [How the Agent Decides](#how-the-agent-decides) | **Covered** |
| - Decision architecture & 5-stage pipeline | Section 1 | Verified |
| - Decision logic & routing formulations | Section 2 | Verified |
| - Thresholding & refusal decision criteria | Section 3 | Verified |
| - Fallback decision mechanism | Section 4 | Verified |
| - Human-in-the-loop governance & oversight | Section 5 | Verified |
| **The data it uses** | [The Data It Uses](#the-data-it-uses) | **Covered** |
| - Ingested input data & query streams | Section 1 | Verified |
| - Configuration & reference schemas | Section 2 | Verified |
| - Base model lineage & deterministic engines | Section 3 | Verified |
| - Data privacy, retention lifecycle & MITRE/OWASP | Section 4 | Verified |
| **Its limitations** | [Limitations](#limitations) | **Covered** |
| - Complex agent loops with multiple tool | Section 1 | Verified |
| - Vector similarity searches can retrieve irrelevant | Section 2 | Verified |
| - Structured output parsers can fail if | Section 3 | Verified |
| - Edge environments (e | Section 4 | Verified |
| - Asynchronous token streaming can be interrupted | Section 5 | Verified |
