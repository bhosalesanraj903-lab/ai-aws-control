# Agent.md — Agentic AWS Resource Management

Quick reference for understanding the application before making changes.

---

## What This Is

A conversational AI agent that manages AWS infrastructure through natural language. Users type requests like "list my S3 buckets" or "check Lambda errors", and Claude (claude-sonnet-4-6) interprets the intent, calls the appropriate AWS APIs, and returns human-readable responses.

**Stack:** Streamlit frontend → FastAPI backend → Claude AI agent → AWS Cloud Control API / CloudWatch

---

## File Map

| File | Role |
|------|------|
| [main.py](main.py) | FastAPI server — two endpoints: `GET /` (health), `POST /chat` |
| [aws_agent.py](aws_agent.py) | Core agent logic — Claude API calls, tool routing, AWS execution |
| [aws_config.py](aws_config.py) | AWS credentials loading and boto3 session setup |
| [chat.py](chat.py) | Streamlit frontend — chat UI and session state |
| [processor.py](processor.py) | Thin bridge: instantiates `AWSAgenticAgent` and calls `process_request()` |
| [prompt.py](prompt.py) | `SYSTEM_PROMPT` (agent behavior) and `SUMMARY_PROMPT` (human-readable formatting) |
| [tools.json](tools.json) | JSON schema definitions for the two tools Claude can call |
| [requirements.txt](requirements.txt) | Python dependencies |

---

## Request Flow

```
User types in chat.py
  → POST http://localhost:8000/chat  (message + history)
    → main.py calls processor.py
      → AWSAgenticAgent.process_request()
        → Claude API call #1 (with tools)
          → Tool result: _execute_aws_operation() or _query_cloudwatch_logs()
        → Claude API call #2: _generate_summary() — formats result for humans
          → response + updated_history returned to Streamlit
```

---

## Core Class: `AWSAgenticAgent` ([aws_agent.py](aws_agent.py))

| Method | What it does |
|--------|-------------|
| `__init__()` | Initializes Anthropic client + AWSConfig; validates AWS connection |
| `process_request(message, history)` | Main entry point — orchestrates the full agentic loop |
| `_execute_aws_operation(op, type, id, props)` | Calls AWS Cloud Control API; returns response or request token for async ops |
| `_query_cloudwatch_logs(function_name, hours, limit)` | Fetches Lambda ERROR logs from CloudWatch (default: last 1h, max 10 entries) |
| `_generate_summary(result)` | Second Claude call to convert raw JSON/tool output into natural language |
| `_format_tools()` | Loads tool schemas from tools.json |
| `_build_system_prompt()` | Returns the SYSTEM_PROMPT from prompt.py |

---

## Tools Claude Can Call

Defined in [tools.json](tools.json):

### `aws_cloud_control`
- **Purpose:** CRUD + list operations on any AWS resource
- **Key params:** `operation` (create/read/update/delete/list), `resource_type` (e.g. `AWS::S3::Bucket`), `resource_id` (optional), `properties` (JSON, for create/update)
- **Backed by:** `boto3` `cloudcontrol` client

### `cloudwatch_logs`
- **Purpose:** Query Lambda function error logs
- **Key params:** `function_name`, `hours` (default 1), `limit` (default 10)
- **Backed by:** `boto3` `logs` client — filters for `"ERROR"` entries

---

## Environment Setup

Requires a `.env` file (not committed):

```
ANTHROPIC_API_KEY=...
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
AWS_DEFAULT_REGION=us-east-1   # optional, defaults to us-east-1
```

AWS identity is validated at startup via `sts.get_caller_identity()`.

---

## Running Locally

```bash
# Backend
uvicorn main:app --reload --port 8000

# Frontend (separate terminal)
streamlit run chat.py --server.port 8501
```

---

## Key Design Decisions

- **Two Claude API calls per request:** First call selects and runs the tool; second call (`_generate_summary`) converts raw output into human language. Adds latency (~1–2s) but significantly improves response quality.
- **Full history on every request:** Entire conversation history is sent to Claude each call — enables multi-turn context but grows in size over long sessions.
- **Cloud Control API as the universal layer:** Handles most AWS services (S3, EC2, RDS, Lambda, etc.) via a single boto3 client rather than per-service clients.
- **Async tokens:** Cloud Control API is async for write operations; `_execute_aws_operation` returns the request token when the operation is still in progress.
- **MCP dependency installed but unused:** `mcp>=1.0.0` is in requirements.txt; the project name suggests MCP integration is a planned direction.

---

## AWS IAM Permissions Required

```json
{
  "Actions": [
    "cloudcontrol:*",
    "s3:*",
    "logs:DescribeLogGroups",
    "logs:DescribeLogStreams",
    "logs:GetLogEvents",
    "logs:FilterLogEvents"
  ]
}
```

---

## Known Limitations / Future Work (from README)

- CORS is wide-open (`*`) — needs restriction for production
- Credentials use static keys (no IAM role / SSO support yet)
- No streaming responses — full response arrives at once
- `mcp` package installed but not yet wired up
- Roadmap: ECS/DynamoDB support, cost management, multi-account, real-time monitoring dashboard
