# git notes: part 1 (solo dev)

notes from working through the first half of primeagen's boot.dev git course. this half is about using git by yourself, not with a team. part 2 (forking, conflicts, cherry picking, worktrees etc) is a separate post.

## table of contents

- [the big realization: `.git` is the whole thing](#the-big-realization-git-is-the-whole-thing)
- [commits are full snapshots, not diffs](#commits-are-full-snapshots-not-diffs)
- [three states a file can be in](#three-states-a-file-can-be-in)
- [plumbing vs porcelain](#plumbing-vs-porcelain)
- [objects: trees and blobs](#objects-trees-and-blobs)
- [branches are just pointers](#branches-are-just-pointers)
- [merging](#merging)
- [fast forward merges](#fast-forward-merges)
- [rebase](#rebase)
- [reset: soft vs hard](#reset-soft-vs-hard)
- [remotes, origin, upstream](#remotes-origin-upstream)
- [.gitignore](#gitignore)
- [quick reference](#quick-reference)

## the big realization: `.git` is the whole thing

a repo is just a normal folder plus a hidden `.git` folder. that's it. and `.git` isn't some metadata sidecar, it's literally the entire state of the project: every commit, every branch, config, all of it, stored as plain files. `git init` just sets up that structure.

so any time i wonder "how does git know X," the answer is always: it's a file in `.git` somewhere.

## commits are full snapshots, not diffs

this one broke my mental model. i assumed each commit only stored what changed, like a delta. it doesn't. every single commit is a complete snapshot of the whole project at that point in time. git keeps things small through compression, not through storing diffs.

so when i cat a commit object out and follow it down (commit → tree → blob), i'm literally walking through a full copy of the project as it existed at that commit.

## three states a file can be in

- **untracked**: git has no idea this file exists. delete it and it's gone, no recovery.
- **staged** (the "index"): you've told git this change is going into the next commit.
- **committed**: locked into history.

`git add` moves untracked → staged. one gotcha: `git add .` only adds from the current directory downward, not the whole repo, so running it from a subfolder won't pick up files elsewhere.

## plumbing vs porcelain

porcelain = the commands you actually use day to day (`add`, `commit`, `log`, `merge`, `rebase`). plumbing = the low level stuff underneath (`cat-file`, `hash-object`, `commit-tree`). you'll live in porcelain 99% of the time, but poking at the plumbing is what makes the porcelain actually make sense instead of feeling like magic.

## objects: trees and blobs

everything git stores lives in `.git/objects`, named by sha hash. two terms to know:

- **tree** = a directory
- **blob** = a file's contents

a commit points to a root tree. that tree points to blobs (files) and other trees (subfolders). you can walk this by hand with:

```
git cat-file -p <hash>
```

cat a commit, get a tree hash. cat the tree, get a blob hash. cat the blob, get the actual file contents. doing this once made the whole "commit = snapshot" thing click for real.

## branches are just pointers

a branch is a mutable pointer to a commit. nothing more. it's a single file living in `.git/refs/heads/` containing a commit hash. creating a branch doesn't copy any data, it just writes a new pointer file at your current commit.

```
git switch -c new-branch
```

this creates and switches in one step. `git checkout -b` does the same thing but it's the older command; `switch` is the one being pushed as the modern default.

## merging

when two branches diverge, they share a **merge base**: the nearest commit both branches have in common. merging replays both branches' changes into a new commit that has **two parents**, this is the only kind of commit with two parents. that's the definition of a merge commit.

## fast forward merges

if the branch you're merging into is a direct ancestor of the branch you're merging from (nothing diverged), git doesn't create a merge commit at all. it just slides the pointer forward. no new commit, no two parents, just a pointer move.

this matters because it's the reason people care about keeping history linear, a fast forward merge is basically free and keeps your log clean.

## rebase

took me a while to get why people either love or hate this. what rebase actually does: it takes the commits that diverged on your branch and replays them on top of a different base commit (usually the latest main).

the important part: replayed commits get **new hashes**. that's not a bug, it's because a commit's hash depends on its parent, and rebasing changes the parent. so d and e might have identical messages/content but completely different hashes after a rebase.

the payoff: once your branch is rebased onto the tip of main, there's no divergence left, so merging back in becomes a fast forward. no merge commit needed.

## reset: soft vs hard

`git reset` moves your branch pointer backward. two flags that matter:

- `--soft`: moves the pointer, keeps everything as staged changes. nothing is lost, you just "uncommit."
- `--hard`: moves the pointer AND wipes the index and working directory. anything uncommitted is gone.

one thing to be careful of: `reset --hard` never touches untracked files, git never knew about them so it has nothing to reset. but if you have real uncommitted changes sitting in your working tree, `--hard` will nuke them, and recovering that later is a pain (reflog only tracks commits, not uncommitted work).

rule i'm taking from this: default to `--soft` unless i'm sure i want to throw work away.

## remotes, origin, upstream

git is fully distributed. there's no special "central" repo built into git itself, a remote is just another repo somewhere. GitHub is just someone else's repo that the whole industry agreed to treat as the source of truth by convention.

- `origin` = your main remote, by convention
- `upstream` = the authoritative remote when you're working off a fork

`git fetch` pulls in new data but doesn't touch your local branches at all. you still need to merge or rebase to actually bring those changes into your work.

## .gitignore

tells git which files to never track (classic case: `node_modules`). you can also flip it and ignore everything, then explicitly re-include what you want.

good thing to know: there's also `.git/info/exclude`, which works exactly like `.gitignore` but is local only, it doesn't get committed or shared with the team. good spot for personal junk files (scratch notes, local scripts) you don't want cluttering the shared ignore list.

## quick reference

| command | what it does |
|---|---|
| `git init` | creates the `.git` structure |
| `git add <file>` | untracked/modified → staged |
| `git commit` | staged → permanent snapshot |
| `git switch -c <name>` | create + switch to a new branch |
| `git cat-file -p <hash>` | inspect any git object |
| `git reset --soft <ref>` | move branch pointer back, keep changes staged |
| `git reset --hard <ref>` | move branch pointer back, wipe everything |
| `git fetch` | download remote data, don't touch local branches |
| `git remote add origin <url>` | wire up a remote |

---

*source: [boot.dev's learn git course](https://boot.dev), taught by primeagen*
