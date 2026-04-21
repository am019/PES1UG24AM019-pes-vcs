# PES-VCS Lab Report

**Name:** Ahan Mysore  
**SRN:** PES1UG24AM019  
**GitHub Username:** `am019`

This repository contains my implementation of PES-VCS, a small version control system that supports object storage, tree construction, staging, commits, and commit history traversal. I preserved the original lab handout as [LAB_HANDOUT.md](LAB_HANDOUT.md).

## Repository Quick Links

- Report screenshots: [`screenshots/images`](screenshots/images)
- Raw command outputs: [`screenshots/logs`](screenshots/logs)
- Screenshot generation helpers: [`scripts`](scripts)

## Build And Test

On Ubuntu 22.04, install the required packages and build with:

```bash
sudo apt update && sudo apt install -y gcc build-essential libssl-dev
make
make all
```

Set the author before running commit commands:

```bash
export PES_AUTHOR="Ahan Mysore <PES1UG24AM019>"
```

Key verification commands used during this submission:

```bash
make all
./test_objects
./test_tree
make test-integration
```

## Screenshots

| Requirement | Saved Artifact |
| --- | --- |
| 1A | `screenshots/images/1A_test_objects.png` |
| 1B | `screenshots/images/1B_objects_find.png` |
| 2A | `screenshots/images/2A_test_tree.png` |
| 2B | `screenshots/images/2B_tree_xxd.png` |
| 3A | `screenshots/images/3A_init_add_status.png` |
| 3B | `screenshots/images/3B_index_cat.png` |
| 4A | `screenshots/images/4A_pes_log.png` |
| 4B | `screenshots/images/4B_find_pes.png` |
| 4C | `screenshots/images/4C_refs_head.png` |
| Final | `screenshots/images/final_integration.png` |

### Phase 1

**1A. `./test_objects` output**

![Phase 1A](screenshots/images/1A_test_objects.png)

**1B. `find .pes/objects -type f`**

![Phase 1B](screenshots/images/1B_objects_find.png)

### Phase 2

**2A. `./test_tree` output**

![Phase 2A](screenshots/images/2A_test_tree.png)

**2B. Raw tree object with `xxd`**

![Phase 2B](screenshots/images/2B_tree_xxd.png)

### Phase 3

**3A. `pes init` -> `pes add` -> `pes status`**

![Phase 3A](screenshots/images/3A_init_add_status.png)

**3B. `cat .pes/index`**

![Phase 3B](screenshots/images/3B_index_cat.png)

### Phase 4

**4A. `pes log` with three commits**

![Phase 4A](screenshots/images/4A_pes_log.png)

**4B. `find .pes -type f | sort`**

![Phase 4B](screenshots/images/4B_find_pes.png)

**4C. `cat .pes/refs/heads/main` and `cat .pes/HEAD`**

![Phase 4C](screenshots/images/4C_refs_head.png)

### Final Integration Test

![Final Integration](screenshots/images/final_integration.png)

Raw command transcripts are also available under [`screenshots/logs`](screenshots/logs).

## Implementation Summary

### Files Completed

- `object.c`
- `tree.c`
- `index.c`
- `commit.c`
- `pes.c` received a small runtime-safety adjustment for large index allocations on this machine

### Phase 1: Object Storage

- Implemented `object_write` in `object.c` using the Git-style `<type> <size>\0<data>` format.
- Computed SHA-256 on the complete stored object and used that hash for content-addressable storage.
- Added object deduplication, shard directory creation, atomic temp-file writes, and integrity verification during reads.

### Phase 2: Tree Objects

- Implemented recursive tree construction in `tree.c`.
- Built directory trees from staged index paths, including nested directories such as `src/main.c`.
- Serialized and stored every subtree before writing the root tree object.

### Phase 3: Index

- Implemented index loading from the `.pes/index` text format.
- Implemented atomic index saves with sorting by path and `fsync()` before rename.
- Implemented file staging that hashes file contents as blobs and records metadata for status checks.

### Phase 4: Commits And History

- Implemented commit creation in `commit.c`.
- Created commits from the staged tree, linked parent commits through `HEAD`, and updated the active branch reference.
- Verified history traversal through `pes log` and through the integration sequence.

## Analysis Answers

### Q5.1

`pes checkout <branch>` would need to update `.pes/HEAD` so that it points to `ref: refs/heads/<branch>`, and if the branch does not already exist, `.pes/refs/heads/<branch>` must exist and contain the target commit hash. After resolving the target commit, checkout must read the commit object, load its root tree, recursively walk all tree entries, and rewrite the working directory so that tracked files exactly match that snapshot. Files present in the current branch but not in the target branch must be deleted, files that changed must be rewritten from blob contents, and the index should be refreshed to match the checked-out snapshot. The operation is complex because it mixes metadata updates in `.pes/` with destructive filesystem changes in the working directory, and it must refuse or carefully preserve user changes that would otherwise be overwritten.

### Q5.2

To detect a dirty-working-directory conflict, first load the index and use it as the recorded snapshot of what the user last staged or checked out. For every tracked path, compare the current filesystem metadata against the index entry: if the file is missing, its size changed, or its `mtime` changed, that file is potentially dirty. For any potentially dirty file, hash the current file contents and compare the resulting blob hash against the index hash; if they differ, the working copy has uncommitted changes. Then compare the target branch's tree against the current branch's tree. If a path is dirty in the working directory and the target branch would write a different blob at that same path, checkout must stop with a conflict. This uses only the index for the expected working copy state and the object store for the current and target snapshots.

### Q5.3

In detached `HEAD` state, `HEAD` contains a commit hash directly instead of a branch reference, so new commits still get created but no branch name moves forward with them. Each new commit points to the previous detached commit as its parent, creating a valid history chain that is reachable only through the temporary `HEAD` value. If the user later checks out a branch, those commits can become hard to find because no named ref points at them anymore. A user can recover them by creating a branch that points to the detached commit before leaving it, or by locating the commit hash from logs or reflog-style history and then writing that hash into a new branch ref.

### Q6.1

Garbage collection can use a mark-and-sweep algorithm. Start from every live reference, which includes all branch heads under `.pes/refs/heads/` and `HEAD` if it is detached. Walk every reachable commit, mark that commit hash, mark its tree hash, recursively walk every subtree reachable from that tree, and mark every blob hash encountered. A hash set is the right data structure for this because reachability checks and duplicate avoidance need to be fast even when many commits share trees and blobs. After marking finishes, scan `.pes/objects/**` and delete any object whose hash is not in the reachable set. In a repository with 100,000 commits and 50 branches, the walk would visit every reachable commit plus every distinct tree and blob reachable from them. The exact object count depends on sharing, but it is reasonable to expect at least hundreds of thousands of objects and potentially several million in a large active repository.

### Q6.2

Garbage collection is dangerous during commit creation because a commit is assembled in stages: blobs may already be written, then trees are written, and only after that does the branch ref move to the new commit. A concurrent GC that scans refs at the wrong moment could see those newly written blobs and trees as unreachable because the final commit object or branch update has not happened yet, and it could delete them before the commit finishes. That would leave the repository with a commit that points to missing objects. Real Git avoids this with techniques such as keeping temporary objects protected during writes, using reflogs and grace periods so newly created unreachable objects are not collected immediately, and coordinating GC so it does not race with in-progress object creation.

## Notes

- The main implementation files are `object.c`, `tree.c`, `index.c`, and `commit.c`.
- I also added a small local artifact workflow under [`scripts`](scripts) to render the required command screenshots into `screenshots/images`.
- The CLI wrapper `pes.c` needed a minimal safety fix so large in-memory index allocations would not crash on this machine during nested-path staging.
- The generated PNG screenshots were rendered from the saved terminal transcripts so the artifacts are reproducible.
