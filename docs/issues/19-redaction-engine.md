# Title

Redaction engine: secret masking for all exports

## Summary

Implement `src/export/redact.ts` per DESIGN §21: the built-in secret rule
set, user rules from config, a recursive JSON string walker that masks
matches with `«redacted:<ruleName>»`, and the `RedactionReport` consumed by
every exporter (20–22) and embedded in manifests/data islands.

## Context

Sharing without leaking keys is a launch-blocking safety property (US-5,
DESIGN §24). Redaction runs at export time only — the local DB keeps
originals. This module is pure and dependency-free, so it lands before the
exporters and is testable in isolation.

## Scope

- `src/export/redact.ts` + exhaustive tests
- Fixture secret corpus: `fixtures/redaction/secrets.txt` (synthetic,
  clearly fake values that still MATCH the rules) + negative corpus
  `fixtures/redaction/negatives.txt`

## Detailed Requirements

1. Rule table: implement all 12 built-in rules from DESIGN §21 exactly
   (names and patterns). Each rule: `{name, regex (with g flag), replace}`
   where `replace` defaults to the full-match replacement
   `«redacted:<name>»` but two rules keep structure:
   - `bearer-header`: keep group 1 prefix, mask the token part.
   - `env-secret-assignment`: keep the variable name and `=`/`:`, mask the
     value only.
2. `compileRules(config) → Rule[]`: built-ins + `redaction.customPatterns`
   (compile with user flags but force `g`; invalid handled at config load,
   issue 02). Custom rules run AFTER built-ins. Rule name collision with a
   built-in → config error (add to issue-02 validator in this issue).
3. `redactString(s, rules, report) → string`: apply rules in order;
   overlapping matches resolved by earlier-rule-wins (subsequent rules run
   on the already-masked string — masking is idempotent because
   replacements do not match any rule; test this).
4. `redactValue(v, rules, report) → v'`: recursive walker over
   objects/arrays/strings (numbers/bools/null pass through); does NOT
   mutate input; object key ORDER preserved. Keys themselves are not
   redacted (values only) — documented limitation comment.
5. `redactExport(bundle: {session, events, blobs: Map<hash,text>}, rules) →
   {bundle', report}`:
   - session: title, projectDir and all string fields walked.
   - events: `text` and `payload` walked (payload parsed → walked →
     re-serialized).
   - blobs: each blob's text redacted; **content change ⇒ new sha256** —
     recompute hash, rewrite `$blob.hash` references inside event payloads
     via a hash-mapping pass, and update preview strings (previews were
     derived from original content — regenerate as first 1024 chars of the
     redacted blob).
   - report: `{applied: true, rules: [{name, count}] (only count>0),
     total}`; counts = number of replacements (regex match count).
6. `disabled` path: when `--no-redact` (exporter passes `apply: false`) →
   return input untouched + `{applied: false, rules: [], total: 0}`.
7. Performance: single-pass-per-rule over strings; blob redaction streams
   are unnecessary (blobs already in memory at export); budget: 10 MB of
   text through 12+ rules < 2 s (informal perf test, skipped in CI by env
   flag, measured in issue 23).

## Acceptance Criteria

- [ ] Positive corpus test: every line in `fixtures/redaction/secrets.txt`
      (≥ 2 synthetic samples per built-in rule, including a multi-line
      private-key block and a JWT) is fully masked — assert NO line still
      matches any rule after redaction.
- [ ] Negative corpus: normal code/text lines (git SHAs, UUIDs, base64
      snippets < thresholds, the word "token" alone, `sk-` prefix shorter
      than 20 chars) are unchanged.
- [ ] Structural tests: bearer keeps `Authorization: Bearer ` prefix;
      env-assignment keeps `MY_API_KEY=`; nested payload walk; array walk;
      no input mutation (deep-freeze input in test); key order stable
      (JSON.stringify equality on a no-match object).
- [ ] Blob rehash test: blob containing a fake key → new hash, event
      `$blob.hash` remapped, preview regenerated from redacted text, report
      counts it once.
- [ ] Idempotence: redact(redact(x)) === redact(x).
- [ ] Report counts exact on a constructed multi-hit document.
- [ ] Custom rule from config applied after built-ins; collision with
      built-in name → config error test (issue-02 validator extended).
- [ ] Lint/typecheck/CI green.

## Validation

`npm test -- export/redact`. Reviewer spot-checks the corpus files for
realism (patterns match) and fakeness (values are obviously synthetic, e.g.
`sk-ant-api03-FAKEFAKE…`). The CI secret-guard grep (issue 05) must be
updated to EXCLUDE `fixtures/redaction/` only if it false-positives — prefer
constructing corpus values that the guard's narrower patterns do not flag
(document which).

## Dependencies

- 02 (config customPatterns), 04 (blob hashing helpers)

## Non-goals

- No redaction of the local DB or UI views (exports only).
- No entropy-based detection (v2), no key-name-based object dropping.

## Design References

- DESIGN §21 (rules, report), §24 (safety), §20.2–20.4 (consumers)
