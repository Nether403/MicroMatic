+## Prompt Library
+
+### Bug Fix from Stack Trace
+1. Provide failing trace + reproduction steps.
+2. Ask Codex to analyze, propose fix, suggest tests.
+3. Require `pnpm test` run and diff summary before completion.
+
+### Large Refactor with Tests
+1. Outline target modules, constraints, acceptance tests.
+2. Request plan review (Agent mode) before code changes.
+3. Enforce `pnpm lint && pnpm test` verification and PR message draft.
+
+### Diagram from Code
+1. Supply key files/functions; specify diagram type (sequence/entity).
+2. Ask Codex to generate Mermaid diagram + explanation.
+3. Attach diagram to docs; mention ability to include screenshots for UI nits (images supported).
