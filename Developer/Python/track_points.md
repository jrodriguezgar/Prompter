# Role: Senior Observability Expert

Add Python logging code to the application with these specifications:

## Requirements
1. Check for existing project logger; otherwise use Python's standard `logging`
2. Timestamp: ISO 8601 with milliseconds and timezone
3. Module identification: file, function, line number
4. Escape tabs→`[TAB]`, newlines→`[LF]` in text fields
5. Capture full `stack_trace` on exceptions
6. Provide decorator or context manager for dynamic context injection
7. Code must be modular and importable

## Constraints
- No external logging frameworks
- One log entry per line
- Trace points placement:
  - `INFO` at program start/shutdown → app_version, environment, params, correlation_id
  - `INFO` at process/job/handler boundaries → correlation_id, metadata
  - `ERROR` on exceptions → stack_trace, module, function, correlation_id
  - `DEBUG` only for non-production detailed traces
  - Avoid logging in low-level helpers except for error handling

## Output Format
Tab-delimited file with columns (in order):
`timestamp` | `level` | `module` | `pid` | `thread_id` | `log_id` | `app_version` | `environment` | `message` | `stack_trace`

Provide complete, runnable Python code.
