---
name: claude-to-deerflow
description: "Use from an external shell or coding agent to interact with a separate running DeerFlow instance via HTTP. Do not use from an active DeerFlow conversation or subagent: use the built-in task tool for delegation there."
---

# DeerFlow Skill

Communicate with a running DeerFlow instance via its HTTP API. DeerFlow is an AI agent platform
built on LangGraph that orchestrates sub-agents for research, code execution, web browsing, and more.

## Scope

This is an **external-client** skill. It is for a shell or coding agent outside
the DeerFlow instance that it is calling.

Do **not** use it while you are already executing as a DeerFlow lead agent or
subagent. Calling DeerFlow from its own agent creates a nested thread and can
duplicate or interrupt delegation. For a user request in the current DeerFlow
conversation, delegate directly with the built-in `task` tool and select the
configured `researcher` subagent.

## Architecture

DeerFlow exposes two API surfaces behind an Nginx reverse proxy:

| Service        | Direct Port | Via Proxy                        | Purpose                          |
|----------------|-------------|----------------------------------|----------------------------------|
| Gateway API    | 8001        | `$DEERFLOW_GATEWAY_URL`          | REST endpoints and embedded agent runtime |
| LangGraph-compatible API | 8001 | `$DEERFLOW_LANGGRAPH_URL`       | Agent threads, runs, streaming   |

## Environment Variables

All URLs are configurable via environment variables. **Read these env vars before making any request.**

| Variable                | Default                                  | Description                        |
|-------------------------|------------------------------------------|------------------------------------|
| `DEERFLOW_URL`          | `http://localhost:2026`                  | Unified proxy base URL             |
| `DEERFLOW_GATEWAY_URL`  | `${DEERFLOW_URL}`                        | Gateway API base; overrides all defaults |
| `DEERFLOW_LANGGRAPH_URL`| `${DEERFLOW_URL}/api/langgraph`          | LangGraph API base; overrides all defaults |
| `DEER_FLOW_CHANNELS_GATEWAY_URL` | Used when the skill runs in Docker | Internal Gateway URL, normally `http://gateway:8001` |
| `DEER_FLOW_CHANNELS_LANGGRAPH_URL` | Used when the skill runs in Docker | Internal LangGraph URL, normally `http://gateway:8001/api` |
| `DEER_FLOW_INTERNAL_AUTH_TOKEN` | Optional for standalone calls | Gateway internal-auth token |

When making curl calls, always resolve the URL like this:

```bash
# Resolve base URLs from env (do this FIRST before any API call)
if [ -n "${DEERFLOW_URL:-}" ]; then
  DEFAULT_GATEWAY_URL="$DEERFLOW_URL"
  DEFAULT_LANGGRAPH_URL="$DEERFLOW_URL/api/langgraph"
else
  DEERFLOW_URL="http://localhost:2026"
  DEFAULT_GATEWAY_URL="${DEER_FLOW_CHANNELS_GATEWAY_URL:-$DEERFLOW_URL}"
  DEFAULT_LANGGRAPH_URL="${DEER_FLOW_CHANNELS_LANGGRAPH_URL:-$DEERFLOW_URL/api/langgraph}"
fi
DEERFLOW_GATEWAY_URL="${DEERFLOW_GATEWAY_URL:-$DEFAULT_GATEWAY_URL}"
DEERFLOW_LANGGRAPH_URL="${DEERFLOW_LANGGRAPH_URL:-$DEFAULT_LANGGRAPH_URL}"
```

## Available Operations

### 1. Health Check

Verify DeerFlow is running:

```bash
curl -s "$DEERFLOW_GATEWAY_URL/health"
```

### 2. Send a Message (Streaming)

This is the primary operation. It creates a thread and streams the agent's response.

**Step 1: Create a thread**

```bash
curl -s -X POST "$DEERFLOW_LANGGRAPH_URL/threads" \
  -H "Content-Type: application/json" \
  -d '{}'
```

Response: `{"thread_id": "<uuid>", ...}`

**Step 2: Stream a run**

```bash
curl -s -N -X POST "$DEERFLOW_LANGGRAPH_URL/threads/<thread_id>/runs/stream" \
  -H "Content-Type: application/json" \
  -d '{
    "assistant_id": "lead_agent",
    "input": {
      "messages": [
        {
          "type": "human",
          "content": [{"type": "text", "text": "YOUR MESSAGE HERE"}]
        }
      ]
    },
    "stream_mode": ["values", "messages-tuple"],
    "stream_subgraphs": true,
    "config": {
      "recursion_limit": 1000
    },
    "context": {
      "thinking_enabled": true,
      "is_plan_mode": true,
      "subagent_enabled": true,
      "thread_id": "<thread_id>"
    }
  }'
```

The response is an SSE stream. Each event has the format:
```
event: <event_type>
data: <json_data>
```

Key event types:
- `metadata` — run metadata including `run_id`
- `values` — full state snapshot with `messages` array
- `messages-tuple` — incremental message updates (AI text chunks, tool calls, tool results)
- `end` — stream is complete

**Context modes** (set via `context`):
- Flash mode: `thinking_enabled: false, is_plan_mode: false, subagent_enabled: false`
- Standard mode: `thinking_enabled: true, is_plan_mode: false, subagent_enabled: false`
- Pro mode: `thinking_enabled: true, is_plan_mode: true, subagent_enabled: false`
- Ultra mode (the helper script default): `thinking_enabled: true, is_plan_mode: true, subagent_enabled: true`

### 3. Continue a Conversation

To send follow-up messages, reuse the same `thread_id` from step 2 and POST another run
with the new message.

### 4. List Models

```bash
curl -s "$DEERFLOW_GATEWAY_URL/api/models"
```

Returns: `{"models": [{"name": "...", "provider": "...", ...}, ...]}`

### 5. List Skills

```bash
curl -s "$DEERFLOW_GATEWAY_URL/api/skills"
```

Returns: `{"skills": [{"name": "...", "enabled": true, ...}, ...]}`

### 6. Enable/Disable a Skill

```bash
curl -s -X PUT "$DEERFLOW_GATEWAY_URL/api/skills/<skill_name>" \
  -H "Content-Type: application/json" \
  -d '{"enabled": true}'
```

### 7. List Agents

```bash
curl -s "$DEERFLOW_GATEWAY_URL/api/agents"
```

Returns: `{"agents": [{"name": "...", ...}, ...]}`

### 8. Get Memory

```bash
curl -s "$DEERFLOW_GATEWAY_URL/api/memory"
```

Returns user context, facts, and conversation history summaries.

### 9. Upload Files to a Thread

```bash
curl -s -X POST "$DEERFLOW_GATEWAY_URL/api/threads/<thread_id>/uploads" \
  -F "files=@/path/to/file.pdf"
```

Supports PDF, PPTX, XLSX, DOCX — automatically converts to Markdown.

### 10. List Uploaded Files

```bash
curl -s "$DEERFLOW_GATEWAY_URL/api/threads/<thread_id>/uploads/list"
```

### 11. Get Thread History

```bash
curl -s "$DEERFLOW_LANGGRAPH_URL/threads/<thread_id>/history"
```

### 12. List Threads

```bash
curl -s -X POST "$DEERFLOW_LANGGRAPH_URL/threads/search" \
  -H "Content-Type: application/json" \
  -d '{"limit": 20, "sort_by": "updated_at", "sort_order": "desc"}'
```

## Usage Script

For sending messages and collecting the full response, use the helper script:

```bash
bash /path/to/skills/claude-to-deerflow/scripts/chat.sh "Your question here"
```

See `scripts/chat.sh` for the implementation. The script:
1. Checks health
2. Supplies the CSRF cookie/header pair required by Gateway POST routes
3. Creates a thread
4. Streams the run and collects the final AI response
5. Prints the result

## Parsing SSE Output

The stream returns SSE events. To extract the final AI response from a `values` event:
- Look for the last `event: values` block
- Parse its `data` JSON
- The `messages` array contains all messages; the last one with `type: "ai"` is the response
- The `content` field of that message is the AI's text reply

## Error Handling

- If health check fails, DeerFlow is not running. Inform the user they need to start it.
- If the stream returns an error event, extract and display the error message.
- Common issues: port not open, services still starting up, config errors.

## Tips

- For quick questions, use flash mode (fastest, no planning).
- For research tasks, use pro or ultra mode (enables planning and sub-agents).
- You can upload files first, then reference them in your message.
- Thread IDs persist — you can return to a conversation later.
