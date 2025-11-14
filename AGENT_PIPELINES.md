# Agent Pipelines Documentation

This document provides a comprehensive overview of all agent workflows and data pipelines in the DeepClaude application, describing how prompts, messages, and data flow through the dual-stage AI processing system.

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Non-Streaming Pipeline](#non-streaming-pipeline)
3. [Streaming Pipeline](#streaming-pipeline)
4. [Authentication Pipeline](#authentication-pipeline)
5. [Validation Pipeline](#validation-pipeline)
6. [Cost Calculation Pipeline](#cost-calculation-pipeline)
7. [Error Handling Pipeline](#error-handling-pipeline)

---

## Architecture Overview

### Dual-Stage Processing Model

DeepClaude uses a **dual-stage AI processing architecture** that combines two different AI models for enhanced reasoning and response quality:

**Stage 1 - DeepSeek R1 (Reasoning):**
- Performs deep Chain-of-Thought reasoning
- Generates detailed thinking process
- Outputs structured reasoning in `<think>` tags

**Stage 2 - Claude (Synthesis):**
- Receives DeepSeek's reasoning as context
- Synthesizes creative, well-formatted responses
- Leverages DeepSeek's reasoning for higher quality outputs

### High-Level Architecture

```
┌─────────────┐
│   Client    │
│  (Frontend) │
└──────┬──────┘
       │
       │ HTTP Request (JSON)
       │ - system prompt
       │ - messages
       │ - config options
       ▼
┌─────────────────────────────────────┐
│         Backend Handler             │
│  (src/handlers.rs)                  │
├─────────────────────────────────────┤
│  1. Validate system prompt          │
│  2. Extract API tokens              │
│  3. Route to stream/non-stream      │
└──────┬──────────────────────────────┘
       │
       ├─────────────┬─────────────┐
       │             │             │
       ▼             ▼             ▼
┌──────────┐  ┌──────────┐  ┌──────────┐
│ DeepSeek │  │  Extract │  │  Claude  │
│   API    │─▶│ Reasoning│─▶│   API    │
└──────────┘  └──────────┘  └──────────┘
   Stage 1        Stage 1.5     Stage 2

       │             │             │
       └─────────────┴─────────────┘
                    │
                    ▼
            ┌───────────────┐
            │   Response    │
            │   Builder     │
            └───────┬───────┘
                    │
                    ▼
            ┌───────────────┐
            │    Client     │
            │  (Frontend)   │
            └───────────────┘
```

### Key Components

**Location:** `src/handlers.rs`

**Main Handler:**
- `handle_chat()` - Routes to streaming or non-streaming based on request

**Processing Handlers:**
- `chat()` - Non-streaming request processor (Line 199)
- `chat_stream()` - Streaming request processor (Line 338)

**Support Functions:**
- `extract_api_tokens()` - Extracts authentication tokens (Line 49)
- `calculate_deepseek_cost()` - Calculates DeepSeek usage costs (Line 90)
- `calculate_anthropic_cost()` - Calculates Claude usage costs (Line 118)

