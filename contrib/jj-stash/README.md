# jj-stash

Park commit chains out of every jj UI, then bring them back unchanged.

A prototype, as a shell script over jj and git, of something jj does not have: a
way to take a chain of commits out of the visible graph without losing it. The
use case is a repo with a wide fan-out of open threads, viewed in a UI that
cannot filter by revset. VisualJJ 0.35.3 has no revset setting at all, so
`revsets.log` and aliases do not reach it. The only way to shrink its graph is to
make commits invisible to jj itself.

```
jj-stash push NAME 'REVSET'   park a chain (REVSET must include its own descendants)
jj-stash list                 parked chains, one line each
jj-stash show NAME            print the plaintext
jj-stash pop NAME             bring the chain back and drop the stash
```

## How a chain is parked

Two things are written, then the chain is abandoned:

- **`.claude/stashes/NAME.md`, plaintext for reading.** It records what the chain
  sits on, its bookmarks, then every commit oldest first, each with its change
  ID, commit ID, parents, author, full description and `--git` diff. It exists
  so the chain can be read before deciding whether and when to bring it back.
  It doubles as a fallback patch. It lives in the main repo's root, whichever
  workspace `push` runs from, and must be ignored or jj would snapshot it into
  `@`: if nothing ignores `.claude/stashes/`, the script adds it to
  `.git/info/exclude`.
- **`refs/stash/NAME/{root-N,head-N,bookmark/<name>}`, git refs.** These keep
  the commits alive. jj imports only `refs/heads`, `refs/tags` and
  `refs/remotes`, so these refs do not make anything visible, but git's gc
  treats them as roots and never prunes the objects.

`jj abandon` then removes the chain from the view. It is gone from `jj log`,
VisualJJ and jjk.

## Why pop restores the same commits, not a replay

jj writes a `change-id` header into every git commit it creates:

```
tree bd3489b…
parent 3c61b3b…
author …
committer …
change-id qqrsypwrlqxqstzkuztmtzuwoopqussy
```

So the parked git objects *are* the jj commits. `pop` never re-applies
anything. It makes the original commits visible again, so change IDs, commit
IDs and timestamps all survive, and an export/replay step is unnecessary.

`pop`, in order:

1. **Record parents.** Before anything is revived, record the current version
   of every parent the chain's roots sat on. Reviving brings old versions back
   too, which would make `change_id()` ambiguous.
2. **Revive.** A bookmark on a hidden commit makes it and its ancestors visible
   again, and they stay visible after the bookmark is deleted. So each head
   gets a temporary bookmark, which is then deleted.
3. **Rebase if needed.** If a parent was rewritten while the chain was parked,
   `jj rebase -s ROOT -o CURRENT_PARENT…`. Descendants follow. Rebasing a
   hidden root directly, without step 2, revives only the root, not its
   descendants.
4. **Clean up.** Step 2 also revived the *old* version of any rewritten parent.
   Once the chain has moved off it, that copy has no children and duplicates a
   live change, so it is abandoned. Nothing else is touched.
5. **Restore bookmarks** by change ID, so they follow a rebase.
6. **Delete the refs and the plaintext.**

`push` refuses a revset that would strand a child (abandoning a commit whose
child is not abandoned rebases the child), one that contains any workspace's
working-copy commit, and one that contains immutable commits.

## Tested

Run against a copy of a real repo (jj 0.45.1, colocated, 131 mutable commits,
three workspaces):

| case | result |
|---|---|
| revset that strands a child | refused, naming the child |
| push, then pop, parents unchanged | identical change IDs **and** commit IDs; bookmark restored; no refs or plaintext left |
| a parent rewritten while parked | root rebased onto the new version; stale copy abandoned; no divergent commits; thread count unchanged |
| `jj util gc --expire now` then `git gc --prune=now`, while parked | objects survive; pop works |
| push from a secondary workspace nested under `.claude/worktrees/`, pop from the main one | plaintext lands in the main root, is ignored, and the main `@` is not snapshotted |
| second push | does not re-append the exclude rule |

## Limits

- **bash 4 or later.** `pop` uses an associative array. macOS's `/bin/bash`
  is 3.2.
- **Local only.** `jj git push` does not push `refs/stash/*`, so a stash exists
  in one repo. Back up `.claude/stashes/` if it matters.
- **Pushed bookmarks are not handled.** A remote bookmark (`name@origin`) keeps
  its commit visible, so a chain carrying one cannot be hidden this way.
  `abandon` also deletes the local bookmark, and a later `jj git push --deleted`
  would push that deletion. Only local bookmarks were tested.
- **Merges inside the chain.** Only the roots' parents are re-targeted. If a
  merge commit inside the chain has a parent outside it, and that parent is
  rewritten while parked, the old version of that parent comes back.
- **Conflicted bookmarks** (more than one target) are not recorded.

## If this became `jj stash`

The primitives are all in jj already. Hidden commits stay addressable by
commit ID, a bookmark revives them, and the view's heads decide visibility. A
native version would presumably keep parked heads in the view or op store
rather than in git refs, which would also make it work with non-git backends,
and could show them with a revset like `stashed()`.
