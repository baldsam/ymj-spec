# YMJ Specification (YAML · Markdown · JSON)

**YMJ** is a machine-first, deterministic document format that combines:
1.  **YAML Header**: Structured identity and document semantics (about the doc).
2.  **Markdown Body**: Human-readable content (the payload).
3.  **JSON Footer**: Canonical operational metadata (how to handle the doc).

## The "Sandwich" Format

A `.ymj` file is strictly structured as:

```markdown
---
doc_summary: A short summary of the document
kind: note
version: 1.0.0
subject: Document Title
maintained_by: Author Name
ymj_spec: 1.0.0
---
# Document Title

This is the body of the document. It is standard Markdown.

- It can contain lists
- Code blocks
- And anything else

```json
{
  "schema": "1",
  "payload_hash": "sha256-of-body...",
  "index": {
    "embedding": [0.1, 0.2, ...],
    "tags": ["example", "ymj"]
  }
}
```
```

## Key Features

*   **Self-Contained RAG**: The JSON footer can store vector embeddings of the document body, allowing "Serverless RAG" without an external vector database.
*   **Deterministic Parsing**: The three-part structure is unambiguous. Tools can easily split the file into Header, Body, and Footer.
*   **Separation of Concerns**:
    *   **Header**: Identity (What is this?)
    *   **Body**: Content (What does it say?)
    *   **Footer**: Operations (How do I use/index this?)

## Tooling

The reference implementation for YMJ is available in the [SFA Toolkit](https://github.com/sfa-toolkit) as `sfa_ymj.py`.

### Common Operations

*   **Parse**: Split a file into its three components.
*   **Lint**: Validate structure and check if the `payload_hash` matches the body.
*   **Enrich**: Generate embeddings and update the hash in the footer.
*   **Migrate**: Convert legacy Markdown files to YMJ.

## Specification

See [SPECIFICATION.md](./SPECIFICATION.md) for the full technical details.

## Selector Registry

See [YMJ_SELECTOR_REGISTRY.md](./YMJ_SELECTOR_REGISTRY.md) for the registry of body selectors used to extract specific payloads from the Markdown body.
