# Contributing to harness-plugin-developer-skill

This repository contains one artifact — a Codex skill that teaches DeepSeek Harness plugin development — kept in three synchronized copies. Contributions must respect the anti-hallucination rules built into the skill itself.

## Source of truth

- `harness-plugin-developer/SKILL.md` is the **authoritative source**. All content edits start there.
- `harness-plugin-developer.skill.md` is the derived single-file copy; its frontmatter carries `derived-from: harness-plugin-developer/SKILL.md`. It must match the authoritative body exactly, apart from that one frontmatter line.
- `C:\Users\Elysia\.codex\skills\harness-plugin-developer\SKILL.md` is the installed copy Codex actually loads. It must be byte-identical to the authoritative file.

## Change rules

1. Edit only the authoritative `SKILL.md` (plus the root docs such as this file).
2. Re-sync after every edit: copy the authoritative body into the single-file copy (keeping its `derived-from` line), then copy the authoritative file to the Codex install path. Plain file copies — there is no build system.
3. Verify with SHA256: the authoritative and installed copies must be identical; the single-file copy may differ only by its `derived-from` frontmatter line. Also run the skill validator: `python C:\Users\Elysia\.codex\skills\.system\skill-creator\scripts\quick_validate.py <skill-dir>`.

## Facts about Harness internals

- Every fact about DeepSeek Harness internals must be verified in the local checkout (`D:\Ai\deepseek-harness`) with `rg` before it is written.
- The knowledge base is pinned to a specific checkout revision — see the "Knowledge Base Version Anchor" section in `SKILL.md`. After updating the checkout, run the upstream-change comparison procedure listed there and update the pin.
- The 78 built-in rows are the backbone of §9/§9.5/§10: list them with `rg -n "^- id:" packages/bundle/base/cordis.patch.yml` and compare against the tables.
- Anything that cannot be verified must be written as "needs verification" — never as a bare assertion.

## Delivery checklist (before opening a pull request)

- [ ] All three copies are in sync (SHA256 comparison above).
- [ ] `quick_validate.py` passes on the skill directory.
- [ ] Every new Harness fact cites a source path (`packages/...`, `docs/...`).
- [ ] Cross-references (§N) still point at existing sections after any renumbering.
- [ ] Code skeletons in §5 and §11 still compile against the pinned checkout.
- [ ] README section navigation, line counts, and the Reality-check row count match the current file.

## Non-goals

- No automated build or test step: this is documentation, verified by source reading and the validator.
- No automation of the copy sync: the three copies are intentional, reviewable files.
