# Git Workflow Reference

How to scope git operations to the **plugin being built**, not whatever repo happens to sit above it. Read this before running any git command in `/plugin-build`.

## The problem this prevents

A WordPress plugin under development is almost never the top of its own repo. It usually lives nested inside something:

- `wp-content/plugins/my-plugin/` inside a full WordPress site that is itself a git repo
- a subfolder of a monorepo
- a subfolder of an unrelated repo that happens to be an ancestor on disk

`git rev-parse --is-inside-work-tree` returns `true` if the current directory is **anywhere inside** a work tree — it walks **up** the tree until it finds any `.git`. So a naive "is this a repo?" check finds the **ancestor** repo, skips `git init`, and then every `git checkout -b`, `git add`, and `git commit` silently operates on the wrong repo at the wrong root.

The fix is two rules:

1. **Pin the plugin directory** explicitly, once, at the start.
2. **Scope every git command to it** with `git -C "$PLUGIN_DIR"` — never rely on the current working directory.

## Step 1 — Resolve and pin the plugin directory

Determine the plugin's own root: the directory containing the **main plugin file** (the PHP file with the `Plugin Name:` header) and/or `composer.json` / the plugin bootstrap. This is the directory the task list is building into.

Record it for the whole session and announce it to the user:

```bash
PLUGIN_DIR="/absolute/path/to/wp-content/plugins/my-plugin"
```

Use `$PLUGIN_DIR` in every git command below. Do not assume it equals the current working directory.

## Step 2 — Detect the repo situation correctly

Do **not** use `--is-inside-work-tree`. Compare the repo top-level against the plugin directory:

```bash
TOPLEVEL=$(git -C "$PLUGIN_DIR" rev-parse --show-toplevel 2>/dev/null)
```

Three outcomes:

- **`$TOPLEVEL` is empty** → the plugin directory is not in any repo. Initialise a dedicated repo and make a baseline commit:

  ```bash
  git -C "$PLUGIN_DIR" init
  git -C "$PLUGIN_DIR" add -A
  git -C "$PLUGIN_DIR" commit -m "Initial state"
  ```

- **`$TOPLEVEL` equals `$PLUGIN_DIR`** → the plugin is already its own repo. Proceed; do nothing.

- **`$TOPLEVEL` differs from `$PLUGIN_DIR`** → the plugin is **nested inside a parent repo**. This is the case that silently commits to the wrong place. **Stop and ask the user** which they want (see next section). Do not commit anything until they choose.

## Step 3 — The nested-repo decision (ask the user)

When the plugin sits inside a parent repo, present these two options and wait for an answer:

- **Dedicated repo (recommended).** Give the plugin its own repository so the one-commit-per-task history stays clean and never touches the parent:

  ```bash
  git -C "$PLUGIN_DIR" init
  git -C "$PLUGIN_DIR" add -A
  git -C "$PLUGIN_DIR" commit -m "Initial state"
  ```

  (If the parent repo tracks this path, mention they may want to add the plugin dir to the parent's `.gitignore` or remove it from the parent index to avoid a nested-repo clash. Don't do this without asking.)

- **Commit into the parent repo, scoped to the plugin subtree.** Keep using the parent repo, but **only ever stage paths inside the plugin directory** — never `git add -A` or `git add .` from a parent location. Every `add` must be a path under `$PLUGIN_DIR`.

Record which option the user picked and apply it consistently for the rest of the build.

## Step 4 — Branching

Create the working branch on the **plugin's** repo, scoped with `-C`:

```bash
CURRENT=$(git -C "$PLUGIN_DIR" rev-parse --abbrev-ref HEAD)
# Stay on it if already a matching feature/* branch; otherwise:
git -C "$PLUGIN_DIR" checkout -b feature/<slugified-feature-name>
```

Never build directly on `main`/`master`. Tell the user the branch name.

## Step 5 — Per-task staging and commit

Stage **only** the files changed for the current task, addressed inside the plugin dir, then commit scoped to that repo:

```bash
git -C "$PLUGIN_DIR" add path/to/changed-file.php path/to/TASKS.md
git -C "$PLUGIN_DIR" commit -m "Task 1.1: <short task title>"
```

If the user chose "commit into the parent repo" in Step 3, the paths you pass to `add` must still resolve to files inside the plugin directory — never stage anything outside it.

Confirm the commit landed in the right repo when in doubt:

```bash
git -C "$PLUGIN_DIR" log --oneline -1
git -C "$PLUGIN_DIR" rev-parse --show-toplevel
```

## Step 6 — Push and PR (end of build)

All push/PR commands are scoped the same way:

```bash
git -C "$PLUGIN_DIR" push -u origin <branch-name>
```

For the PR, run `gh` from inside the plugin directory so it targets the plugin's remote:

```bash
gh -C "$PLUGIN_DIR" pr create ...   # if your gh supports -C
# otherwise run gh with the plugin directory as the working directory
```

If no GitHub remote exists for the plugin repo, print the branch name and tell the user to open the PR manually.

## The rules, condensed

- Resolve `$PLUGIN_DIR` once; announce it.
- Detect with `rev-parse --show-toplevel`, never `--is-inside-work-tree`.
- If nested in a parent repo, **ask before committing**.
- Put `-C "$PLUGIN_DIR"` on **every** git command.
- Stage explicit paths inside the plugin dir; never `git add -A` from a parent.
- When unsure where a commit landed, check `git -C "$PLUGIN_DIR" rev-parse --show-toplevel`.
