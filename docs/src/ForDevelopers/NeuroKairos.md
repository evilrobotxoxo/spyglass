# NeuroKairos Synchronization for Spyglass

## Context

Multiple recording systems (ephys, cameras, behavior controllers) run on
independent clocks that drift relative to each other. pynapple-irig solves
this by decoding IRIG-H timecodes into `ClockTable` objects that map sample
indices to UTC timestamps. The pynapple core library needed modifications
(`time_origin`, `align_to`) to support this. The question: **does spyglass
core need similar modifications, or can a separate `spyglass-neurokairos`
package work with spyglass as-is?**

## Finding: Spyglass Core Needs One Small Change

An external `spyglass-neurokairos` package can already:

- Define DataJoint schemas referencing spyglass tables (Session, Nwbfile,
  IntervalList) as foreign keys
- Use `SpyglassMixin` on all its tables
- Create `AnalysisNwbfile` entries to store ClockTable data in NWB format
- Insert into `IntervalList` for synchronized valid times

**The one issue: `SHARED_MODULES` gating.** Spyglass has a hardcoded list of
recognized schema prefixes in `src/spyglass/utils/database_settings.py:13`.
This list is checked in three places:

1. **`SpyglassMixin.__init__`** (`utils/dj_mixin.py:74-83`) -- logs
   `logger.error()` if schema prefix not recognized. Not a hard failure, but
   noisy.
2. **`DatabaseSettings`** (`utils/database_settings.py:81`) -- controls MySQL
   permission grants. The `dj_user` role won't automatically get write access
   to `neurokairos_*` databases.
3. **`dj_graph.py:346`** -- dependency graph traversal treats unrecognized
   prefixes as "out of scope", which could affect `cautious_delete` and export
   operations on neurokairos tables.

### Change to spyglass

Add `"neurokairos"` to the `SHARED_MODULES` list in
`src/spyglass/utils/database_settings.py`:

```python
SHARED_MODULES = [
    "behavior",
    "common",
    "decoding",
    "lfp",
    "linearization",
    "mua",
    "neurokairos",   # <-- added
    "position",
    "ripple",
    "sharing",
    "spikesorting",
]
```

This ensures:

- No spurious error logs when neurokairos tables are instantiated
- Database permissions are automatically granted to `dj_user` role
- Graph traversal includes neurokairos tables for delete/export operations

### What does NOT need to change in spyglass

- **No time model changes needed.** Unlike pynapple (which needed
  `time_origin` on every data container), spyglass stores timestamps in
  IntervalList and NWB files -- both are flexible enough to hold UTC-domain
  values. The conversion between session-relative and UTC time is the
  responsibility of spyglass-neurokairos, not spyglass core.
- **No new mixin needed.** SpyglassMixin + AnalysisNwbfile + IntervalList
  already provide everything.
- **No Merge table changes.** spyglass-neurokairos defines its own Merge table
  (can't dynamically add Parts to existing ones, but doesn't need to -- IRIG
  output is its own pipeline, not a variant of LFP/position).

## spyglass-neurokairos Architecture (Separate Repo)

The external package is structured as follows. This is NOT part of the spyglass
repo -- it lives in a separate repo.

```
spyglass-neurokairos/
├── src/spyglass_neurokairos/
│   ├── __init__.py
│   ├── v1/
│   │   ├── __init__.py
│   │   ├── irig_params.py        # IRIGDecodingParams (Lookup)
│   │   ├── irig_selection.py     # IRIGDecodingSelection (Manual)
│   │   ├── irig_decoding.py      # IRIGDecoding (Computed)
│   │   └── irig_utils.py         # ClockTable <-> NWB serialization, helpers
│   └── neurokairos_merge.py      # NeuroKairosOutput (Merge)
└── tests/
```

### Key design decisions

- **Depends on both `spyglass-neuro` and `pynapple-irig`** at runtime
- **Schemas named `neurokairos_v1`, `neurokairos_merge`** -- hence the
  SHARED_MODULES addition
- **IRIGDecoding.make()** calls pynapple-irig decoders, serializes ClockTable
  into AnalysisNwbfile, creates IntervalList entries
- **ClockTable stored in AnalysisNwbfile** as TimeSeries (source array) +
  TimeSeries (reference array) + ScratchData (metadata JSON)

## Testing Strategy

### For the spyglass core change (this repo)

No new tests needed -- this is a one-line addition to a constant list. Existing
tests continue to pass.

Verification:

```bash
pytest --no-docker --no-dlc -m 'not slow and not very_slow' tests/
```

### For spyglass-neurokairos (separate repo)

**Unit tests** (no database needed):

- `clock_table_to_nwb()` / `clock_table_from_nwb()` roundtrip -- verify
  ClockTable survives NWB serialization
- `derive_valid_times()` -- verify correct intervals from various ClockTable
  shapes (gaps, concatenated files)
- Parameter validation

**Integration tests** (require Docker MySQL, following spyglass test patterns):

- Full pipeline: insert params -> insert selection ->
  `IRIGDecoding.populate()` -> verify IntervalList + AnalysisNwbfile entries
- `task_mode='load'` with pre-computed ClockTable
- `task_mode='trigger'` with synthetic IRIG data (reuse pynapple-irig's test
  data generators)
- `fetch1_clock_table()` convenience method roundtrip
- Merge table insertion and retrieval

**Test data**: Generate synthetic ClockTable fixtures (60 entries, known
source/reference values) -- no need for full IRIG signal decoding in
spyglass-neurokairos tests since that's tested in pynapple-irig.
