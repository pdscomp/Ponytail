# Ponytail

A skill that makes an AI coding agent write like a lazy senior developer.

Lazy here means efficient, not careless. It has seen every over-engineered
codebase and been paged at 3am for one. The best code is the code never
written.

## What it does

It gives the agent a ladder to climb before writing anything. Stop at the
first rung that holds:

1. Does this need to exist at all?
2. Is it already in this codebase? Reuse it.
3. Does the standard library do it?
4. Does a native platform feature cover it?
5. Does an already-installed dependency solve it?
6. Can it be one line?
7. Only then: the minimum code that works.

Then it holds the agent to the consequences: fix the root cause rather than
the reported symptom, no abstraction with one implementation, no scaffolding
"for later", mark deliberate shortcuts with the ceiling and the upgrade path,
and leave one runnable check behind for non-trivial logic.

It also defines what lazy never means: input validation at a trust boundary,
error handling that prevents data loss, security, accessibility, and anything
the user explicitly asked for.

## Why it works

Most agent slop is not incompetence. It is the model reaching for a framework
because that is what the training data did. Ponytail replaces "what would a
senior engineer do" with a short ordered list of cheaper answers, and then
asks for the fix to be provable. The diff gets smaller and the reasoning gets
shorter at the same time.

Two behaviors matter as much as the ladder:

- **Bug fix = root cause.** Before editing, find every caller of the function
  you are about to touch. One guard in the shared path beats a guard in every
  caller, and patching only the path the ticket names leaves the siblings
  broken.
- **Read fully, then be lazy.** Laziness that skips comprehension to ship a
  small diff is the dangerous kind. It dresses up as efficiency and ships a
  confident wrong fix.

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

## Install

Hermes indexes any directory under its skills tree containing a `SKILL.md`:

```bash
git clone https://github.com/pdscomp/Ponytail.git ~/.hermes/skills/software-development/ponytail
```

Then invoke it with `/ponytail full` (or `lite` / `ultra`), or paste the
`SKILL.md` into any other agent's context. It is plain markdown with no
tooling attached, so it also works as a system-prompt fragment for Codex,
Claude Code, or a bare API call.

## References

| File | Use |
|---|---|
| `references/stateful-preflight-audits.md` | Read-only audit checklist for changes to live backend selection, persisted work, telemetry policy, or confirmation-owned UI |

## License

MIT.
