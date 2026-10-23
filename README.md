# Ponytail — Hermes edition

> "The best code is the code you never wrote."

An adapted copy of [DietrichGebert/ponytail](https://github.com/DietrichGebert/ponytail),
tuned for the [Hermes Agent](https://github.com/NousResearch/hermes-agent) skill
system. The idea, the ladder, and the safety carve-outs are Dietrich's. This
repo exists because the copy actually running on my machines has drifted, and a
diff is more honest than a description.

Original work: **[DietrichGebert/ponytail](https://github.com/DietrichGebert/ponytail)**,
MIT, canonical file `skills/ponytail/SKILL.md`. See [Credit](#credit) for the
pinned revision.

## What it does

It makes an agent write like a lazy senior developer. Lazy here means
efficient, not careless. The agent climbs a ladder before writing anything and
stops at the first rung that holds:

1. Does this need to exist at all?
2. Is it already in this codebase? Reuse it.
3. Does the standard library do it?
4. Does a native platform feature cover it?
5. Does an already-installed dependency solve it?
6. Can it be one line?
7. Only then: the minimum code that works.

Then it holds the agent to the consequences: fix the root cause rather than
the reported symptom, no abstraction with one implementation, no scaffolding
"for later", mark deliberate shortcuts with their ceiling and upgrade path,
and leave one runnable check behind for non-trivial logic.

It also defines what lazy never means: input validation at a trust boundary,
error handling that prevents data loss, security, accessibility, and anything
the user explicitly asked for.

## Why it works

Most agent slop is not incompetence. It is the model reaching for a framework
because that is what the training data did. Ponytail replaces "what would a
senior engineer do" with a short ordered list of cheaper answers, and then asks
for the fix to be provable. The diff gets smaller and the reasoning gets
shorter at the same time.

Two behaviors matter as much as the ladder:

- **Bug fix = root cause.** Before editing, find every caller of the function
  you are about to touch. One guard in the shared path beats a guard in every
  caller, and patching only the path the ticket names leaves the siblings
  broken.
- **Read fully, then be lazy.** Laziness that skips comprehension to ship a
  small diff is the dangerous kind. It dresses up as efficiency and ships a
  confident wrong fix.

## What this copy adds

- **Hermes frontmatter and invocation.** `/pony lite|full|ultra`, sticky across
  sessions, plus the intensity table.
- **Tool discipline.** Rules for the failure mode a long-running agent actually
  hits: don't repeat a failing call, don't re-read an artifact whose content you
  already have, don't stall when one exploration path dies. Break the loop with
  a different tool, not a third retry.
- **A preflight-audit reference.** `references/stateful-preflight-audits.md`
  for read-only audits of changes to live backend selection, persisted work,
  telemetry policy, or confirmation-owned UI.
- **Human comments.** No agent-signature markers in shipped code; the ceiling
  convention stays internal.

Roughly 9% of the wording still matches upstream and the rest was rewritten.
That is why this is published as an adaptation with a pinned revision rather
than something claiming to be stock.

## Install

Hermes indexes any directory under its skills tree containing a `SKILL.md`:

```bash
git clone https://github.com/pdscomp/Ponytail.git ~/.hermes/skills/software-development/ponytail
```

Then `/ponytail full` (or `lite` / `ultra`), or `/ponytail off` to revert. It
is plain markdown with no tooling attached, so it also works as a
system-prompt fragment for Codex, Claude Code, or a bare API call. If you are
not on Hermes, take the original; it is harness-agnostic by design.

## Output

Code first. Then at most three short lines: what was skipped, and when to add
it.

```text
[code]

skipped: [what you did not do]
add when: [condition that would justify the bigger version]
```

Explanation the user asked for is not debt. The rule is only against
unrequested prose.

## References

| File | Use |
|---|---|
| `references/stateful-preflight-audits.md` | Read-only audit checklist for changes to live backend selection, persisted work, telemetry policy, or confirmation-owned UI |

## Credit

Adapted from **[DietrichGebert/ponytail](https://github.com/DietrichGebert/ponytail)**
by **Dietrich Gebert**, MIT licensed, canonical file `skills/ponytail/SKILL.md`.

Forked at revision **[`b6c0448`](https://github.com/DietrichGebert/ponytail/tree/b6c04480c03e8db2f035751d7c46289779ec3362)**
(2026-07-10, "fix: narrow the ponytail: marker to real corner-cuts, keep the
prefix (#120) (#577)"), the revision current when this copy was made. Upstream
has since rebuilt the file; its `main` is
`09c434ab73ca43761c5d8b0c5d341926c2fba710`. If you want the maintained
original rather than my adaptation, go upstream.

The ladder, the YAGNI framing, the output pattern, and the safety carve-outs
are his. Local: the Hermes frontmatter and invocation, the tool-discipline
section, the preflight-audit reference, and the comment-marker rules.

## License

MIT. Original copyright © 2026 Dietrich Gebert; modifications © 2026 Paul
Swenson. See [LICENSE](LICENSE).
