# NeuroKairos Synchronization for Spyglass

## Context

Multiple recording systems (ephys, cameras, behavior controllers) run on
independent clocks that drift relative to each other. pynapple-irig decodes
IRIG-H timecodes into `ClockTable` objects that map sample indices to UTC
timestamps.

**The problem:** Spyglass stores all timestamps as floats relative to
`timestamps_reference_time` (seconds since a per-session reference datetime).
It assumes all data within a session shares the same clock. Cross-pipeline
operations (ripple detection, decoding) combine data from different sources
(e.g., LFP + position) via `interpolate_to_new_time()` -- simple pandas
interpolation that assumes a shared time reference. There is no clock drift
correction or cross-device synchronization.

If two recording systems have different clocks, existing spyglass pipelines
will **silently produce wrong results** when combining their data.

## Solution: Correct Timestamps at the NWB Level

The `spyglass-neurokairos` package (separate repo) corrects timestamps
**before** data enters downstream pipelines, not after fetching. It produces
corrected AnalysisNwbfiles with UTC-domain timestamps. This makes
synchronization transparent to all downstream pipelines.

**Pipeline:**

1. **IRIGDecoding** (Computed): Decodes IRIG channel -> ClockTable. Stores the
   ClockTable in an AnalysisNwbfile for provenance.
2. **SynchronizedRecording** (Computed): Takes raw NWB data + ClockTable ->
   produces a new AnalysisNwbfile with corrected (UTC-domain) timestamps. Data
   arrays are unchanged; only the timestamps array is rewritten using
   `ClockTable.synchronize()`.
3. Downstream pipelines (LFP filtering, spike sorting, etc.) can be pointed at
   the synchronized AnalysisNwbfile instead of the raw NWB file. They see UTC
   timestamps and work correctly across recording systems.

### Why NWB-level correction is needed

Timestamps live inside NWB files (HDF5) as per-sample float arrays. Downstream
pipelines read these timestamps directly:

```python
# lfp_merge.py:51 -- LFP fetched with NWB timestamps as index
pd.DataFrame(
    nwb_lfp["lfp"].data,
    index=pd.Index(nwb_lfp["lfp"].timestamps, name="time"),
)

# ripple.py:322 -- position interpolated to LFP timestamps (assumes shared clock)
position_info = interpolate_to_new_time(position_info, interval_ripple_lfps.index)
```

If timestamps in the NWB file are corrected to UTC, these pipelines just work
with no code changes.

## Change to Spyglass Core

**One change:** Add `"neurokairos"` to the `SHARED_MODULES` list in
`src/spyglass/utils/database_settings.py` (committed in `f818adda`).

This list is checked in three places:

1. **`SpyglassMixin.__init__`** -- logs `logger.error()` if schema prefix not
   recognized
2. **`DatabaseSettings`** -- controls MySQL permission grants for `dj_user` role
3. **`dj_graph.py`** -- dependency graph traversal for cautious_delete/export

Adding `"neurokairos"` ensures no spurious errors, correct permissions, and
proper graph traversal for neurokairos tables.

### What does NOT need to change in spyglass

- **No time model changes.** Timestamps are already flexible floats; UTC values
  work fine.
- **No new mixin needed.** SpyglassMixin + AnalysisNwbfile + IntervalList
  provide everything.
- **No pipeline changes.** Downstream pipelines work transparently because they
  read timestamps from NWB files -- if those timestamps are in UTC, the
  pipelines just work.
- **No Merge table changes.** spyglass-neurokairos defines its own Merge table.

## spyglass-neurokairos Architecture (Separate Repo)

```
spyglass-neurokairos/
├── src/spyglass_neurokairos/
│   ├── __init__.py
│   ├── v1/
│   │   ├── __init__.py
│   │   ├── irig_params.py          # IRIGDecodingParams (Lookup)
│   │   ├── irig_selection.py       # IRIGDecodingSelection (Manual)
│   │   ├── irig_decoding.py        # IRIGDecoding (Computed) -- decodes IRIG -> ClockTable
│   │   ├── sync_recording.py       # SynchronizedRecording (Computed) -- rewrites timestamps
│   │   └── irig_utils.py           # ClockTable <-> NWB serialization, helpers
│   └── neurokairos_merge.py        # NeuroKairosOutput (Merge)
└── tests/
```

### Key design decisions

- **Schemas:** `neurokairos_v1`, `neurokairos_merge`
- **Dependencies:** `spyglass-neuro` + `pynapple-irig` at runtime
- **ClockTable storage:** TimeSeries (source array) + TimeSeries (reference
  array) + ScratchData (metadata JSON) in AnalysisNwbfile
- **SynchronizedRecording output:** New AnalysisNwbfile with identical data but
  UTC-corrected timestamps. This is the file downstream pipelines consume.
- **IntervalList entries:** UTC-domain valid times inserted for the synchronized
  recording

### Open questions

- How does SynchronizedRecording integrate with existing pipeline entry points?
  E.g., does spyglass's Recording table (spikesorting) accept an
  AnalysisNwbfile as input, or only raw NWB files?
- For cross-session alignment (ephys Session A + camera Session B), how are the
  synchronized recordings linked? May require a new "AlignedSession" or
  "RecordingGroup" concept.

## Testing

### For the spyglass core change (this repo)

No new tests needed. The only change is adding a string to a constant list.

```bash
pytest --no-docker --no-dlc -m 'not slow and not very_slow' tests/
```

### For spyglass-neurokairos (separate repo)

**Unit tests:**

- ClockTable NWB roundtrip serialization
- Timestamp rewriting correctness (known ClockTable + known input timestamps ->
  expected UTC timestamps)
- Valid times derivation from ClockTable

**Integration tests:**

- Full pipeline: params -> selection -> `IRIGDecoding.populate()` ->
  `SynchronizedRecording.populate()` -> verify corrected AnalysisNwbfile
  timestamps
- Verify downstream pipeline (e.g., LFP filtering) produces correct results
  when fed synchronized vs. unsynchronized data

**Test data:** Synthetic ClockTable fixtures (60 entries, known
source/reference values). No need for full IRIG signal decoding -- that's tested
in pynapple-irig.
