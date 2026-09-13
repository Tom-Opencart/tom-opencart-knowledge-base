# Verification & Testing Standards for OpenCart Work

## Scope

**How to verify OpenCart changes honestly.** The static gate set, the structural checks a linter cannot
perform, how to build a static checker that does not lie, how to positive-test a gate, and how to report
evidence. This is a methodology pack, not a PHP unit-testing guide, and not a catalogue of baseline
behavior.

## Baseline

- Version: LiveStore `v3.0.4.5` (only where a claim is about core; most of this pack is method)
- Source: `https://raw.githubusercontent.com/19th19th/LiveStore/v3.0.4.5/upload/{path}`
- Verified on: 2026-09-13

## Confidence Levels

- `[verified]`: read directly from the cited baseline source
- `[baseline]`: grep or method listing only
- `[exists]`: existence-checked only
- `[needs-verification]`: inference
- `method`: a procedure recommended by this pack, not a claim about core behavior

---

## 1. The central rule

**Static analysis proves syntax and structure. It never proves behavior.**

| Category | Can establish | Cannot establish |
| --- | --- | --- |
| `php -l` | the file parses | that the code is reachable, correct, or safe |
| Twig tag balance | the template parses | that it renders, or that data is escaped correctly |
| XML well-formedness | the manifest parses | that a `<search>` block still matches its target |
| ZIP inspection | layout and separators | that the installer accepts the archive |
| Diff-scope check | which files ship | that the change works |
| Any style/standards gate | conformance | absence of a logic or authorization defect |

**A green static run is not evidence that a change works.** It is evidence that the change is
syntactically and structurally well-formed. State that distinction explicitly in every report.

## 2. The static gate set

Run on every change:

1. **PHP lint** — `php -l` on **every changed** `.php` file. Proves parseability only.
2. **Twig balance** — matched `{% %}` / `{{ }}` / `{# #}` and matched block tags on every changed `.twig`.
3. **XML validity** — `install.xml` parsed by a real DOM parser, not a regex.
4. **Archive layout** — entries use `/` separators (never `\`), `install.xml` sits at the archive root, and
   the archive contains exactly the intended paths. A partial/diff archive must contain **only** changed
   paths.
5. **Line endings** — the shipped artifact is LF, independent of the build host.
6. **Determinism (where the archive is committed)** — rebuilding identical source yields a byte-identical
   archive; otherwise every build produces a spurious change.

## 3. Structural checks a linter cannot do

### 3.1 Admin write-path permission audit

Full procedure in `opencart-core/permissions.md`. Summary: for every admin controller method that performs
a write, require a **dominating** `hasPermission('modify', <route>)` check. A route-level `access` grant is
not protection.

### 3.2 Orphaned validator check

For every `validate*()` method defined in a controller, confirm at least one call site. A validator that is
defined but never called is worse than a missing one: the code reads as if the path is protected.

### 3.3 Inert-setting check

For every setting exposed in an admin template, confirm a runtime consumer exists. Procedure in
`opencart-core/settings-and-config.md`.

### 3.4 Orphan-file check

For every new language file, controller and template, confirm a consumer. A language file whose controller
does not exist in the tree is dead weight; a template with no renderer is unreachable code.

### 3.5 Schema-trigger check

Confirm no DDL is reachable from a page load. **Search must follow cross-file calls**: the classic miss is
a controller `index()` calling a model's `upgradeSchema()`/`install()`, which a single-file scan cannot see.

## 4. Building a static checker that does not lie

Each trap below makes a checker either silently useless (false negatives) or actively misleading (false
positives). Every one of these was hit in practice, not theorised.

| Trap | Why it breaks | Correct approach |
| --- | --- | --- |
| Masking strings/comments before searching for a literal | blanking the *contents* of `'modify'` destroys the very text being searched for, so **no** method looks protected | parse structure over a **masked** copy; match content over the **raw** text. Two copies, two purposes |
| Fixed whitelist of entity names in write detection | misses `deleteThing()`, `syncNow()`-style calls, so whole methods are never examined | detect mutating **verbs** on model calls plus SQL keywords, not a noun list |
| Regex requiring empty parentheses `\(\)` | misses `validate($id)`, so the call site is invisible | match `name\s*\(` and read the arguments |
| Naive block matching that cannot close an empty body | `{\n\t}` never matches a pattern expecting a line before `}`, so the regex swallows following methods and reports wrong line numbers | count braces to find block boundaries |
| Checking only one syntactic form of a guard | a two-condition `\|\|` guard, or a guard in an enclosing `if`, is invisible | test whether the check **dominates** the write (placed brace / enclosing branch), not whether one spelling appears |
| Treating existence as verification | a file that exists may contain nothing relevant | read the block and cite lines |

## 5. Positive-testing a gate — mandatory

**A gate must first be shown to FAIL on synthetic violations. A gate that has never failed has not been
shown to work.**

Minimum case list for a write-path permission gate:

| Case | Expected |
| --- | --- |
| method with a write and no check at all | flagged |
| method using the weaker `access` check instead of `modify` | flagged |
| check placed **after** the write | flagged |
| check in a branch that does not dominate the write | flagged |
| correct inline check | not flagged |
| correct guard call in an enclosing `if` | not flagged |
| correct early `return` before the write | not flagged |
| method with no write at all | not flagged |

Record the pass/fail count. A gate whose author never ran these cases is a hypothesis, not a gate.

## 6. Evidence discipline for reporting

Report so that a reader can re-run and re-check:

- the **exact command** and its output, not a summary of it;
- **exact file paths and line numbers** for every claim;
- an explicit statement of what was **NOT** verified;
- never a behavior claim you did not observe.

**Honest-caveat pattern.** When a gate previously reported success and a defect was later found in exactly
the area it covered, the finding is about the **gate**, not only the code. Say so, and treat every prior
green result from that gate as unreliable until it is re-established.

**Never present an inference as a fact.** Use the confidence tags from `WORKFLOW.md`.

## 7. What genuinely requires a live store

Only a running store with real data can confirm:

- rendered output, including escaping and layout;
- actual query results and performance;
- authorization behavior for a restricted or demo admin user;
- installer/refresh behavior on a real archive;
- cache interaction and invalidation;
- browser JavaScript behavior and template rendering under real data.

**Rule: a release claim of "works" requires a live check performed by a human or in a real environment.**
Static gates may support such a claim; they cannot substitute for it.

## 8. ENFORCEABLE CHECKS

1. `php -l` on every changed PHP file, exit status checked.
2. Twig tag balance on every changed `.twig`.
3. `install.xml` parsed by a DOM parser.
4. Archive entry separators and root layout inspected.
5. Diff archive contains only intended paths.
6. Every gate that is relied upon has a recorded positive-test result.
7. Every report states its unverified areas.

## Verification Notes

- This pack is **methodology**: its recommendations are tagged `method` and are not claims about baseline
  behavior. The eight traps in §4 are drawn from concrete checker failures observed while auditing a real
  OpenCart theme; they are stated generically because the failures are general.
- No baseline source is required for §§1-8 except the cross-referenced packs, each of which cites its own
  verified lines.
- The specific gate commands (`php -l`, DOM parsing, ZIP inspection) are standard tooling, not
  OpenCart-specific behavior.
- Not verified: whether LiveStore ships CI enforcement for any of these gates. `[needs-verification]`
