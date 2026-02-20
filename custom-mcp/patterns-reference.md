# Agentic Tool Patterns -- Quick Reference

Condensed from the 54 Patterns for Agentic Tools. Apply these when building Arcade MCP tools.
Full catalog: https://arcade.dev/patterns/llm.txt

---

## Guiding Principle

> "Your agents are only as good as your tools."

Design tools for LLM comprehension, not human reading. When tools are well-designed, orchestration stays simple and agents behave predictably.

---

## 1. Tool Types

Classify every tool you build:

| Type | Description | Characteristics |
|------|-------------|-----------------|
| **Query Tool** | Read-only, retrieves data | Safe to retry, cacheable, parallelizable |
| **Command Tool** | Performs actions with side effects | May need confirmation for destructive ops |
| **Discovery Tool** | Reveals schema, capabilities, or available operations | `list_tables()`, `who_am_i()`, `get_capabilities()` |

---

## 2. Tool Interface Patterns

### Tool Description
- Write docstrings for LLM comprehension, not humans
- Include examples of valid inputs in the description
- Mention prerequisites ("Call `list_projects` first to get project IDs")

### Constrained Input
- Use `Enum` types instead of free-form strings
- Use `Annotated` with min/max ranges for numeric params
- Validate format with clear error messages

### Smart Defaults
- Provide sensible defaults for optional parameters
- Use context-aware defaults (current user, timezone)
- Document defaults in the `Annotated` description

### Natural Identifier
- Accept human-friendly names (email, username, display name)
- Resolve to system IDs internally
- On ambiguity, return candidates via `RetryableToolError`

### Parameter Coercion
- Accept flexible input formats, normalize internally
- Dates: ISO, relative ("yesterday"), natural language
- Enums: accept string values, cast to Enum in the tool body

### Performance Hint
- Note efficient usage in descriptions: "Prefer `thread_id` (faster) over `subject` search"
- Warn about expensive operations in parameter descriptions

---

## 3. Tool Discovery Patterns

### Dependency Hint
- Embed "call X before Y" guidance in tool descriptions
- Example: "Requires a `project_id`. Call `list_projects` first if you don't have one."

### Identity Anchor
- Provide a `who_am_i()` tool that returns user context (email, name, ID, permissions)
- Agents call this first to establish session context

---

## 4. Tool Composition Patterns

### Abstraction Ladder
Provide tools at multiple granularity levels:
- Low: `create_file(path, bytes)`
- Mid: `create_document(title, content)`
- High: `draft_report(topic)`

### Task Bundle
Combine multi-step operations into a single tool:
- `send_dm(username, message)` = resolve user + open channel + send
- Name after the task, not the steps
- Handle intermediate errors internally

### Batch Operation
- Accept arrays of items in a single call
- Return per-item results with status
- Handle partial failures (see Partial Success)

---

## 5. Tool Execution Patterns

### Synchronous Execution
- Default for most tools. Request-response, bounded time.

### Async Job
For long-running operations:
- Start: return `job_id` immediately
- Poll: `check_job_status(job_id)`
- Result: `get_job_result(job_id)`

### Idempotent Operation
- Make operations safe to retry with identical results
- Accept an idempotency key if applicable
- Critical for Command Tools that agents may retry

---

## 6. Tool Output Patterns

### Response Shaper
- Transform raw API responses into agent-friendly flat dicts
- Select only relevant fields
- Rename cryptic field names for clarity

### Token-Efficient Response
- Return essential fields only, not entire API payloads
- Truncate long text fields
- Return counts instead of full lists when appropriate

### Paginated Result
- Use cursor-based pagination (not page numbers)
- Always include `has_more` or `next_page_token`
- Include total count if available

### Progressive Detail
- Return summary by default
- Accept an `include_details` flag for full data
- Expand specific fields on request

### GUI URL
- Include web URLs to view/edit the resource in a browser
- View URL, edit URL, dashboard links

### Partial Success
For batch operations, report mixed results:
```python
return {
    "succeeded": [...],
    "failed": [{"id": "x", "error": "reason"}],
    "total": 10,
    "success_count": 8,
    "failure_count": 2,
}
```

---

## 7. Tool Context Patterns

### Secret Injection
- Inject credentials at runtime via `context.get_secret()`
- Never accept secrets as tool parameters
- Never log or return secret values

### Context Boundary
- Define scope limits (root paths, tenant scope, permission limits)
- Validate boundaries in code, not prompts

---

## 8. Tool Resilience Patterns

### Recovery Guide (RetryableToolError)
Provide actionable error messages:
- **What** went wrong
- **Why** it failed
- **How to fix** (specific values, valid options)

```python
raise RetryableToolError(
    message="Label 'urgent' not found.",
    additional_prompt_content=f"Valid labels: {valid_labels}",
)
```

### Error Classification
Distinguish error types in your tool:

| Error Type | Class | When to Use |
|------------|-------|-------------|
| Retryable | `RetryableToolError` | LLM can fix the input |
| Permanent | `ToolExecutionError` | Known failure, no retry will help |
| Upstream | Auto-adapted | HTTP errors from `httpx`/`requests` |

### Confirmation Request
When input is ambiguous, return candidates:
```python
raise RetryableToolError(
    message="Multiple users match 'john'.",
    additional_prompt_content="Did you mean: john.doe, john.smith, john.k?",
)
```

### Fuzzy Match Threshold
- >90% confidence: auto-accept the match
- 50-90%: return options via `RetryableToolError`
- <50%: ask for different input

### Graceful Degradation
- Return partial results when full operation fails
- Note what failed and suggest remediation

---

## 9. Tool Security Patterns

### Permission Gate
- Enforce access control in code, not prompts
- "Prompts express intent; code enforces rules"

### Scope Declaration
- Declare minimum required OAuth scopes per tool
- Use the narrowest scope that works
- Different tools can require different scopes

### Audit Trail
Log tool invocations:
- What: tool name, parameters (secrets redacted)
- Who: user context
- When: timestamp
- Result: success/failure

---

## 10. Cross-Cutting Patterns

### Tool Adapter
- Wrap legacy or complex APIs as clean, LLM-friendly tools
- Hide implementation complexity
- Shape responses for agent consumption

### Canonical Tool Model
- Use consistent data models across tools (User, Item, Event)
- Map API responses to canonical form
- Consistent field naming across all tools

---

## Quick Decision Checklist

When building each tool, ask:

1. Is this a Query, Command, or Discovery tool?
2. Are all string inputs constrained (Enums) where possible?
3. Does every optional param have a smart default?
4. Is the response shaped and token-efficient?
5. Does the error path guide the LLM toward a fix?
6. Are secrets injected, never passed through parameters?
7. Is the OAuth scope the minimum required?
8. Does the response include GUI URLs?
9. Is pagination cursor-based with `next_page_token`?
10. Would a `who_am_i()` discovery tool help this server?
