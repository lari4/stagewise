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
## 3. Tool Execution Pipeline

**Purpose:** Execute file operation tools requested by the LLM, handling both client-side and browser-side tools with proper parallelization.

**Entry Point:** `onFinish` callback → `processParallelToolCalls()`

**Location:** `agent/client/src/utils/process-parallel-tool-calls.ts`

### ASCII Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│            LLM Requests Tool Calls (onFinish)                    │
│  Input: TypedToolCall[]                                          │
│    [                                                             │
│      { name: 'readFileTool', input: {...}, id: '123' },          │
│      { name: 'grepSearchTool', input: {...}, id: '456' },        │
│      ...                                                         │
│    ]                                                             │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│         Phase 1: Categorize by Runtime                           │
│  Filter by tool.runtime property:                                │
│    - clientToolCalls: runtime='client' (file operations)         │
│    - browserToolCalls: runtime='browser' (DOM operations)        │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│         Phase 2: Execute Client Tools in Parallel                │
│  Promise.all(clientToolCalls.map(processToolCall))               │
│                                                                  │
│  For each tool:                                                  │
│    1. Create ToolCallContext                                     │
│    2. Validate tool has execute method                           │
│    3. Execute: tool.execute(input, context)                      │
│    4. Measure duration                                           │
│    5. Handle result/error                                        │
│    6. Call onToolCallComplete callback                           │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│         Phase 3: Execute Browser Tools Sequentially              │
│  for (browserToolCall of browserToolCalls) {                     │
│    await processBrowserToolCall(...)                             │
│  }                                                               │
│                                                                  │
│  Note: Currently returns "Not yet implemented"                   │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│         Phase 4: Collect Results                                 │
│  allResults = [...clientResults, ...browserResults]              │
│  Filter out null values                                          │
│  Return ToolCallProcessingResult[]                               │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│         Attach Results to Messages                               │
│  Function: attachToolOutputToMessage()                           │
│                                                                  │
│  For each result:                                                │
│    1. Find message by messageId                                  │
│    2. Find tool part by toolCallId                               │
│    3. Update tool part state:                                    │
│       - SUCCESS: state='output-available', attach output         │
│       - ERROR: state='output-error', attach error message        │
│    4. Update Karton chat state                                   │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│         Return Results to Main Pipeline                          │
│  - Tool results attached to history                              │
│  - Ready for recursive callAgent() call                          │
│  - LLM will process tool results in next iteration               │
└─────────────────────────────────────────────────────────────────┘
```

### Tool Workflows by Type

#### Read File Tool Workflow

```
User/LLM Request
     │
     ▼
┌─────────────────────┐
│ readFileTool        │
│ Params:             │
│   target_file       │
│   start_line        │
│   end_line          │
└─────┬───────────────┘
      │
      ▼
┌─────────────────────────────────┐
│ 1. Validate params              │
│ 2. Resolve absolute path        │
│ 3. Check file exists            │
│ 4. Check file size (if entire)  │
│ 5. Read file content            │
│ 6. Return result with line info │
└─────┬───────────────────────────┘
      │
      ▼
Result: { content, totalLines }
→ Back to LLM in next iteration
```

#### Grep Search Tool Workflow

```
User/LLM Request
     │
     ▼
┌─────────────────────┐
│ grepSearchTool      │
│ Params:             │
│   query (regex)     │
│   file_pattern      │
│   case_sensitive    │
└─────┬───────────────┘
      │
      ▼
┌──────────────────────────────────┐
│ 1. Validate params               │
│ 2. Build exclude patterns        │
│ 3. Execute ripgrep               │
│ 4. Collect matches (max 100)     │
│ 5. Check if truncated            │
│ 6. Format results                │
└─────┬────────────────────────────┘
      │
      ▼
Result: { matches[], totalMatches, filesSearched, truncated }
→ LLM analyzes matches
```

#### Multi-Edit Tool Workflow

```
User/LLM Request
     │
     ▼
┌──────────────────────┐
│ multiEditTool        │
│ Params:              │
│   file_path          │
│   edits[]            │
│     old_string       │
│     new_string       │
│     replace_all      │
└─────┬────────────────┘
      │
      ▼
┌───────────────────────────────────┐
│ 1. Validate file exists           │
│ 2. Read current content           │
│ 3. Apply edits sequentially       │
│ 4. Write modified content         │
│ 5. Create undo callback           │
│ 6. Generate diff                  │
└─────┬─────────────────────────────┘
      │
      ▼
Result: { editsApplied, diff }
+ undoExecute callback stored
→ LLM continues with next step
```

### Tool Call Context Structure

```typescript
{
  tool: Tool,              // Tool definition with execute method
  toolName: string,        // e.g., "readFileTool"
  toolCallId: string,      // Unique ID for this invocation
  input: any,              // Tool parameters
  history: History,        // Full message history
  onToolCallComplete?: (result) => void  // Callback for progress updates
}
```

### Tool Result Structure

```typescript
{
  success: boolean,
  toolCallId: string,
  duration: number,       // Execution time in ms
  error?: {
    type: 'error' | 'user_interaction_required',
    message: string
  },
  result?: ToolResult     // From tool.execute()
}
```

### Available Tools (7 total)

1. **readFileTool:** Read file contents with line-by-line control
2. **overwriteFileTool:** Overwrite entire file or create new
3. **listFilesTool:** List files/directories with filtering
4. **grepSearchTool:** Fast regex searches using ripgrep
5. **globTool:** Find files matching glob patterns
6. **multiEditTool:** Multiple find-replace edits in one file
7. **deleteFileTool:** Delete file with undo capability

Each tool execution:
- Returns a `ToolResult` object
- May include `diff` for file changes
- May include `undoExecute` callback
- Includes success/error status

### Tool Part States

- `input-available`: Tool call awaiting execution
- `input-streaming`: Tool call being prepared
- `output-available`: Tool executed successfully
- `output-error`: Tool execution failed

---
## 4. Message Conversion Pipeline

**Purpose:** Convert UI messages (Karton format) to Model messages (Claude API format) with proper context enrichment and sanitization.

**Entry Point:** `callAgent()` → `uiMessagesToModelMessages()`

**Location:** `agent/client/src/utils/ui-messages-to-model-messages.ts`

### ASCII Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│              UI Messages (Karton Format)                         │
│  Input: ChatMessage[]                                            │
│    [                                                             │
│      { role: 'user', content: [...], metadata: {...} },          │
│      { role: 'assistant', content: [...], toolCalls: [...] },    │
│      ...                                                         │
│    ]                                                             │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│          For Each Message: Route by Role                         │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
              ┌──────────┴──────────┐
              │   Message Role?     │
              └──────────┬──────────┘
                         │
      ┌──────────────────┼──────────────────┐
      │                  │                  │
     USER            ASSISTANT           OTHER
      │                  │                  │
      ▼                  ▼                  ▼
┌──────────┐    ┌──────────────┐    ┌──────────┐
│ USER     │    │ ASSISTANT    │    │ OTHER    │
│ PIPELINE │    │ PIPELINE     │    │ PIPELINE │
└────┬─────┘    └─────┬────────┘    └────┬─────┘
     │                │                  │
     │                │                  │
     ▼                ▼                  ▼
```

### User Message Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│              User Message Processing                             │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│    Call: prompts.getUserMessagePrompt(config)                    │
│    Input: UserMessagePromptConfig                                │
│      { userMessage: ChatMessage }                                │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│    Step 1: Convert UI Parts to Model Content                     │
│    - Text parts → { type: 'text', text: '...' }                  │
│    - File parts → { type: 'document', ... }                      │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│    Step 2: Enrich with Browser Metadata (if available)           │
│    Call: browserMetadataToContextSnippet()                       │
│    Prompt Used: browser-metadata.ts                              │
│    Output:                                                       │
│      <browser-metadata>                                          │
│        <current-url>...</current-url>                            │
│        <viewport-resolution>...</viewport-resolution>            │
│        <user-agent>...</user-agent>                              │
│        <locale>...</locale>                                      │
│        ...                                                       │
│      </browser-metadata>                                         │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│    Step 3: Enrich with Selected Elements (if available)          │
│    Call: htmlElementToContextSnippet()                           │
│    Prompt Used: html-elements.ts                                 │
│    Output:                                                       │
│      <dom-elements>                                              │
│        <html-element type="div" selector=".class" xpath="...">   │
│          <!-- Element HTML structure -->                         │
│        </html-element>                                           │
│      </dom-elements>                                             │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│    Step 4: Build Final User Message                              │
│    UserModelMessage {                                            │
│      role: 'user',                                               │
│      content: [                                                  │
│        { type: 'text', text: userMessage },                      │
│        { type: 'text', text: browserMetadataXml },               │
│        { type: 'text', text: selectedElementsXml }               │
│      ]                                                           │
│    }                                                             │
└─────────────────────────────────────────────────────────────────┘
```

### Assistant Message Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│              Assistant Message Processing                        │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│    Check: Does message contain only reasoning parts?             │
│    - Look for 'reasoning', 'step-start' content                  │
│    - No text or tool calls                                       │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
                  ┌──────┴──────┐
                  │ Only        │
                  │ Reasoning?  │
                  └──────┬──────┘
                         │
           ┌─────────────┴─────────────┐
           │                           │
          YES                          NO
           │                           │
           ▼                           ▼
    ┌──────────┐              ┌────────────────┐
    │   SKIP   │              │  PROCESS       │
    │ Message  │              │  Message       │
    └──────────┘              └────────┬───────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────┐
│    Clone Message & Clean Tool Outputs                            │
│    For each tool call result:                                    │
│      - Remove 'diff' field (saves tokens)                        │
│      - Remove 'undoExecute' callback (not serializable)          │
│                                                                  │
│    Why? Binary diff data wastes tokens and isn't needed by LLM   │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│    Convert to Model Format                                       │
│    Call: convertToModelMessages([cleanedMessage])                │
│    Output: AssistantModelMessage                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Data Enrichment Details

**Browser Metadata Context:**
- Current URL and page title
- Zoom level
- Viewport resolution (width x height)
- Device pixel ratio
- User agent string
- Locale

**DOM Elements Context:**
- Element type (tag name)
- CSS selector (ID or class-based)
- XPath (for element identification)
- All element attributes
- Text content (truncated if > 10,000 chars)

**Tool Output Sanitization:**
- Removes: `diff` field (binary/large data)
- Removes: `undoExecute` callback (not serializable)
- Keeps: Success status, message, core result data
- Purpose: Save tokens and prevent API errors

### Prompts Used in Conversion

1. **System Prompt:** `getSystemPrompt()` from `agent/prompts/src/promts-xml/system.ts`
   - Injected once at the start of each agent call
   - Contains core agent instructions + dynamic project context

2. **User Message Prompt:** `getUserMessagePrompt()` from `agent/prompts/src/promts-xml/user.ts`
   - Called for each user message
   - Enriches with browser and DOM context

3. **Browser Metadata:** `browserMetadataToContextSnippet()` from `agent/prompts/src/promts-xml/browser-metadata.ts`
   - Appended to user messages when metadata available

4. **DOM Elements:** `htmlElementToContextSnippet()` from `agent/prompts/src/promts-xml/html-elements.ts`
   - Appended to user messages when elements selected

### Data Flow Summary

```
Karton UI Messages
  │
  ├─> User Messages
  │     └─> + Browser metadata
  │     └─> + Selected elements
  │     └─> UserModelMessage
  │
  ├─> Assistant Messages
  │     └─> Filter reasoning-only
  │     └─> Clean tool outputs
  │     └─> AssistantModelMessage
  │
  └─> Other Messages
        └─> Direct conversion
        └─> ModelMessage

All Messages Combined
  │
  └─> Sent to Claude API
```

---
