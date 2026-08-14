# Sample Error Log Report

**Source:** `error_log` Data Table
**Rows:** 9
**Period:** Testing and live execution across Chat Flow executions 102–140

Unedited export from the `error_log` Data Table. All 9 rows are real test executions.

---

## Log

| execution_id | workflow | node | error_message |
|---|---|---|---|
| 102 | Virtural Coffee Chat Bot (Chat Flow) | Chat Bot Conversation Logging | Data table with ID '000' could not be found in this project |
| 103 | Virtural Coffee Chat Bot (Chat Flow) | Chat Bot Conversation Logging | Data table with ID '000' could not be found in this project |
| 113 | Virtural Coffee Chat Bot (Chat Flow) | Chat Bot Conversation Logging | Data table with ID '000' could not be found in this project |
| 116 | Virtural Coffee Chat Bot (Chat Flow) | RAG Agent | Bad request - please check your parameters |
| 123 | Virtural Coffee Chat Bot (Chat Flow) | Stop and Error | RAG Agent reported a tool failure |
| 126 | Virtural Coffee Chat Bot (Chat Flow) | Listen for Silent Errors | 🤖 Bzzt! I'm having a technical issue completing that right now. This error has been recorded and a notification has been sent so it can be looked into. If we are unable to move forward, please email John directly at info@curiousjohnadams.com. |
| 128 | Virtural Coffee Chat Bot (Chat Flow) | Listen for Silent Errors | A tool call failed during a chat reply — check execution 128 in n8n for the specific error. |
| 134 | Virtural Coffee Chat Bot (Chat Flow) | Listen for Silent Errors | A tool call failed during a chat reply — check execution 134 in n8n for the specific error. |
| 140 | Virtural Coffee Chat Bot (Chat Flow) | RAG Agent | Bad request - please check your parameters |

---

## What the node column shows

Three different capture mechanisms are visible in the `node` column — this is the error-handling architecture in action across real executions:

- **Chat Bot Conversation Logging** (execs 102, 103, 113) — the logging node's own error output. The `chat_log` Data Table ID was misconfigured during early testing. Caught by the logging node's error path.
- **RAG Agent** (execs 116, 140) — hard crashes. The agent itself failed (bad model parameter during testing). Caught by n8n's native error trigger on the Error Logging + Alert workflow.
- **Stop and Error** (exec 123) — an intermediate error-handling node used in an earlier build iteration, now removed. Caught an explicit tool failure signal from the agent.
- **Listen for Silent Errors** (execs 126, 128, 134) — tool calls that failed underneath the agent without crashing it. The agent absorbed the failure, returned the standard Bzzt phrase, and the `Listen for Silent Errors` IF node detected it and routed to the error system.
