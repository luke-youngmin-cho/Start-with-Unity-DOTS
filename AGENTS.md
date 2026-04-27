# AGENTS.md

Guidance for AI coding agents working in this repository.

This is a **docs-only** repository — a community manual for Unity DOTS / Entities 6.5.0 and Netcode for Entities 6.5.0. There is no Unity project to build, no tests to run. All work is reading, writing, and editing Markdown files.

> Claude Code reads `CLAUDE.md`, which imports this file via `@AGENTS.md`. Edit this file — not `CLAUDE.md` — when updating agent rules.

---

## 1. Target versions

| Component | Version |
|---|---|
| Unity | **6000.5+** (LTS) |
| Entities | **6.5.0** |
| Netcode for Entities | **6.5.0** |
| Collections / Mathematics / Entities Graphics | Core Packages (built into the Editor — no Package Manager install) |

Do **not** reference Entities 1.x APIs as if they were current. The 1.x → 6.5 migration lives in `Migration/` and is framed as legacy throughout. Netcode for Entities migration notes live in `Changelog/Netcode for Entities 1.4 → 6.5 Key Changes.md`; keep package-version facts tied to the Netcode changelog, not memory.

---

## 2. Language

**English only.** Every doc, every code comment. Do not add Korean / Japanese / other languages even if the user writes prompts in another language — that is a prompt-language choice, not a content-language choice.

---

## 3. Folder purposes

| Folder | Purpose | Audience |
|---|---|---|
| `Getting Started/` | First-run setup, Core Packages, Hello-DOTS tutorial. | New to DOTS. |
| `DOTS Workflows/` | Conceptual + hands-on topics (baker, ECS nouns, components, systems, jobs, ECB, Netcode workflows). Main body. | Working through the manual. |
| `Optimizations and Debugging/` | Chunk layout, Inspector/Query/Systems windows, profiler, managed reference audit. **No "DOTS" prefix on this folder name.** | Performance / debugging pass. |
| `Migration/` | Upgrading a 1.x project to 6.5 on Unity 6000.5+. | Existing 1.x users — distinct from Getting Started. |
| `Changelog/` | Version diff summary (1.4 → 6.5). | Reference. |

Canonical reading order for humans: `README.md` § "Recommended Learning Path". Do not duplicate that order here — link to it.

**Terminology lookup:** [`GLOSSARY.md`](GLOSSARY.md) at repo root — consult for one-line definitions of any term used in the manual. Deep discussions live in the linked docs.

---

## 4. Page structure standard

Every doc follows this skeleton (modelled on Unity's Netcode for Entities manual):

```
# <Doc Title>
### Unity 6000.5 · Entities 6.5.0
---
## 1. <Overview — why this doc exists>
## 2. <Core concept A>
## 3. <Core concept B>
...
## N-1. Code example / walk-through
## N. Troubleshooting
    | Symptom | Cause / Fix |
```

Conventions:
- `#` for the doc title, `###` for the fixed `Unity 6000.5 · Entities 6.5.0` subtitle.
- Main sections: `## N. <Title>` (numbered with a trailing period).
- Sub-sections: `### N.M <Title>` (see `DOTS Workflows/06_Enableable Component.md` §5.1, §5.2).
- Separate every `##` block with a `---` horizontal rule.
- The **final section is a Troubleshooting table** with `| Symptom | Cause / Fix |`. Omit the section only when the topic genuinely doesn't produce troubleshooting cases — don't fake entries.

`_TEMPLATE.md` at repo root is the canonical skeleton. Copy it when creating a new doc.

**Golden docs to pattern-match:**
- `DOTS Workflows/06_Enableable Component.md` — exemplar troubleshooting table + sub-numbered examples.
- `Getting Started/03_Hello DOTS — First Entity.md` — exemplar tutorial pacing.
- `DOTS Workflows/03_ECS Core Concepts.md` — exemplar conceptual doc with Mermaid diagram.

---

## 5. Code examples

- Always language-tagged: ` ```csharp ` (never a bare ` ``` `).
- Prefer **`ISystem`** over `SystemBase`. Prefer **`IJobEntity` / `IJobChunk`** over the obsolete `Entities.ForEach` (still compiles on 6.5 with deprecation warnings; removal planned for a future major release).
- Examples are **Burst-compatible**. Jobs carry `[BurstCompile]` unless the example is specifically demonstrating a non-Burst case.
- Use `SystemAPI.Query<RefRW<T>>()` / `SystemAPI.GetComponentLookup<T>()` patterns.
- Include enough context that a copy-paste compiles: `using` directives, `partial` modifier on system/job structs, namespace where relevant.

**Must NOT appear in new code examples:**
- `Entities.ForEach` — obsolete since 1.4; still compiles on 6.5 with deprecation warnings; removal planned for a future major release. Migrate via [`Migration/04_foreach → IJobEntity.md`](Migration/04_foreach → IJobEntity.md).
- `IAspect` — obsolete since 1.4; still compiles on 6.5 with deprecation warnings; removal planned for a future major release. Migrate via [`Migration/05_IAspect Removal.md`](Migration/05_IAspect Removal.md).
- Raw `Entity` handles in save files or cross-world protocols — use a game-owned stable ID or content key.
- Managed Unity object fields in hot `IComponentData` — use baked data, `BlobAssetReference<T>`, or `UnityObjectRef<T>` depending on the use case.

---

## 6. Links

- Use **relative paths with literal characters** (unencoded): `../DOTS Workflows/01_Baker Pattern & SubScene.md`.
- GitHub renders both encoded and literal correctly; literal is friendlier to filesystem-reading tools.
- Verify the target file exists on disk before committing a new cross-link.

---

## 7. Frontmatter

When adding or editing a doc, keep (or add) a YAML block with at minimum:

```yaml
---
title: <Doc title>
updated: YYYY-MM-DD    # ISO date of last meaningful edit
folder: <Getting Started | DOTS Workflows | Optimizations and Debugging | Migration | Changelog>
---
```

Do not invent additional fields without updating this spec first.

---

## 8. What NOT to invent

Accuracy matters more than completeness here. When documenting API surface:

1. **Never** invent method names, type names, or flags. If unsure, write a stub and ask the maintainer rather than guessing.
2. Cross-check against official package documentation:
   - Official manual: https://docs.unity3d.com/Packages/com.unity.entities@6.5/manual/index.html
   - Changelog: https://docs.unity3d.com/Packages/com.unity.entities@6.5/changelog/CHANGELOG.html
   - Netcode manual: https://docs.unity3d.com/Packages/com.unity.netcode@6.5/manual/index.html
   - Netcode changelog: https://docs.unity3d.com/Packages/com.unity.netcode@6.5/changelog/CHANGELOG.html
3. If you cannot verify an API name in the above, leave the section as `> TODO: verify <name> against 6.5.0 API surface` and stop there.

---

## 9. Scope discipline

- **Don't** rename existing files — cross-links break silently on rename.
- **Don't** add new top-level folders or restructure the approved layout.
- **Don't** merge or split existing docs without maintainer approval.
- **Small fixes** (typo, link correction, troubleshooting-table backfill) — just do them.
- **New docs** — copy `_TEMPLATE.md`, fill it in, then add links from:
  - `README.md` § Recommended Learning Path (if part of the main flow)
  - Any related existing doc that logically precedes or follows it

---

## 10. Maintainer-only context

- Legacy 1.4 docs: `legacy/entities-1.4` branch, pinned at commit `2296a52`.
- Main branch history: scaffold `cdb4aba` → first-draft completion `61c7685`. Further work is incremental.
- Top-level layout is user-approved and stable. Structural changes need explicit sign-off.
- **"Placeholder release" in the Changelog** — `Changelog/Entities 1.4 → 6.5 Key Changes.md` §2 describes Entities 6.5.0 as a "placeholder release". This is **Unity's own wording** (direct quote from the official package `CHANGELOG.html`) signifying a version rename with no substantive package-level API changes. It is not a TODO or an incomplete section — leave as-is.
