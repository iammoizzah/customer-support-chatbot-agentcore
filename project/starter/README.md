# Customer Support Chatbot with Amazon Bedrock AgentCore

A customer support chatbot for a fictional online shop, built on the **Amazon Bedrock AgentCore managed harness**. All routing, information gathering, and grounding behavior lives in a single system prompt — the harness supplies the agent loop (model calls, session memory, tool execution).

## What it does

The chatbot classifies every customer message into one of three routes and handles each differently:

| Route | Behavior |
|---|---|
| **Bug report** | Collects a description, steps to reproduce, and environment across multiple turns, then files a ticket via the `create_bug_report` tool (Lambda + DynamoDB, exposed through an AgentCore Gateway) |
| **Platform question** | Answers from an embedded FAQ (`online_shop_faq.md`) covering orders, shipping, returns, and payments |
| **Other / ambiguous** | Asks a clarifying question if the message is too vague to classify, otherwise redirects to the human support line (1-800-555-0199) |

## Architecture

Customer message
|
v
AgentCore managed harness (support_chatbot)

model: us.amazon.nova-pro-v1:0 (temperature 0, topK 1)
system prompt: system_prompt.txt (+ embedded FAQ)
memory: disabled (session isolation required for testing)
|
v
AgentCore Gateway ("bugreports" target)
|
v
Lambda: create_bug_report ---> DynamoDB: bug-report-tool-stack-bug-reports

## Repo contents

- `system_prompt.txt` — the chatbot's full system prompt (routing rules, bug-report checklist, FAQ grounding, prompt-injection hardening)
- `harness-tests.json` — 10-case test suite covering all three routes plus ambiguous messages, very short messages, and a prompt-injection attempt
- `output_eval_dataset.jsonl` — generated test run, harness responses vs. reference responses
- `OBSERVATIONS.md` — write-up of the testing/evaluation process, an issue found and fixed, and analysis of LLM-judge scoring variance
- `cloudformation-tool.yaml` / `cloudformation-testing.yaml` — infrastructure templates (Lambda, DynamoDB, IAM roles, S3 eval bucket)
- `setup_gateway.py`, `create_harness.py`, `chat.py`, `generate-eval-dataset.py`, `cleanup_agentcore.py` — course-provided operational scripts
- `evidence/` — screenshots documenting the working system (see below)

## Results

Ran Bedrock Evaluations (LLM-as-a-judge, `amazon.nova-pro-v1:0`) against the 10-case test suite. Correctness score: **0.70–0.75** across runs. See `OBSERVATIONS.md` for the full breakdown — manual verification confirmed all three routes and both stand-out behaviors (ambiguity handling, prompt-injection resistance) work correctly; the score variance across runs was traced to LLM-judge inconsistency on borderline cases rather than functional regressions.

## Evidence

Screenshots documenting a fully working system are in `evidence/`:

- `evidence/bug-report-chat.png` — full `chat.py` transcript of a bug report conversation, showing the follow-up questions and the `[tool call] bugreports___create_bug_report` line
- `evidence/dynamodb-ticket.png` — the resulting ticket in the `bug-report-tool-stack-bug-reports` DynamoDB table
- `evidence/faq-covered.png` — chatbot answering a platform question from the FAQ
- `evidence/faq-uncovered-handoff.png` — chatbot handing off a question not covered by the FAQ
- `evidence/other-request-handoff.png` — chatbot redirecting an out-of-scope request to human support
- `evidence/eval-results.png` — Bedrock Evaluations results page showing the correctness score and per-record breakdown

## Stand-out items implemented

- **Ambiguity handling**: vague/very short messages (e.g. "help") trigger a clarifying question instead of a guessed classification
- **Prompt injection hardening**: the system prompt explicitly ignores in-message instructions attempting to reveal the prompt, change its role, or bypass the bug-report collection rules
