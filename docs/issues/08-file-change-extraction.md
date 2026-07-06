# Title

File-change extraction: derive diff events from edit tools

## Summary

Implement `src/ingest/filechange.ts`: a stateful processor that watches the
normalized `tool_call`/`tool_result` stream during import and synthesizes
`file_change` events (path, kind, unified diff, add/del counts) for Claude
Code's Edit/Write/MultiEdit/NotebookEdit tools and Codex's `apply_patch`
(function call or shell heredoc), per DESIGN §12.

## Context

"Search diffs on a timeline" is a headline feature (README). Transcripts do
not store diffs as first-class records; they must be reconstructed from tool
payloads. This module is pure (stream in → extra drafts out) and sits between
adapters and the importer (issue 09 wires it).

## Scope

- deps: `diff@^5` (jsdiff), `@types/diff`
- `src/ingest/filechange.ts` + `filechange.test.ts`
- Patch-envelope parser for the `*** Begin Patch` grammar

## Detailed Requirements

1. API:

```ts
class FileChangeExtractor {
  /** feed each normalized draft in stream order; returns derived drafts to
      append immediately AFTER the fed draft */
  process(draft: NormalizedEventDraft): NormalizedEventDraft[];
  /** call at end of file: flush changes for tool_calls that never got results */
  finish(): NormalizedEventDraft[];
}
```

2. Pairing state: on `tool_call` for a relevant tool, store
   `{toolUseId → pending}`. On `tool_result` with a matching toolUseId, emit
   the derived `file_change` draft(s) (using result detail when useful) and
   clear pending. `finish()` emits from remaining pendings (call-only data,
   `ts` of the call).
3. Claude tools (match `toolName` exactly):
   - `Write {file_path, content}` → one change. kind: `create` unless
     `result.detail?.type === 'update'` → `modify`. diff: for create,
     `createTwoFilesPatch('/dev/null', file_path, '', content)`; for modify
     with `detail.originalFile` string → full diff original→content
     (`fragment:false`); modify without original → no diff, `fragment:true`.
   - `Edit {file_path, old_string, new_string}` → kind `modify`. Prefer
     `detail.structuredPatch` (array of `{oldStart,oldLines,newStart,
     newLines,lines[]}` hunks) → serialize into a unified diff body with
     proper `@@ -a,b +c,d @@` headers, `fragment:false`. Fallback:
     `createTwoFilesPatch(file_path, file_path, old_string, new_string)`
     with `fragment:true`. `replace_all: true` with fallback path → note
     appended to diff header line `# replace_all`.
   - `MultiEdit {file_path, edits[]}` → one `file_change`, kind `modify`,
     diff = concatenation of per-edit fragment diffs separated by
     `\n# edit N\n`, `fragment:true` (unless structuredPatch present → same
     rule as Edit).
   - `NotebookEdit {notebook_path,…}` → kind `modify`, path only, no diff.
4. Codex `apply_patch`:
   - Trigger A: `tool_call` with `toolName === 'apply_patch'`; patch text =
     `input.input ?? input.patch ?? (typeof input === 'string' ? input :
     null)`.
   - Trigger B: `toolName in {'shell','local_shell_call'}` whose
     `displayCommand ?? joined command` contains `apply_patch`; patch text =
     first substring between `*** Begin Patch` and `*** End Patch` inclusive
     in the raw command string(s).
   - Envelope parser (grammar per research §2.4): sections
     `*** Add File: p` → create (body lines start `+`; diff = new-file
     unified diff), `*** Update File: p` (+ optional next line
     `*** Move to: q` → renameFrom p, path q, kind `rename` when only moved,
     `modify` when hunks exist) with `@@` hunk bodies copied verbatim into a
     unified diff (`fragment:false`), `*** Delete File: p` → kind `delete`,
     diff omitted.
   - One `file_change` draft per file section, in section order.
   - Malformed envelope → zero drafts + one
     `error {origin:'parser', text:'apply_patch parse failed: …'}` draft.
5. Common: `additions`/`deletions` = count of body lines starting `+`/`-`
   excluding `+++`/`---` headers; `derivedFrom = toolUseId`; `ts` = the
   result's ts (or call's when finishing); `filePath` column = path (rename:
   new path); paths kept as written (no resolution — importer/query treat
   them as opaque; DESIGN §12 normalization note is satisfied by storing
   as-is when not resolvable).
6. Determinism + purity: no I/O, no clock; identical stream → identical
   output.

## Acceptance Criteria

- [ ] Unit tests per tool: Write create / Write update with originalFile /
      Edit with structuredPatch (header math verified against hand-computed
      `@@` values) / Edit fallback fragment / MultiEdit / NotebookEdit.
- [ ] apply_patch tests: Add+Update+Move+Delete envelope from
      `fixtures/codex/rollout-tools.jsonl` yields the expected ordered
      changes with kinds `create/rename-or-modify/delete`; shell-heredoc
      trigger B covered; malformed envelope → error draft, no throw.
- [ ] additions/deletions counts asserted on every diff-producing test.
- [ ] `finish()` emits pending call-only changes (test: Edit call with no
      result).
- [ ] Non-file tools (Bash, Read, web_search) produce zero drafts (explicit
      test).
- [ ] Lint/typecheck/CI green.

## Validation

`npm test -- ingest/filechange`. Reviewer hand-checks one generated unified
diff by applying it mentally against the fixture strings (documented in PR).

## Dependencies

- 06, 07 (draft shapes for both sources; fixtures)

## Non-goals

- No extraction from raw shell redirects/git commands (DESIGN §3.2).
- No filesystem reads to fetch file contents.
- No DB writes (importer wires persistence in issue 09).

## Design References

- DESIGN §8.1 (file_change payload), §12 (extraction table)
- docs/research/transcript-formats.md §1.4, §2.4
