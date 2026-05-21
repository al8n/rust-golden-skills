# rust-golden-skills

A collection of **Claude Code skills** encoding hard-won "golden rule"
conventions for writing Rust. Each skill is a self-contained `SKILL.md` that
Claude (or any agent that reads skills) loads to write and review Rust code that
matches these conventions by construction — so you don't have to repeat the same
review feedback on every PR.

## Skills

| Skill | What it covers |
|-------|----------------|
| [`rust-type-conventions`](skills/rust-type-conventions/SKILL.md) | Declaring structs, enums, and their accessors: encapsulation, `const fn`, opt-in `new()`/`Default` (delegate, don't re-derive), getter projection (`Option<&T>` / `&[T]` / `&str`), the `set_`/`with_`/`update_`/`maybe_`/`clear_` mutator vocabulary for `Option<T>`/`bool` fields, `#[must_use]` on consuming builders, `#[inline(always)]` on cheap accessors, `thiserror` errors, open `Other(_)` / coded `Unknown(n)` enums, no-module-name-stutter naming, and no-std feature tiers. |

## Layout

```
skills/
  <skill-name>/
    SKILL.md      # YAML frontmatter (name + description) + the skill body
```

A skill's `description` frontmatter is what an agent matches against to decide
when to load it; the body is the actual guidance.

## Using a skill

Skills are picked up from a skills directory. Make a skill available by copying
or symlinking its folder into one:

```sh
# User-level — applies in every project on this machine
ln -s "$PWD/skills/rust-type-conventions" ~/.claude/skills/rust-type-conventions

# Project-level — commit it so your whole team gets it
ln -s "$PWD/skills/rust-type-conventions" /path/to/project/.claude/skills/rust-type-conventions
```

Once installed, the agent loads it automatically when you write or review Rust
types, or you can invoke it explicitly (e.g. `/rust-type-conventions`) or
reference it in a prompt ("apply rust-type-conventions").

## Contributing a new skill

1. `mkdir -p skills/<skill-name>` and add a `SKILL.md`.
2. Frontmatter must have a `name` (kebab-case, matching the folder) and a
   `description` (one paragraph stating *when* to use it — that's the trigger).
3. Keep it prescriptive and example-driven; end with a review checklist.
4. Add a row to the **Skills** table above.

## License

Dual-licensed under either of [Apache-2.0](LICENSE-APACHE) or
[MIT](LICENSE-MIT) at your option.
