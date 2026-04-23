# PES-VCS Lab Report

This README contains the required report content, screenshot references, analysis answers, and exact commands to reproduce the screenshots manually.

## Phase 1: Object Storage Foundation

### Screenshot 1A and 1B

Current evidence is captured in a combined screenshot that includes:

- `./test_objects` showing all Phase 1 tests passing
- `find .pes/objects -type f` showing the sharded object-store layout

File:

![`screenshots/phase1_combined_1A_1B.png`](screenshots/phase1_combined_1A_1B.png)

Commands used:

```bash
make test_objects
./test_objects
find .pes/objects -type f
```

Notes:

- This screenshot currently combines the outputs requested for `1A` and `1B`.
- If separate captures are needed later, they can be added alongside this one.

## Phase 2: Tree Objects

### Screenshot 2A

`./test_tree` output showing all Phase 2 tests passing:

![`screenshots/phase2_2A_test_tree.png`](screenshots/phase2_2A_test_tree.png)

Commands used:

```bash
export PES_AUTHOR="Devansh Gupta <PES1UG24CS145>"
make test_tree
./test_tree
```

### Screenshot 2B

Raw tree object dump:

![`screenshots/phase2_2B_tree_xxd.png`](screenshots/phase2_2B_tree_xxd.png)

Supporting helper-command capture:

![`screenshots/phase2_2B_tree_helper_command.png`](screenshots/phase2_2B_tree_helper_command.png)

Commands used:

```bash
export PES_AUTHOR="Devansh Gupta <PES1UG24CS145>"
make test_tree

gcc -Wall -Wextra -O2 -I. -x c - -x none object.o tree.o -lcrypto -o /tmp/tree_path_helper <<'EOF'
#include "pes.h"
#include "tree.h"
#include <stdio.h>
#include <stdlib.h>
#include <sys/stat.h>

int object_write(ObjectType type, const void *data, size_t len, ObjectID *id_out);
void object_path(const ObjectID *id, char *path_out, size_t path_size);

int main(void) {
    system("rm -rf .pes");
    mkdir(".pes", 0755);
    mkdir(".pes/objects", 0755);
    mkdir(".pes/refs", 0755);
    mkdir(".pes/refs/heads", 0755);

    ObjectID a, b, root;
    char ah[HASH_HEX_SIZE + 1], bh[HASH_HEX_SIZE + 1], path[512];

    object_write(OBJ_BLOB, "hello\n", 6, &a);
    object_write(OBJ_BLOB, "world\n", 6, &b);
    hash_to_hex(&a, ah);
    hash_to_hex(&b, bh);

    FILE *f = fopen(".pes/index", "w");
    fprintf(f, "100644 %s 0 6 README.md\n", ah);
    fprintf(f, "100644 %s 0 6 src/main.c\n", bh);
    fclose(f);

    tree_from_index(&root);
    object_path(&root, path, sizeof(path));
    puts(path);
    return 0;
}
EOF

TREE_PATH=$(/tmp/tree_path_helper)
echo "$TREE_PATH"
xxd "$TREE_PATH" | head -20
```

## Phase 3: The Index

### Screenshot 3A and 3B

Current evidence is captured in one combined screenshot that includes:

- `pes init`
- `pes add file1.txt file2.txt`
- `pes status`
- `cat .pes/index`

File:

![`screenshots/phase3_combined_3A_3B.png`](screenshots/phase3_combined_3A_3B.png)

Commands used:

```bash
export PES_AUTHOR="Devansh Gupta <PES1UG24CS145>"
make pes
PES_BIN="$(pwd)/pes"
WORKDIR="$(mktemp -d)"
cd "$WORKDIR"

"$PES_BIN" init
echo "hello" > file1.txt
echo "world" > file2.txt
"$PES_BIN" add file1.txt file2.txt
"$PES_BIN" status
cat .pes/index
```

## Phase 4: Commits and History

### Screenshot 4A, 4B and 4C

Current evidence is captured across two screenshots. Together they include:

- `pes log`
- `find .pes -type f | sort`
- `cat .pes/refs/heads/main`
- `cat .pes/HEAD`

Files:

![`screenshots/phase4_combined_4A_4B_4C_part1.png`](screenshots/phase4_combined_4A_4B_4C_part1.png)
![`screenshots/phase4_combined_4A_4B_4C_part2.png`](screenshots/phase4_combined_4A_4B_4C_part2.png)

Commands used:

```bash
export PES_AUTHOR="Devansh Gupta <PES1UG24CS145>"
make pes
PES_BIN="$(pwd)/pes"
WORKDIR="$(mktemp -d)"
cd "$WORKDIR"

"$PES_BIN" init
echo "Hello" > hello.txt
"$PES_BIN" add hello.txt
"$PES_BIN" commit -m "Initial commit"

echo "World" >> hello.txt
"$PES_BIN" add hello.txt
"$PES_BIN" commit -m "Add world"

echo "Goodbye" > bye.txt
"$PES_BIN" add bye.txt
"$PES_BIN" commit -m "Add farewell"

"$PES_BIN" log
find .pes -type f | sort
cat .pes/refs/heads/main
cat .pes/HEAD
```

## Phase 5: Branching and Checkout

### Q5.1

`pes checkout <branch>` would need to update both metadata and the working tree.

Files inside `.pes/` that would change:

- `.pes/HEAD` if checkout is moving to a branch name and HEAD should point to `ref: refs/heads/<branch>`
- `.pes/index` so the staged snapshot matches the checked-out commit's tree

Files inside `.pes/` that would be read:

- `.pes/refs/heads/<branch>` to obtain the target commit hash
- the commit object for that hash
- the root tree and all referenced subtrees/blobs reachable from that commit

Working-directory actions:

- remove tracked files that exist in the current branch but not in the target branch
- create files that exist in the target branch but not in the current branch
- overwrite tracked files whose blob hashes differ
- recreate directory structure as implied by tree objects
- refresh the index entries so they match the checked-out snapshot

This is complex because checkout is not just changing one pointer. It is a coordinated filesystem rewrite that must compare two snapshots, preserve untracked files safely, detect conflicts with local modifications, and keep HEAD, refs, index, and the working tree mutually consistent.

### Q5.2

To detect a dirty-working-directory conflict using only the index and object store:

1. Read the current index entry for each tracked path.
2. For each indexed path, compare the current working-file metadata against the cached index metadata (`mtime` and `size`). If either differs, the file is potentially dirty.
3. For potentially dirty files, re-hash the working file as a blob and compare that blob hash with the hash stored in the index.
4. Resolve the target branch's commit, then walk its tree to determine the blob hash that branch wants for the same path.
5. Refuse checkout if all of the following are true:
   - the working file differs from the index
   - the target branch has a different blob hash for that path than the current index/HEAD snapshot
   - checkout would therefore overwrite the user's uncommitted content

In short, the index tells us what the user last staged/checked out, the object store tells us what each branch snapshot contains, and a working-file re-hash confirms whether the user's current file has drifted.

### Q5.3

In detached HEAD state, HEAD stores a commit hash directly instead of a branch ref. New commits still work, but each new commit advances only HEAD itself, not any branch name. That means those commits can become hard to find once the user checks out another branch, because no normal branch ref points to them anymore.

The user can recover those commits by:

- creating a new branch at the detached commit before leaving it
- copying the commit hash from `pes log` and creating a ref later
- keeping a manual note of the commit hash and reattaching a branch to it

The commits are not immediately lost; they are just unreferenced by a friendly name.

## Phase 6: Garbage Collection and Space Reclamation

### Q6.1

A simple mark-and-sweep garbage collector would work:

1. Start from every branch ref in `.pes/refs/heads/`.
2. Mark each referenced commit hash as reachable.
3. For each reachable commit:
   - mark its tree hash
   - if it has a parent, mark the parent commit and continue walking history
4. For each reachable tree:
   - parse every tree entry
   - mark each referenced blob or subtree hash
   - recurse into subtree hashes
5. After marking is complete, scan `.pes/objects/` and delete every object whose hash is not in the reachable set.

The right data structure for the reachable set is a hash table, or in C terms a hash-set-like structure, because membership checks must be fast when traversing large object graphs.

For a repository with 100,000 commits and 50 branches, the number of visited commits is usually much closer to 100,000 than to `100,000 * 50`, because many branches share history. The collector would visit:

- each unique reachable commit once
- each unique reachable tree once
- each unique reachable blob once

The exact total depends on tree/blob sharing, but the traversal cost is proportional to the number of unique reachable objects, not the number of refs times the full history length.

### Q6.2

Running GC concurrently with commit creation is dangerous because commit creation is multi-step:

1. write blobs and trees
2. write the new commit object
3. update the branch ref to point to that new commit

A race can happen if GC scans refs after step 1 or step 2 but before step 3. At that instant, the new blobs/tree/commit may exist on disk but still be unreachable from any branch ref. GC could incorrectly classify them as garbage and delete them. Then the concurrent commit finishes and updates the branch ref to point at objects that no longer exist.

Git avoids this kind of race by being conservative:

- it does not aggressively delete very recent unreachable objects immediately
- it uses additional safety mechanisms such as reflogs and grace periods
- it typically performs garbage collection in controlled conditions rather than blindly deleting anything currently unreachable at one instant

## Final Integration Test

Final end-to-end integration evidence is captured across three screenshots:

![`screenshots/final_integration_test_part1.png`](screenshots/final_integration_test_part1.png)
![`screenshots/final_integration_test_part2.png`](screenshots/final_integration_test_part2.png)
![`screenshots/final_integration_test_part3.png`](screenshots/final_integration_test_part3.png)

Commands used:

```bash
cd /home/d3va/Downloads/OS/OS_Unit4_Orange/os-u4-orange-problem
export PES_AUTHOR="Devansh Gupta <PES1UG24CS145>"
make test-integration
```

## Exact Screenshot Commands

Use the exact commands below when you want to recapture the screenshots manually.

Set the author once at the start of the terminal session:

```bash
export PES_AUTHOR="Devansh Gupta <PES1UG24CS145>"
```

### 1A

```bash
cd /home/d3va/Downloads/OS/OS_Unit4_Orange/os-u4-orange-problem
make clean
make test_objects
./test_objects
```

### 1B

Run this immediately after `1A`:

```bash
find .pes/objects -type f | sort
```

### 2A

```bash
cd /home/d3va/Downloads/OS/OS_Unit4_Orange/os-u4-orange-problem
make clean
make test_tree
./test_tree
```

### 2B

```bash
cd /home/d3va/Downloads/OS/OS_Unit4_Orange/os-u4-orange-problem
make clean
make test_tree
gcc -Wall -Wextra -O2 -I. -x c - -x none object.o tree.o -lcrypto -o /tmp/tree_path_helper <<'EOF'
#include "pes.h"
#include "tree.h"
#include <stdio.h>
#include <stdlib.h>
#include <sys/stat.h>

int object_write(ObjectType type, const void *data, size_t len, ObjectID *id_out);
void object_path(const ObjectID *id, char *path_out, size_t path_size);

int main(void) {
    system("rm -rf .pes");
    mkdir(".pes", 0755);
    mkdir(".pes/objects", 0755);
    mkdir(".pes/refs", 0755);
    mkdir(".pes/refs/heads", 0755);

    ObjectID a, b, root;
    char ah[HASH_HEX_SIZE + 1], bh[HASH_HEX_SIZE + 1], path[512];

    object_write(OBJ_BLOB, "hello\n", 6, &a);
    object_write(OBJ_BLOB, "world\n", 6, &b);
    hash_to_hex(&a, ah);
    hash_to_hex(&b, bh);

    FILE *f = fopen(".pes/index", "w");
    fprintf(f, "100644 %s 0 6 README.md\n", ah);
    fprintf(f, "100644 %s 0 6 src/main.c\n", bh);
    fclose(f);

    tree_from_index(&root);
    object_path(&root, path, sizeof(path));
    puts(path);
    return 0;
}
EOF
TREE_PATH=$(/tmp/tree_path_helper)
echo "$TREE_PATH"
xxd "$TREE_PATH" | head -20
```

### 3A

```bash
cd /home/d3va/Downloads/OS/OS_Unit4_Orange/os-u4-orange-problem
make clean
make pes
PES_ROOT="/home/d3va/Downloads/OS/OS_Unit4_Orange/os-u4-orange-problem"
WORKDIR="$(mktemp -d)"
cd "$WORKDIR"
"$PES_ROOT/pes" init
echo "hello" > file1.txt
echo "world" > file2.txt
"$PES_ROOT/pes" add file1.txt file2.txt
"$PES_ROOT/pes" status
```

### 3B

Run this immediately after `3A`:

```bash
cat .pes/index
```

### 4A

```bash
cd /home/d3va/Downloads/OS/OS_Unit4_Orange/os-u4-orange-problem
make clean
make pes
PES_ROOT="/home/d3va/Downloads/OS/OS_Unit4_Orange/os-u4-orange-problem"
WORKDIR="$(mktemp -d)"
cd "$WORKDIR"
"$PES_ROOT/pes" init
echo "Hello" > hello.txt
"$PES_ROOT/pes" add hello.txt
"$PES_ROOT/pes" commit -m "Initial commit"
echo "World" >> hello.txt
"$PES_ROOT/pes" add hello.txt
"$PES_ROOT/pes" commit -m "Add world"
echo "Goodbye" > bye.txt
"$PES_ROOT/pes" add bye.txt
"$PES_ROOT/pes" commit -m "Add farewell"
"$PES_ROOT/pes" log
```

### 4B

Run this immediately after `4A`:

```bash
find .pes -type f | sort
```

### 4C

Run this immediately after `4A`:

```bash
cat .pes/refs/heads/main
cat .pes/HEAD
```

### Final Integration Test

```bash
cd /home/d3va/Downloads/OS/OS_Unit4_Orange/os-u4-orange-problem
make clean
make all
make test-integration
```
