# jj-stash

Park commit chains out of every jj UI, then bring them back unchanged.

A prototype, as a shell script over jj and git, of something jj does not have: a
way to take a chain of commits out of the visible graph without losing it. The
use case is a repo with a wide fan-out of open threads, viewed in a UI that
cannot filter by revset. VisualJJ 0.35.3 has no revset setting at all, so
`revsets.log` and aliases do not reach it. The only way to shrink its graph is to
make commits invisible to jj itself.

```
jj-stash push NAME 'REVSET'        park a chain; children outside it are detached
jj-stash list                      parked chains, with what each needs popped first
jj-stash show NAME                 print the plaintext
jj-stash pop [--with-deps] NAME    bring the chain back and drop the stash
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
- **`refs/jj-stash/NAME/{root-N,head-N,bookmark/<name>,reattach/<change>/N}`,
  git refs.** These keep
  the commits alive. jj imports only `refs/heads`, `refs/tags` and
  `refs/remotes`, so these refs do not make anything visible, but git's gc
  treats them as roots and never prunes the objects. The namespace is not
  `refs/stash/`: `git stash` keeps its own data in the single ref
  `refs/stash`, and git cannot hold both `refs/stash` and `refs/stash/…`, so
  in a colocated repo one of the two tools would break.

A child outside the chain is **detached, not stranded**. The typical case is a
merge that collects several lanes. The child is rebased onto its other
parents, and a `reattach/<child change ID>/N` ref records which chain commits
were its parents. `pop` adds them back. A child with no parent outside the
chain has nowhere to go, so `push` refuses and names it.

`push` is all or nothing. It records the operation it started from, and any
failure before the final abandon restores that operation and removes the refs
and plaintext it wrote.

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
5. **Restore bookmarks** by change ID, so they follow a rebase. Only local,
   non-conflicted bookmarks are recorded. A remote that is tracked but was
   never pushed is listed as `name@origin` with no target, and is skipped.
6. **Reattach** each detached child: its current parents, then the chain
   commits it lost, in that order.
7. **Delete the refs and the plaintext.**

`push` refuses a revset that would leave a child with no parent, one that
contains any workspace's working-copy commit, and one that contains immutable
commits.

## Dependencies between stashes

`pop` only knows each root's *direct* parent. Stash a dangling commit, then
the chain it dangles from, and the two now depend on each other in one
direction. Popping them in the wrong order is where it goes wrong:

- **Nothing rewritten in between:** reviving the dangling commit makes its
  ancestors visible, so its parent comes back as an ancestor while the other
  stash still claims it. That leaks into view. Edit the leaked commit, and
  popping the chain later revives the old version beside the edit: divergence.
- **An ancestor rewritten while both are parked:** the first pop revives the old
  ancestor and it diverges at once. The second pop happens to repair it, but
  the state in between is wrong.

So a stash **depends** on another when one of its roots sits on a commit that is
hidden and parked there, or when a child it detached (`reattach/`) is hidden
and parked there. Membership is jj's `roots::heads` for each stash, computed
with `git rev-list --ancestry-path`. That leaves out the ancestors of a merge's
outside parent, which a plain `rev-list heads ^parents` would wrongly include.
Everything is keyed by change ID, so it holds across rewrites.

- `pop NAME` refuses while NAME depends on another stash, names it, and
  changes nothing.
- `pop --with-deps NAME` pops the dependencies first, recursively, then NAME. A
  cycle is refused.
- `push` prints a note for every existing stash that now depends on the new
  one.
- `list` shows a `needs:` column.

Two alternatives were considered and not built. Restoring onto the *nearest
visible ancestor* is easy to find but amounts to `jj abandon` of the missing
link: the revision's diff conflicts wherever it depends on what the skipped
commits introduced. Parking a *partial graph* (a middle commit with its
descendants left visible) is impossible without rewriting those descendants,
because a commit's parents are part of its identity. They would show as
conflicted while parked. Parking related pieces as one stash, a union in one
revset, avoids both.

## Tested

Run against a copy of a real repo (jj 0.45.1, colocated, 131 mutable commits,
three workspaces):

| case | result |
|---|---|
| revset that would leave a child with no parent | refused, naming the child |
| a two-commit lane whose tip is one of 25 parents of a merge (real case: an issue lane under a "merge: issue lanes for review" commit) | lane hidden; merge kept with 24 parents; pop restores identical lane IDs, the merge's 25th parent and the `issue/134` bookmark |
| failure injected after the merge was detached, before abandon | op restored: merge back to 25 parents, no refs or plaintext left |
| push, then pop, parents unchanged | identical change IDs **and** commit IDs; bookmark restored; no refs or plaintext left |
| a parent rewritten while parked | root rebased onto the new version; stale copy abandoned; no divergent commits; thread count unchanged |
| `jj util gc --expire now` then `git gc --prune=now`, while parked | objects survive; pop works |
| push from a secondary workspace nested under `.claude/worktrees/`, pop from the main one | plaintext lands in the main root, is ignored, and the main `@` is not snapshotted |
| second push | does not re-append the exclude rule |
| dangling commit parked, then the 16-commit chain it sits on | push of the chain notes the dependency; `pop` of the dangling one is refused and changes nothing; the right order restores identical IDs |
| same, `pop --with-deps` on the dangling one | pops the chain, then it; identical IDs; no refs left |
| a lane that feeds a merge parked, then the merge's line | popping the lane first is refused (its reattach target is parked); `--with-deps` restores the line, then the merge gets the lane back as a parent |
| real stashes, parked by hand in the wrong order (a PR lane, then the readme lane it sits on) | found by `list`; `pop` refused; `--with-deps` restores both, PR back on the readme commit |
| an independent stash | pops without `--with-deps`; no false dependency across 27 real stashes |

## Limits

- **bash 4 or later.** `pop` uses an associative array. macOS's `/bin/bash`
  is 3.2.
- **Local only.** `jj git push` does not push `refs/jj-stash/*`, so a stash exists
  in one repo. Back up `.claude/stashes/` if it matters.
- **Pushed bookmarks are not handled.** A remote bookmark (`name@origin`) keeps
  its commit visible, so a chain carrying one cannot be hidden this way.
  `abandon` also deletes the local bookmark, and a later `jj git push --deleted`
  would push that deletion. Only local bookmarks were tested.
- **Parent order of a reattached merge** is not preserved: the chain's
  commits are appended after the child's current parents.
- **Merges inside the chain.** Only the roots' parents are re-targeted. If a
  merge commit inside the chain has a parent outside it, and that parent is
  rewritten while parked, the old version of that parent comes back.
- **Conflicted bookmarks** (more than one target) are not recorded.
- **Dependency cycles** are refused, but no real cycle was found to test it.
  Chains deeper than one level (A needs B needs C) are handled by the same
  recursion but were not tested directly.

## Found by testing

- `jj bookmark list -r` also returns remote-tracking entries. An unpushed
  tracked remote has no target, and the first version passed that non-commit
  to `git update-ref`.
- `git for-each-ref 'bookmark/*'` matches one path component, so `issue/134`
  was parked but not restored. A glob-free prefix fixed it.
- An ignore check run relative to a secondary workspace under
  `.claude/worktrees/` matched that directory's own ignore rule. The plaintext
  then went unignored and jj snapshotted it.
- Under `set -e` and `pipefail`, a dependency check whose last test came out
  false returned non-zero and would have aborted the script from inside
  `$(...)`.
- `list` took 38 seconds over 27 stashes: the ownership map was built inside
  a `$(...)` subshell, thrown away, and rebuilt per stash. It is now built
  once in the main shell, which the subshells inherit (2.3 seconds).

## If this became `jj stash`

The primitives are all in jj already. Hidden commits stay addressable by
commit ID, a bookmark revives them, and the view's heads decide visibility. A
native version would presumably keep parked heads in the view or op store
rather than in git refs, which would also make it work with non-git backends,
and could show them with a revset like `stashed()`.
