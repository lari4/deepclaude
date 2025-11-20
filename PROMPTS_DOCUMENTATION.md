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

---

## Prompt Validation and Processing

### System Prompt Validation

**Purpose:** Ensures that system prompts are not duplicated in both the root level and messages array, which would cause API errors.

**Location:** `src/models/request.rs:75-80`

**Validation Logic:**
```rust
pub fn validate_system_prompt(&self) -> bool {
    let system_in_messages = self.messages.iter().any(|msg| matches!(msg.role, Role::System));

    // Only invalid if system prompt is provided in both places
    !(self.system.is_some() && system_in_messages)
}
```

**Validation Rules:**
1. System prompt can be provided at root level (`system` field)
2. System prompt can be provided in messages array (message with `role: "system"`)
3. System prompt cannot be provided in both places simultaneously
4. System prompt is optional - can be omitted entirely

**Error Handling:** If validation fails, the API returns a `BAD_REQUEST` error with the message: "System prompt cannot be provided in both root and messages"

---

### System Prompt Positioning

**Purpose:** Ensures the system prompt is always the first message in the conversation, regardless of where it was provided in the request.

**Location:** `src/models/request.rs:90-105`

**Positioning Logic:**
```rust
pub fn get_messages_with_system(&self) -> Vec<Message> {
    let mut messages = Vec::new();

    // Add system message first
    if let Some(system) = &self.system {
        messages.push(Message {
            role: Role::System,
            content: system.clone(),
        });
    }

    // Add remaining messages
    messages.extend(
        self.messages
            .iter()
            .filter(|msg| !matches!(msg.role, Role::System))
            .cloned()
    );

    messages
}
```

**Positioning Rules:**
1. System prompt is extracted from root level or messages array
2. System prompt becomes the first message in the processed array
3. All other messages maintain their relative order
4. Any system messages in the messages array are filtered out to prevent duplication

---

### System Prompt Retrieval

**Purpose:** Retrieves the system prompt from either the root level or messages array for processing.

**Location:** `src/models/request.rs:115-122`

**Retrieval Logic:**
```rust
pub fn get_system_prompt(&self) -> Option<&str> {
    self.system.as_deref().or_else(|| {
        self.messages
            .iter()
            .find(|msg| matches!(msg.role, Role::System))
            .map(|msg| msg.content.as_str())
    })
}
```

**Retrieval Priority:**
1. First checks the root level `system` field
2. If not found, searches the messages array for a system role message
3. Returns `None` if no system prompt is found in either location

---

### Backend Handler Validation

**Purpose:** Validates system prompts before processing requests through the dual-stage pipeline.

**Location:** `src/handlers.rs`

**Validation Points:**

**Non-Streaming Requests (Lines 204-206):**
```rust
if !request.validate_system_prompt() {
    return Err(ApiError::bad_request("System prompt cannot be provided in both root and messages"));
}
```

**Streaming Requests (Lines 470+):**
```rust
// Same validation applied to streaming endpoints
if !request.validate_system_prompt() {
    return Err(ApiError::bad_request("System prompt cannot be provided in both root and messages"));
}
```

**Processing Flow:**
1. Request received from frontend
2. System prompt validation performed
3. If validation passes, messages with system prompt are retrieved
4. Messages are passed to DeepSeek API (Line 216-217)
5. DeepSeek reasoning is extracted
6. Messages + reasoning are passed to Anthropic API (Line 250 or 470)

---

## API Request Structure

### Complete Request Format

**Purpose:** Defines the complete structure for API requests to the DeepClaude backend, including all configurable parameters.

**Location:** `README.md:178`, `src/models/request.rs:14-29`

**Request Schema:**
```json
{
    "system": "Optional system prompt string",
    "messages": [
        {
            "role": "user",
            "content": "User message content"
        },
        {
            "role": "assistant",
            "content": "Assistant response content"
        }
    ],
    "stream": false,
    "verbose": false,
    "deepseek_config": {
        "headers": {
            "Custom-Header": "value"
        },
        "body": {
            "temperature": 1.0,
            "max_tokens": 8192
        }
    },
    "anthropic_config": {
        "headers": {
            "Custom-Header": "value"
        },
        "body": {
            "model": "claude-3-5-sonnet-20241022",
            "max_tokens": 8192,
            "temperature": 1.0
        }
    }
}
```

**Field Descriptions:**

**`system` (Optional String):**
- Default system prompt for the conversation
- Applied to both DeepSeek and Claude models
- Cannot be provided if a system message exists in the messages array
- If omitted, models use their default behavior

**`messages` (Required Array):**
- Array of conversation messages
- Each message has `role` ("system", "user", or "assistant") and `content` fields
- Maintains conversation history and context
- System messages in this array are filtered and repositioned to the beginning

**`stream` (Optional Boolean, Default: false):**
- Controls whether responses are streamed or returned as a single block
- When `true`, uses Server-Sent Events (SSE) for streaming
- When `false`, waits for complete response before returning

**`verbose` (Optional Boolean, Default: false):**
- Controls whether DeepSeek's reasoning (thinking) is included in the response
- When `true`, returns both thinking and final response
- When `false`, only returns final Claude response

**`deepseek_config` (Optional Object):**
- Custom configuration for DeepSeek API calls
- `headers`: Custom HTTP headers for DeepSeek requests
- `body`: Override default DeepSeek parameters (temperature, max_tokens, etc.)
- Merged with default configuration

**`anthropic_config` (Optional Object):**
- Custom configuration for Anthropic Claude API calls
- `headers`: Custom HTTP headers for Anthropic requests
- `body`: Override default Claude parameters (model, temperature, max_tokens, etc.)
- Merged with default configuration

---

### Request Data Structure (Rust)

**Purpose:** Defines the Rust data structures for parsing and validating API requests.

**Location:** `src/models/request.rs`

**ApiRequest Structure:**
```rust
#[derive(Debug, Clone, Deserialize, Serialize)]
pub struct ApiRequest {
    #[serde(default)]
    pub stream: bool,

    #[serde(default)]
    pub verbose: bool,

    pub system: Option<String>,
    pub messages: Vec<Message>,

    #[serde(default)]
    pub deepseek_config: ApiConfig,

    #[serde(default)]
    pub anthropic_config: ApiConfig,
}
```

**Message Structure:**
```rust
#[derive(Debug, Clone, Deserialize, Serialize)]
pub struct Message {
    pub role: Role,
    pub content: String,
}
```

**Role Enum:**
```rust
#[derive(Debug, Clone, Deserialize, Serialize, PartialEq)]
#[serde(rename_all = "lowercase")]
pub enum Role {
    System,
    User,
    Assistant,
}
```

**ApiConfig Structure:**
```rust
#[derive(Debug, Clone, Default, Deserialize, Serialize)]
pub struct ApiConfig {
    #[serde(default)]
    pub headers: HashMap<String, String>,

    #[serde(default)]
    pub body: serde_json::Value,
}
```

---

## Summary

### Prompt Categories

1. **System Prompts:** User-configurable instructions that define AI behavior
2. **Configuration Prompts:** Default parameters that control model behavior
3. **Validation Logic:** Rules ensuring prompts are correctly structured
4. **Request Structure:** Format for submitting prompts and messages to the API

### Key Principles

1. **Flexibility:** System prompts are optional and user-customizable
2. **Consistency:** Same system prompt applied to both DeepSeek and Claude for coherent responses
3. **Validation:** Strict validation prevents duplicate or malformed prompts
4. **Configuration:** Separate configuration for each model allows fine-tuned control
5. **Positioning:** System prompts always come first, regardless of input format

### Prompt Flow

```
User Input → Validation → System Prompt Positioning → DeepSeek Processing →
Reasoning Extraction → Claude Processing → Final Response
```

