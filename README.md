# Elastic KB Drafts

This repository holds AI-generated Knowledge Base article drafts created by the `case-resolve-kb` Claude Code skill.

## Structure

```
drafts/
  <case_number>-<slug>.md   # one file per draft
```

## Workflow

1. The `case-resolve-kb` skill detects a resolved support case and opens a PR here.
2. A support engineer reviews the draft.
3. Once approved, the article is published in the [Innovation Hub](https://support.elastic.dev/knowledge/create).

## Draft format

Each draft follows the Elastic KB article template:
- Title ≤ 64 characters
- Summary ≤ 256 characters
- Sections: Problem · Environment · Cause · Resolution · [Workaround] · [References]
- Private context fenced with `{elastic-private-context}` blocks

