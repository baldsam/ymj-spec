# YMJ Selector Registry (Draft)

Purpose
- Canonical list of selector types used under `ops.body_selector`.
- Enables validators to check syntax and warn on risky patterns.

Selector Types
- `first_fenced:<lang>` — Extract first fenced code block with language `<lang>`.
- `nth_fenced:<lang>:<n>` — Extract nth fenced block of `<lang>` (1-based).
- `block_id:<id>` — Extract a fenced block explicitly labeled or annotated by id (tooling-defined).
- `section:<heading>` — Extract Markdown section by exact heading text.
- `heading_path:<h1>/<h2>/...` — Locate content under a specific heading path (Experimental).
- `between:<start_regex>:<end_regex>` — Extract content between regex matches (Permissive only; warn in Strict).
- `regex:<pattern>[/flags]` — Extract first match group from regex (Permissive only; warn in Strict; avoid catastrophic patterns).

Validation Rules
- Structured selectors (`first_fenced`, `nth_fenced`, `block_id`, `section`) are preferred in Strict mode.
- Regex-based selectors (`regex`, `between`) SHOULD be flagged with a warning in Strict mode.
- Implementations SHOULD cap runtime and length for regex operations.

Notes
- Keep this list versioned; align with SPECIFICATION.md §13.

