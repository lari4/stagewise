# Stagewise Agent Pipelines Documentation

This document describes all the agent workflows and pipelines in the Stagewise application, including what prompts are used, what data flows between stages, and how different components interact.

## Table of Contents

1. [Main Agent Execution Flow](#1-main-agent-execution-flow)
2. [Message Streaming Pipeline](#2-message-streaming-pipeline)
3. [Tool Execution Pipeline](#3-tool-execution-pipeline)
4. [Message Conversion Pipeline](#4-message-conversion-pipeline)
5. [Chat Management Pipeline](#5-chat-management-pipeline)
6. [Error Handling Pipeline](#6-error-handling-pipeline)
7. [Undo System Pipeline](#7-undo-system-pipeline)

---

## 1. Main Agent Execution Flow

**Purpose:** The core pipeline that orchestrates the entire agent execution lifecycle, from user input to response generation.

**Entry Point:** `Agent.getInstance()` → `sendUserMessage` procedure

**Location:** `agent/client/src/Agent.ts`

### ASCII Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    USER SENDS MESSAGE                            │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│              Add Message to Chat State (Karton)                  │
│  - User message stored in chat history                           │
│  - Chat error cleared                                            │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│            Gather Context (Prompt Snippets)                      │
│  - Project path snippet                                          │
│  - Project info snippet (frameworks, packages, etc.)             │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Call callAgent()                               │
│  Parameters:                                                     │
│    - chatId: Current chat UUID                                   │
│    - history: All messages (UI format)                           │
│    - clientRuntime: File system access                           │
│    - promptSnippets: Dynamic context                             │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│          Check Recursion Depth (max 20 levels)                   │
│  - Increment depth counter                                       │
│  - Initialize undo stack if needed                               │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│         Generate Chat Title (First Message Only)                 │
│  - Model: gemini-2.5-flash                                       │
│  - Prompt: generateChatTitleSystemPrompt                         │
│  - Output: Short title (max 7 words)                             │
│  - Fallback: "New Chat - {timestamp}"                            │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│           Assemble System Prompt (XMLPrompts)                    │
│  Components:                                                     │
│    1. Core system prompt (1200+ lines)                           │
│       - Agent identity & capabilities                            │
│       - Behavior guidelines                                      │
│       - Coding guidelines                                        │
│       - Tool usage guidelines                                    │
│    2. Prompt snippets (injected dynamically)                     │
│       - <project-info>: Project structure                        │
│       - <project-path>: Working directory                        │
│  Output: SystemModelMessage                                      │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│        Convert Messages (UI → Model Format)                      │
│  Pipeline: uiMessagesToModelMessages()                           │
│  Transformations:                                                │
│    - User: Add browser metadata + selected elements              │
│    - Assistant: Clean tool outputs (remove diffs)                │
│    - Tool: Sanitize large binary data                            │
│  See "Message Conversion Pipeline" for details                   │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│          Stream Text Generation (Claude Sonnet 4)                │
│  Configuration:                                                  │
│    - Model: claude-sonnet-4-20250514                             │
│    - Temperature: 0.7                                            │
│    - Max output tokens: 10,000                                   │
│    - Extended thinking: 10,000 token budget                      │
│    - Tools: 7 file operation tools                               │
│  Messages:                                                       │
│    [SystemMessage, ...HistoryMessages, CurrentUserMessage]       │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│              Parse Streaming Response                            │
│  Function: parseUiStream()                                       │
│  Process:                                                        │
│    - Read stream message by message                              │
│    - Add timestamp metadata                                      │
│    - Update existing or add new to chat state                    │
│    - Track lastMessageId                                         │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│            Execute Tool Calls (if any)                           │
│  Function: processParallelToolCalls()                            │
│  Process:                                                        │
│    1. Categorize tools (client vs browser)                       │
│    2. Execute client tools in parallel                           │
│    3. Execute browser tools sequentially                         │
│    4. Attach results to messages                                 │
│  See "Tool Execution Pipeline" for details                       │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
                    ┌────┴────┐
                    │ Tools?  │
                    └────┬────┘
                         │
           ┌─────────────┴─────────────┐
           │                           │
          YES                          NO
           │                           │
           ▼                           ▼
┌──────────────────────┐   ┌──────────────────────┐
│  RECURSIVE CALL      │   │  COMPLETE            │
│  callAgent()         │   │  - Set agent idle    │
│  with tool results   │   │  - Reset depth       │
│  in history          │   │  - Clear pending ops │
└──────────────────────┘   └──────────────────────┘
           │
           └──────────────────┐
                              │
                              ▼
                    (Loop back to top)
```

### Key Data Transformations

1. **User Input → UI Message**
   - Input: Raw user text + optional file attachments
   - Output: `UIMessage` with `userMessage` role and metadata

2. **UI Messages → Model Messages**
   - Input: Array of `UIMessage` objects
   - Output: Array of `ModelMessage` objects (compatible with Claude API)
   - Enrichments: Browser metadata, selected elements, project context

3. **Streaming Response → UI Messages**
   - Input: `AsyncIterableStream` from Claude
   - Output: `UIMessage` objects with assistant responses and tool calls
   - Metadata: Timestamps, message IDs

4. **Tool Calls → Tool Results**
   - Input: Tool call requests from LLM
   - Output: Tool result objects with success/error status
   - Enrichments: File diffs, undo callbacks

### Configuration

- **Model:** Claude Sonnet 4 (`claude-sonnet-4-20250514`)
- **Temperature:** 0.7
- **Max Recursion:** 20 levels
- **Max Output Tokens:** 10,000
- **Extended Thinking Budget:** 10,000 tokens
- **Agent Timeout:** 3 minutes (default)

### Error Handling

The pipeline includes comprehensive error handling:
- **Authentication errors:** Retry up to 2 times with token refresh
- **Plan limits exceeded:** Emit event and show cooldown timer
- **Insufficient credits:** Emit event and stop execution
- **Abort errors:** Clean up and reset state
- **General errors:** Format and display to user

See "Error Handling Pipeline" for full details.

---
## 2. Message Streaming Pipeline

**Purpose:** Stream LLM responses in real-time and update the UI progressively.

**Entry Point:** `callAgent()` → `streamText()` → `parseUiStream()`

**Location:** `agent/client/src/Agent.ts`

### ASCII Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│           streamText() - Claude Sonnet 4 API Call                │
│  Input:                                                          │
│    - model: claude-sonnet-4-20250514                             │
│    - messages: [SystemMessage, ...History, CurrentUser]          │
│    - temperature: 0.7                                            │
│    - maxTokens: 10000                                            │
│    - tools: cliToolsWithoutExecute                               │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│              Start Streaming Response                            │
│  - AsyncIterableStream created                                   │
│  - Begins yielding message chunks                                │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│           parseUiStream() - For-Await Loop                       │
│  Process each streamed message chunk:                            │
│    1. Check if message already exists (by ID)                    │
│    2. Add metadata: createdAt timestamp                          │
│    3. Determine action: UPDATE or ADD                            │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
                  ┌──────┴──────┐
                  │ Exists?     │
                  └──────┬──────┘
                         │
           ┌─────────────┴─────────────┐
           │                           │
          YES                          NO
           │                           │
           ▼                           ▼
┌──────────────────────┐   ┌──────────────────────┐
│  UPDATE Message      │   │  ADD New Message     │
│  - Find by ID        │   │  - Call onNewMessage │
│  - Replace in place  │   │  - Track lastMsgId   │
│  - Update chat state │   │  - Add to chat       │
└──────────┬───────────┘   └──────────┬───────────┘
           │                           │
           └───────────┬───────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│                UI State Updated (Karton)                         │
│  - User sees streaming response in real-time                     │
│  - Tool calls appear as they're generated                        │
│  - Reasoning (thinking) blocks shown progressively               │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│              Stream Complete (onFinish)                          │
│  - Final message with complete response                          │
│  - Tool calls ready for execution                                │
│  - Pass to Tool Execution Pipeline                               │
└─────────────────────────────────────────────────────────────────┘
```

### Message Types Streamed

1. **Text Content:** Assistant's response text
2. **Reasoning Blocks:** Extended thinking (10,000 token budget)
3. **Tool Calls:** File operation requests
4. **Step Starts:** Intermediate thinking steps

### Data Flow

**Input:**
- System prompt (with dynamic context)
- Message history (cleaned, sanitized)
- Current user message (with browser/DOM context)

**Processing:**
- Streamed in chunks from Claude API
- Each chunk is a partial or complete message
- Messages are incrementally built and updated

**Output:**
- Updated chat state with streaming messages
- Final complete messages with tool calls
- Ready for tool execution phase

---
