# AI Prompts Documentation

This document provides a comprehensive overview of all AI prompts used in the DeepClaude application, organized by theme and purpose.

## Table of Contents

1. [System Prompts](#system-prompts)
2. [Model Configuration Prompts](#model-configuration-prompts)
3. [Prompt Validation and Processing](#prompt-validation-and-processing)
4. [API Request Structure](#api-request-structure)

---

## System Prompts

### Default System Prompt

**Purpose:** This is the primary system prompt used throughout the application. It instructs the AI to be helpful, excel at reasoning, use Markdown formatting, and properly format code snippets with language identifiers.

**Location:**
- `frontend/components/chat.tsx:452`
- `frontend/components/settings.tsx:38,108`

**Usage:** This prompt is sent as the system message to both DeepSeek and Claude models to establish consistent behavior across the dual-stage reasoning pipeline.

**Prompt:**
```text
You are a helpful AI assistant who excels at reasoning and responds in Markdown format. For code snippets, you wrap them in Markdown codeblocks with it's language specified.
```

**Key Characteristics:**
- Establishes helpful assistant persona
- Emphasizes reasoning capability
- Requires Markdown formatting for all responses
- Requires code blocks with language specification

**Customization:** Users can modify this system prompt through the settings interface to customize AI behavior.

---

## Model Configuration Prompts

### DeepSeek R1 Configuration

**Purpose:** Defines default configuration parameters for the DeepSeek R1 reasoning model, which performs the first stage of processing (Chain of Thought reasoning).

**Location:** `src/clients/deepseek.rs`

**Default Model:**
```rust
// Line 69
"deepseek-reasoner"
```

**Default Temperature:**
```rust
// Line 247
1.0
```

**Temperature Purpose:** Controls randomness in responses. A value of 1.0 provides balanced creativity while maintaining logical reasoning chains.

**Default Max Tokens:**
```rust
// Line 246
8192
```

**Max Tokens Purpose:** Allows DeepSeek R1 to generate extensive reasoning chains without truncation, ensuring complete thought processes.

**Response Format Configuration:**
```rust
// Lines 248-250
{
    "type": "text"
}
```

**Response Format Purpose:** Specifies that DeepSeek should return plain text responses, which will be parsed for `<think>` tags containing reasoning content.

---

### Claude Configuration

**Purpose:** Defines default configuration parameters for Claude models, which perform the second stage of processing (creative synthesis based on DeepSeek's reasoning).

**Location:** `src/clients/anthropic.rs`

**Default Model:**
```rust
// Line 51
"claude-3-5-sonnet-20241022"
```

**Model Selection Purpose:** Claude 3.5 Sonnet provides optimal balance of intelligence, speed, and cost for synthesizing DeepSeek's reasoning into final responses.

**Default Max Tokens by Model:**
```rust
// Lines 268-276
match model_name {
    name if name.contains("opus") => 4096,
    _ => 8192,
}
```

**Max Tokens Purpose:**
- **Opus models:** Limited to 4096 tokens for cost optimization (Opus is more expensive)
- **Other models:** 8192 tokens for comprehensive responses

**Available Claude Models:**

**Location:** `frontend/components/chat.tsx:813-818`

```typescript
[
  "claude-3-5-sonnet-20241022",
  "claude-3-5-sonnet-latest",
  "claude-3-5-haiku-20241022",
  "claude-3-5-haiku-latest",
  "claude-3-opus-20240229",
  "claude-3-opus-latest"
]
```

**Model Tier Purposes:**
- **Sonnet models:** Balanced intelligence and speed (recommended for most tasks)
- **Haiku models:** Fastest responses for simple tasks
- **Opus models:** Highest intelligence for complex reasoning tasks

