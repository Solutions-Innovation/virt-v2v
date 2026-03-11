# Enhancement Proposal: Skip fstrim in virt-v2v-in-place

- **Date**: 2026-03-06
- **Branch**: `skip-fstrim-in-place` (based on `v2.9.10`)
- **Target**: Fedora 44 package `virt-v2v-2.9.10-1.fc44`

## Problem Statement

When running `virt-v2v-in-place` on block storage (e.g., `/dev/cdi-block-volume`
in a containerized environment), the conversion takes an excessively long time
due to the `fstrim` step. This step sends TRIM/UNMAP commands to the underlying
storage for every free extent on every filesystem.

On block volumes with fragmented free space, each small TRIM request requires a
round-trip to the storage backend. With many small extents and poor random I/O
performance, this results in conversion times significantly longer than
necessary.

## Background

The `do_fstrim` function in `convert/convert.ml` runs fstrim on all guest
filesystems after conversion. The code comment at lines 155-158 explains the
rationale:

> "Doing fstrim on all the filesystems reduces the transfer size because
> unused blocks are marked in the overlay and thus do not have to be copied."

This optimization makes sense for regular `virt-v2v`, which uses an overlay and
copies data to a target. However, **virt-v2v-in-place has no overlay and no
copy**. The fstrim step only sends UNMAP commands to the real storage, which:

- Provides **no conversion benefit** (nothing is being transferred)
- May marginally reclaim thin-provisioned space, but this is a side effect
- Is **extremely slow** on block volumes with fragmented free space

### Current Implementation

The fstrim call in `convert/convert.ml`:

```ocaml
(* Doing fstrim on all the filesystems reduces the transfer size
 * because unused blocks are marked in the overlay and thus do
 * not have to be copied.
 *)
message (f_"Mapping filesystem data to avoid copying unused and blank areas");
do_fstrim g inspect;
```

The `do_fstrim` function mounts each filesystem with `discard` option and calls
`g#fstrim "/"` with no optional parameters, so it trims everything down to the
filesystem's minimum extent size.

## Proposed Solution

Add a `no_trim : bool` field to the shared `options` record in
`convert/convert.ml` and expose a `--no-trim` CLI flag in `virt-v2v-in-place`.

When `no_trim` is true, the `do_fstrim` step is skipped entirely.

### Scope

- **`virt-v2v-in-place`**: Adds `--no-trim` CLI flag (default: fstrim enabled
  for backward compatibility)
- **`virt-v2v`**: Unaffected; always sets `no_trim = false`

### Files Modified

| File | Change |
|------|--------|
| `convert/convert.ml` | Add `no_trim : bool` to `options` type; guard `do_fstrim` call |
| `convert/convert.mli` | Mirror `no_trim : bool` in interface type |
| `in-place/in_place.ml` | Add `--no-trim` CLI flag; pass to `conv_options` |
| `v2v/v2v.ml` | Set `no_trim = false` in `conv_options` (record completeness) |
| `in-place/virt-v2v-in-place.pod` | Document the `--no-trim` option |

### Implementation Detail

The `options` record follows the existing pattern used by `keep_serial_console`:

```ocaml
type options = {
  block_driver : guestcaps_block_type;
  keep_serial_console : bool;
  no_trim : bool;                       (* new *)
  ks : key_store;
  ...
}
```

The guard in `convert.ml`:

```ocaml
if not options.no_trim then (
  message (f_"Mapping filesystem data to avoid copying unused and blank areas");
  do_fstrim g inspect
) else
  debug "info: skipping fstrim (--no-trim flag)";
```

## Testing

1. **Compilation**: `make` — OCaml's type system catches any missing record
   fields at compile time.
2. **CLI check**: `virt-v2v-in-place --help` — confirm `--no-trim` appears.
3. **Flag active**: `virt-v2v-in-place --no-trim -v -i disk <test-image>` —
   verify fstrim step is skipped in verbose output.
4. **Default behavior**: `virt-v2v-in-place -v -i disk <test-image>` — verify
   fstrim still runs when `--no-trim` is not passed.
5. **Pod validation**: `podcheck.pl` — verify documentation is valid.

## Future Considerations

- **`minimumfreeextent` parameter**: The libguestfs `g#fstrim` API supports an
  optional `~minimumfreeextent` parameter (maps to `FITRIM`'s `minlen` field).
  Setting this to a large value (e.g., 1 MB) would skip trimming small free
  extents, reducing the number of TRIM operations while still trimming large
  contiguous free areas. This could be a useful middle-ground option for regular
  `virt-v2v` users who want faster fstrim without completely disabling it.
- **Auto-detection**: A future enhancement could detect in-place mode at the
  convert level and automatically skip fstrim, rather than requiring a CLI flag.
  However, the explicit flag approach is simpler and more transparent.
