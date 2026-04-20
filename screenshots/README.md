# Screenshot Capture Guide

This folder stores the final screenshots required for submission.

## Phase 1

### 1A: `./test_objects` passing

Run:

```bash
make test_objects
./test_objects
```

Capture the terminal showing:

- the successful build
- `PASS: blob storage`
- `PASS: deduplication`
- `PASS: integrity check`
- `All Phase 1 tests passed.`

Suggested filename:

- `phase1_1A_test_objects.png`

### 1B: object-store sharding

Run immediately after `1A`, before cleaning:

```bash
find .pes/objects -type f
```

Capture the terminal showing the sharded object paths.

Suggested filename:

- `phase1_1B_objects_listing.png`

## Phase 2

### 2A: `./test_tree` passing

Run after Phase 2 code is complete:

```bash
make test_tree
./test_tree
```

Suggested filename:

- `phase2_2A_test_tree.png`

### 2B: raw tree object dump

Run after `2A` and after tree objects have been created:

```bash
find .pes/objects -type f | head
xxd .pes/objects/XX/YYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYY | head -20
```

Replace `XX/YYYY...` with the path of a real tree object from your run.

Suggested filename:

- `phase2_2B_tree_xxd.png`

## Phase 3

### 3A: init, add, status

Run after Phase 3 code is complete:

```bash
make pes
./pes init
echo "hello" > file1.txt
echo "world" > file2.txt
./pes add file1.txt file2.txt
./pes status
```

Suggested filename:

- `phase3_3A_status_sequence.png`

### 3B: index file contents

Run after `3A`:

```bash
cat .pes/index
```

Suggested filename:

- `phase3_3B_index_contents.png`

## Phase 4

### 4A: commit log

Run after Phase 4 code is complete:

```bash
./pes init
echo "Hello" > hello.txt
./pes add hello.txt
./pes commit -m "Initial commit"

echo "World" >> hello.txt
./pes add hello.txt
./pes commit -m "Add world"

echo "Goodbye" > bye.txt
./pes add bye.txt
./pes commit -m "Add farewell"

./pes log
```

Suggested filename:

- `phase4_4A_log.png`

### 4B: repository files

Run after `4A`:

```bash
find .pes -type f | sort
```

Suggested filename:

- `phase4_4B_pes_files.png`

### 4C: reference chain

Run after `4A`:

```bash
cat .pes/refs/heads/main
cat .pes/HEAD
```

Suggested filename:

- `phase4_4C_refs.png`

## Final Integration Test

Run after the whole project is complete:

```bash
make test-integration
```

Suggested filename:

- `final_integration_test.png`
