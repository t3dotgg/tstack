# tstack

Agent skills, one folder each under `skills/`. This is a fork of
[t3dotgg/tstack](https://github.com/t3dotgg/tstack) that also holds the owner's
own skills, and it is merged with upstream regularly.

## Working here

- A skill is `skills/<name>/SKILL.md`, and the `name` in its frontmatter
  matches the folder name.
- Add new skills as new folders. The upstream skills can be edited too, but a
  line that upstream also changes will need a hand merge.
- Commit every change with a conventional message (`feat:`, `fix:`, `chore:`)
  and push it. The repo should never hold uncommitted work.
- The repo is public: no secrets or private client details in any file.
- `CLAUDE.md` only imports this file, so put instructions here.
