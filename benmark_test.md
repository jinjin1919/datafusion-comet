| Group | Range of change (`after refactor` vs `before`) |
|---|---|
| `matched_maps_hash_only` | −2% to **+165%** |
| `matched_maps_normalize_hash` | **−51%** to **+213%** |
| `multi_entry_int_maps_*` | −64% to +82% |
| `regression_maps_*` | −58% to +170% |
| `murmur3/*` | +8% to +22% |
| `matched_maps_hash_buffer_seed_reset` | −61% |

These swings are run-to-run noise, not an effect of the refactor. The `_forward` and `_reversed`
variants of the same workload move in opposite directions, and the assembly check below shows
the refactor compiles to identical machine code.

## Assembly evidence: the refactor produces identical machine code

Compared `962db532f` (before) with `87c95035c` (refactor), crate `datafusion-comet-spark-expr`,
release profile (`codegen-units = 1`), rustc 1.99.1, macOS. Assembly is per-crate, before thin
LTO.

| Function | Before | After | Result |
|---|---|---|---|
| `create_murmur3_hashes` (main body) | 29,250 instructions | 29,250 instructions | identical |
| its closures | 26 | 26 | all 26 match by content |
| `create_xxhash64_hashes` (main body) | 26,978 instructions | 26,978 instructions | identical |
| its closures | 26 | 26 | all 26 match by content |

Raw check, with only `.loc` debug line markers removed and nothing else normalised:

- `create_murmur3_hashes`: 30,380 lines, `diff` empty.
- `create_xxhash64_hashes`: 28,043 lines, `diff` empty.

The only raw differences are `.loc` line numbers, because the Map arm now sits about 41 lines
further down `utils.rs`. `hash_map_entries!` is a macro, so it expands in place, and its
closures (`|v| v`, `|v: i32| v.to_le_bytes()`) are fully inlined.

Control: diffing `create_murmur3_hashes` against `create_xxhash64_hashes` gives 44,694 diff
lines, so the comparison does detect real differences.

What this shows: the refactor gives the same output and performance as the old code. It matches
Spark because the old code did, which `test_specialised_map_pairs_match_spark` checks directly.

### Reproduce

Build each commit in its own worktree. Use a separate `CARGO_TARGET_DIR` per commit, or run
`cargo clean --release -p datafusion-comet-spark-expr` between builds. With a shared target dir,
cargo reuses the first build and you get a false "identical".

```shell
git worktree add --detach ../wt-before 962db532f
git worktree add --detach ../wt-after  87c95035c
cargo rustc --release --manifest-path ../wt-before/native/Cargo.toml \
  -p datafusion-comet-spark-expr --lib -- --emit=asm
# copy target/release/deps/datafusion_comet_spark_expr-*.s to before.s,
# clean the crate, then repeat for ../wt-after and save after.s
```

Then extract one function, drop `.loc` lines, and diff:

```shell
sym='__RNvNtNtCs8HUL0KDG94a_27datafusion_comet_spark_expr10hash_funcs8xxhash6422create_xxhash64_hashes'
for v in before after; do
  awk -v s="$sym" '/^_[A-Za-z0-9_$.]+:/{inf=($0 ~ s"(:|$)")} inf' $v.s | grep -v '^\s*\.loc' > xx-$v.txt
done
diff xx-before.txt xx-after.txt && echo IDENTICAL
```

The crate hash in `sym` (`Cs8HUL0KDG94a`) may differ on another machine. Find it with
`grep -o '__RNv[^:]*create_xxhash64_hashes' before.s | head -1`.

Files from this run (temporary session folder; copy them to keep them):
`/private/tmp/claude-501/-Users-jinxu-datafusion-comet/f0e0371e-4b7c-4590-93f9-e446eb067f23/scratchpad/asm/`
(`before.s`, `after.s`, `raw-before-noloc.txt`, `raw-after-noloc.txt`, `xx-before.txt`,
`xx-after.txt`, `extract.py`, `compare.py`).



