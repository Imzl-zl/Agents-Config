<!-- Inert legacy reference for design comparison. May include zhanggui adaptation notes; not discoverable, not invocable. Do not interpret the frontmatter below as invocation metadata. -->

---
name: using-superpowers
description: Use only when the user explicitly requests the legacy Superpowers bootstrap instead of zhanggui.
disable-model-invocation: true
---

# Legacy Superpowers Bootstrap

This skill is retained for manual compatibility. It is not the entrypoint for the integrated workflow.

When `zhanggui` is available:

1. Prefer `/zhanggui` for ideas, bugs, plans, implementation, and delivery.
2. Do not auto-load this skill at session start.
3. Do not auto-load legacy `brainstorming` before `/zhanggui`.
4. If the user explicitly selects this legacy flow, follow the requested legacy skill without mixing its orchestration rules into `zhanggui`.

Legacy skills may still be invoked individually when the user names them. Their selection is an explicit override, not an implicit route.

