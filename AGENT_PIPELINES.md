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

---

## Non-Streaming Pipeline

**Handler:** `chat()` in `src/handlers.rs:199-322`

**Purpose:** Processes requests synchronously, waiting for complete responses from both AI models before returning a single JSON response to the client.

### Data Flow Diagram

```
┌────────────────────────────────────────────────────────────────┐
│                    INCOMING HTTP REQUEST                       │
├────────────────────────────────────────────────────────────────┤
│ Headers:                                                       │
│   - X-DeepSeek-API-Token: "sk-..."                           │
│   - X-Anthropic-API-Token: "sk-ant-..."                      │
│                                                                │
│ Body:                                                          │
│   {                                                            │
│     "system": "You are a helpful AI assistant...",           │
│     "messages": [                                             │
│       {"role": "user", "content": "Explain quantum physics"} │
│     ],                                                         │
│     "stream": false,                                          │
│     "verbose": false,                                         │
│     "deepseek_config": {...},                                │
│     "anthropic_config": {...}                                │
│   }                                                            │
└────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────┐
│              STEP 1: VALIDATE SYSTEM PROMPT                    │
│              (Line 204-207)                                    │
├────────────────────────────────────────────────────────────────┤
│ Function: request.validate_system_prompt()                    │
│                                                                │
│ Checks:                                                        │
│   ✓ System prompt not in both root AND messages              │
│   ✓ Returns true if valid, false if duplicate                │
│                                                                │
│ Error: Returns BAD_REQUEST if validation fails               │
└────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────┐
│              STEP 2: EXTRACT API TOKENS                        │
│              (Line 210)                                        │
├────────────────────────────────────────────────────────────────┤
│ Function: extract_api_tokens(&headers)                        │
│                                                                │
│ Extracts:                                                      │
│   - deepseek_token: "sk-..."                                  │
│   - anthropic_token: "sk-ant-..."                             │
│                                                                │
│ Returns: (deepseek_token, anthropic_token)                    │
└────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────┐
│              STEP 3: INITIALIZE CLIENTS                        │
│              (Line 213-214)                                    │
├────────────────────────────────────────────────────────────────┤
│ deepseek_client = DeepSeekClient::new(deepseek_token)        │
│ anthropic_client = AnthropicClient::new(anthropic_token)      │
└────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────┐
│         STEP 4: PREPARE MESSAGES WITH SYSTEM PROMPT            │
│              (Line 217)                                        │
├────────────────────────────────────────────────────────────────┤
│ Function: request.get_messages_with_system()                  │
│                                                                │
│ Output:                                                        │
│   messages = [                                                 │
│     {                                                          │
│       "role": "system",                                       │
│       "content": "You are a helpful AI assistant..."         │
│     },                                                         │
│     {                                                          │
│       "role": "user",                                         │
│       "content": "Explain quantum physics"                   │
│     }                                                          │
│   ]                                                            │
└────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────┐
│         STEP 5: CALL DEEPSEEK API (STAGE 1)                   │
│              (Line 220)                                        │
├────────────────────────────────────────────────────────────────┤
│ deepseek_client.chat(messages, &deepseek_config)             │
│                                                                │
│ Sends to DeepSeek:                                            │
│   - All messages (including system prompt)                    │
│   - Custom config (temperature, max_tokens, etc.)            │
│                                                                │
│ DeepSeek Response:                                            │
│   {                                                            │
│     "choices": [{                                             │
│       "message": {                                            │
│         "reasoning_content": "Let me think step by step...",│
│         "content": ""                                        │
│       }                                                        │
│     }],                                                        │
│     "usage": {                                                │
│       "prompt_tokens": 150,                                  │
│       "completion_tokens": 500,                              │
│       "completion_tokens_details": {                         │
│         "reasoning_tokens": 450                              │
│       },                                                      │
│       "prompt_tokens_details": {                             │
│         "cached_tokens": 100                                 │
│       },                                                      │
│       "total_tokens": 650                                    │
│     }                                                          │
│   }                                                            │
└────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────┐
│         STEP 6: EXTRACT REASONING CONTENT                      │
│              (Line 227-238)                                    │
├────────────────────────────────────────────────────────────────┤
│ Extract from: deepseek_response.choices[0].message            │
│                      .reasoning_content                        │
│                                                                │
│ Raw reasoning:                                                 │
│   "Let me think step by step about quantum physics:          │
│    1. Quantum mechanics describes behavior at atomic scale   │
│    2. Key principles include wave-particle duality...        │
│    [detailed reasoning chain]"                               │
│                                                                │
│ Wrap in thinking tags:                                        │
│   thinking_content = "<thinking>\n" +                         │
│                      reasoning_content +                      │
│                      "\n</thinking>"                          │
│                                                                │
│ Result:                                                        │
│   "<thinking>                                                 │
│    Let me think step by step about quantum physics:          │
│    [full reasoning content]                                   │
│    </thinking>"                                               │
└────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────┐
│    STEP 7: PREPARE MESSAGES FOR CLAUDE (STAGE 2)             │
│              (Line 241-245)                                    │
├────────────────────────────────────────────────────────────────┤
│ anthropic_messages = messages (from Step 4)                   │
│                                                                │
│ Add DeepSeek's reasoning as assistant message:               │
│   anthropic_messages.push({                                   │
│     "role": "assistant",                                      │
│     "content": thinking_content                              │
│   })                                                           │
│                                                                │
│ Result:                                                        │
│   [                                                            │
│     {"role": "system", "content": "You are..."},             │
│     {"role": "user", "content": "Explain quantum physics"},  │
│     {"role": "assistant", "content": "<thinking>..."}        │
│   ]                                                            │
└────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────┐
│         STEP 8: CALL ANTHROPIC API (STAGE 2)                  │
│              (Line 248-252)                                    │
├────────────────────────────────────────────────────────────────┤
│ anthropic_client.chat(                                        │
│   anthropic_messages,                                         │
│   system_prompt,                                              │
│   &anthropic_config                                           │
│ )                                                              │
│                                                                │
│ Claude receives:                                              │
│   - Original user question                                    │
│   - DeepSeek's complete reasoning chain                      │
│   - System prompt                                             │
│                                                                │
│ Claude Response:                                              │
│   {                                                            │
│     "model": "claude-3-5-sonnet-20241022",                   │
│     "content": [{                                             │
│       "type": "text",                                        │
│       "text": "# Quantum Physics Explained\n\n..."          │
│     }],                                                        │
│     "usage": {                                                │
│       "input_tokens": 1200,                                  │
│       "output_tokens": 800,                                  │
│       "cache_creation_input_tokens": 0,                      │
│       "cache_read_input_tokens": 500                         │
│     }                                                          │
│   }                                                            │
└────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────┐
│         STEP 9: CALCULATE USAGE COSTS                          │
│              (Line 259-274)                                    │
├────────────────────────────────────────────────────────────────┤
│ DeepSeek Cost Calculation:                                    │
│   - Cache hit cost: (cached_tokens / 1M) * cache_hit_price   │
│   - Cache miss cost: (input - cached / 1M) * miss_price      │
│   - Output cost: (output_tokens / 1M) * output_price         │
│   Total: $0.025                                               │
│                                                                │
│ Anthropic Cost Calculation:                                   │
│   - Input cost: (input_tokens / 1M) * input_price            │
│   - Output cost: (output_tokens / 1M) * output_price         │
│   - Cache write: (cache_write_tokens / 1M) * write_price    │
│   - Cache read: (cache_read_tokens / 1M) * read_price       │
│   Total: $0.018                                               │
│                                                                │
│ Combined Total: $0.043                                        │
└────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────┐
│         STEP 10: BUILD COMBINED RESPONSE                       │
│              (Line 277-319)                                    │
├────────────────────────────────────────────────────────────────┤
│ content = []                                                   │
│                                                                │
│ 1. Add thinking block first:                                  │
│    content.push(ContentBlock::text(thinking_content))        │
│                                                                │
│ 2. Add Claude's response blocks:                             │
│    content.extend(anthropic_response.content)                │
│                                                                │
│ 3. Build ApiResponse:                                         │
│    - created: timestamp                                       │
│    - content: [thinking + claude response]                   │
│    - deepseek_response: (if verbose=true)                    │
│    - anthropic_response: (if verbose=true)                   │
│    - combined_usage: {                                        │
│        total_cost: "$0.043",                                 │
│        deepseek_usage: {...},                                │
│        anthropic_usage: {...}                                │
│      }                                                         │
└────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────┐
│                   RETURN JSON RESPONSE                         │
├────────────────────────────────────────────────────────────────┤
│ {                                                              │
│   "created": "2025-11-14T10:30:00Z",                         │
│   "content": [                                                │
│     {                                                          │
│       "type": "text",                                         │
│       "text": "<thinking>DeepSeek reasoning...</thinking>"   │
│     },                                                         │
│     {                                                          │
│       "type": "text",                                         │
│       "text": "# Quantum Physics Explained\n\n..."          │
│     }                                                          │
│   ],                                                           │
│   "combined_usage": {                                         │
│     "total_cost": "$0.043",                                  │
│     "deepseek_usage": {...},                                 │
│     "anthropic_usage": {...}                                 │
│   }                                                            │
│ }                                                              │
└────────────────────────────────────────────────────────────────┘
```

### Data Transformations

**Input → Stage 1 (DeepSeek):**
```
[System Prompt] + [User Messages] → DeepSeek R1 → Reasoning Content
```

**Stage 1 → Stage 2 (Claude):**
```
[System Prompt] + [User Messages] + [Assistant: <thinking>Reasoning</thinking>] → Claude → Final Response
```

**Output:**
```
[Thinking Block] + [Claude Response] + [Usage Statistics]
```

