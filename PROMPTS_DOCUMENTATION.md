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

