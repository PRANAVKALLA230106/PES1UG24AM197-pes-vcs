# Phase 5 & 6: Analysis Answers

## Branching and Checkout

### Q5.1: Implementing `pes checkout <branch>`

A branch in Git-like systems is simply a reference (file) pointing to a commit hash. So, implementing `pes checkout <branch>` involves two main components: updating internal metadata and synchronizing the working directory.

**Changes in `.pes/`:**

* Update `.pes/HEAD` to point to the new branch reference (e.g., `refs/heads/<branch>`).
* Read the commit hash stored in `.pes/refs/heads/<branch>`.
* Load the corresponding commit object from the object store.
* Extract the tree hash from the commit.
* Update the index (`.pes/index`) to match the tree of the target commit.

**Changes in Working Directory:**

* Compare the current working directory with the target tree.
* Delete files that exist in the current tree but not in the target tree.
* Create/update files to match blobs in the target tree.
* Ensure file contents and modes match exactly.

**Why this is complex:**

* You must carefully handle file differences (additions, deletions, modifications).
* Uncommitted changes can conflict with the target branch.
* Efficiently updating only changed files (not rewriting everything) requires diffing trees.
* Edge cases like file ↔ directory conflicts complicate implementation.

---

### Q5.2: Detecting a Dirty Working Directory Conflict

To detect conflicts during checkout, you must ensure no uncommitted changes would be overwritten.

**Available Data:**

* Index (`.pes/index`): Tracks last staged version of files (path → blob hash).
* Object store: Contains committed versions (via tree and blobs).

**Steps to Detect Conflict:**

1. For each tracked file in the index:

   * Compute the current working directory file hash.
   * Compare it with the hash stored in the index.
   * If different → file is modified (dirty).

2. For each dirty file:

   * Look at the target branch’s tree.
   * If the same file path exists and its blob hash differs from the current branch:

     * This is a **conflict**.

3. If any such conflict exists:

   * Abort checkout with an error.

**Key Idea:**
A conflict occurs when:

* The working directory has uncommitted changes **AND**
* The target branch would overwrite those changes.

---

### Q5.3: Detached HEAD

In a detached HEAD state:

* `.pes/HEAD` contains a commit hash directly instead of pointing to a branch.

**What happens when you commit:**

* New commits are created normally.
* However, no branch points to them.
* These commits are only reachable via HEAD temporarily.

**Risk:**

* If you switch to another branch, these commits may become unreachable.
* Eventually, garbage collection may delete them.

**Recovery:**

* If the commit hash is known:

  * Create a new branch pointing to it:

    ```
    pes branch recovered <commit-hash>
    ```
* Alternatively:

  * Use a reflog (if implemented) to find recent HEAD states.

---

## Garbage Collection and Space Reclamation

### Q6.1: Finding and Deleting Unreachable Objects

**Goal:**
Remove objects (blobs, trees, commits) not reachable from any branch.

**Algorithm (Mark-and-Sweep):**

1. **Mark Phase:**

   * Start from all branch heads (`.pes/refs/heads/*`).
   * For each commit:

     * Mark it as reachable.
     * Traverse its parent commits.
     * Traverse its tree:

       * Mark all trees and blobs recursively.

2. **Data Structure:**

   * Use a **hash set** (e.g., `unordered_set`) to store reachable object hashes.
   * This ensures O(1) lookup.

3. **Sweep Phase:**

   * Iterate through all objects in `.pes/objects/`.
   * If an object is NOT in the reachable set:

     * Delete it.

**Estimate:**

* 100,000 commits, 50 branches:

  * Worst case: all commits reachable.
  * Each commit references:

    * 1 tree
    * ~10–100 blobs (average)
* Rough estimate:

  * ~100k commits
  * ~100k trees
  * ~1M–5M blobs

So, you may traverse **millions of objects**, but each only once due to hashing.

---

### Q6.2: Why Concurrent GC is Dangerous

Running garbage collection during a commit can cause race conditions.

**Race Condition Scenario:**

1. A commit operation:

   * Writes a new blob/tree object.
   * Has NOT yet updated the branch reference.

2. GC starts:

   * Scans reachable objects from branch heads.
   * The new object is NOT yet referenced → considered unreachable.

3. GC deletes the object.

4. Commit finishes:

   * Updates branch to point to a commit referencing the deleted object.

**Result:**

* Repository becomes corrupted (missing objects).

---

**How Git Avoids This:**

Real Git uses multiple safety mechanisms:

1. **Object Creation First, Reference Later:**

   * Objects are written before references are updated.

2. **Atomic Reference Updates:**

   * Branch updates are atomic (no partial state).

3. **Grace Period (Loose Objects):**

   * New objects are not immediately eligible for deletion.

4. **Locking Mechanisms:**

   * Prevent GC from running during critical operations.

5. **Reflog Protection:**

   * Recently referenced commits are preserved.

**Key Insight:**
GC must never delete objects that *might soon become reachable*.

---
