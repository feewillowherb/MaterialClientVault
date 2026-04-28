# CodeRef Vault Guide

This vault follows the standard CodeRef root layout.

- `docs/` stores research notes, explanations, and walkthroughs.
- `repos/` stores Git submodules that point to source repositories or samples.
- `index.yaml` is the canonical metadata entry for this vault root.

Repositories under `repos/` should be maintained through Git submodules rather than copied directly into the vault root.

The canonical `index.yaml` fields are:

- `schemaVersion`
- `type`
- `title`
- `docsRoot`
- `reposRoot`

Keep this structure stable so assistants and tools can understand the vault quickly.

---

## Output Language Configuration

Configure the default output language and rules for handling specialized terminology.

**Parameter Description:**

- `defaultLanguage` — Default output language (e.g., `zh-CN`, `en-US`)
- `preservedTerms` — List specialized terms/terminology to preserve in their original form

**Example Configuration:**

```yaml
language:
  defaultLanguage: en-US
  preservedTerms:
    - API names
    - NuGet package names
    - Code identifiers
    - Framework-specific terminology
    - Brand terms
```

**Output Rules:**

- Default output language is `{{defaultLanguage}}`.
- Terms listed in `preservedTerms` are preserved in their original form without translation.

---

## Research Output Format

All research output should be placed in a dedicated folder under `docs/` rather than scattered as multiple independent files.

### Folder Naming Format

```
docs/<YYYY-MM-DD>-<proposal-or-topic-name>/
```

**Naming Rules:**

- Date format: `YYYY-MM-DD` (ISO 8601)
- Proposal/topic names in English or other languages, with words separated by hyphens `-`
- Avoid special characters except hyphens

### Folder Structure Example

```
docs/2026-04-28-example-research-topic/
├── 00-research-overview.md      # Required: Entry document with summary and TOC
├── 01-section-name.md
├── 02-section-name.md
├── 03-section-name.md
├── 04-quick-reference.md        # Optional: Quick reference card
└── assets/                       # Optional: Diagrams, images, etc.
    ├── diagram.png
    └── ...
```

### Document Numbering Rules

- Documents in a folder are numbered sequentially starting from `00`
- Recommended numbering interval: `00`, `01`, `02`...
- Each research folder must include a `00-research-overview.md` as the entry point

### 00-research-overview.md Structure Reference

```markdown
# {{Research Topic}}

**Date:** YYYY-MM-DD
**Author/Maintainer:** [Name]
**Status:** [In Progress/Completed]

## Summary

A brief overview of the research.

## Document Navigation

- [01-Section Name](01-section-name.md)
- [02-Section Name](02-section-name.md)
- [03-Quick Reference](03-quick-reference.md)

## Key Findings

- Finding 1
- Finding 2
- ...

## Related Resources

- Link 1
- Link 2

---

See individual section documents for details.
```
