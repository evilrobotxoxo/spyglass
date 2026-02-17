# spyglass-neurokairos Bootstrap Context

This document provides architectural context and code patterns for building the
`spyglass-neurokairos` package. It is intended to be copied into the new repo
as a `CLAUDE.md` or similar context file.

---

## The Synchronization Problem

Multiple recording systems (ephys, cameras, behavior controllers) run on
independent clocks that drift relative to each other. `pynapple-irig` decodes
IRIG-H timecodes into `ClockTable` objects that map sample indices to UTC
timestamps.

In pynapple, the workflow is:

```python
clock = decode_irig(irig_channel)       # -> ClockTable
lfp_utc = clock.synchronize(lfp_tsd)    # device-time -> UTC
pos_utc = clock.synchronize(pos_tsd)    # device-time -> UTC
aligned = lfp_utc.align_to(pos_utc)     # both in UTC, combine
```

In spyglass, timestamps live inside NWB files (HDF5) as per-sample float arrays
relative to `timestamps_reference_time` (a per-session reference datetime).
Spyglass assumes all data within a session shares the same clock. Cross-pipeline
operations combine data from different sources via simple interpolation:

```python
# lfp_merge.py:51 -- LFP fetched with NWB timestamps as index
pd.DataFrame(
    nwb_lfp["lfp"].data,
    index=pd.Index(nwb_lfp["lfp"].timestamps, name="time"),
)

# ripple.py:322 -- position interpolated to LFP timestamps (assumes shared clock)
position_info = interpolate_to_new_time(position_info, interval_ripple_lfps.index)
```

If two recording systems have different clocks, these pipelines silently produce
wrong results. Synchronization must happen **before** data enters downstream
pipelines -- at the NWB timestamp level.

## Approach: NWB Timestamp Rewriting

`spyglass-neurokairos` produces corrected AnalysisNwbfiles with UTC-domain
timestamps. Data arrays are unchanged; only the timestamps array is rewritten
using `ClockTable.synchronize()`. Downstream pipelines see UTC timestamps and
work correctly across recording systems with no code changes.

**Pipeline:**

1. **IRIGDecoding** (Computed): Decodes IRIG channel -> ClockTable. Stores the
   ClockTable in an AnalysisNwbfile for provenance.
2. **SynchronizedRecording** (Computed): Takes raw NWB data + ClockTable ->
   produces a new AnalysisNwbfile with corrected timestamps.
3. Downstream pipelines consume the synchronized AnalysisNwbfile.

---

## Spyglass APIs and Patterns

### AnalysisNwbfile

The `AnalysisNwbfile` table stores NWB files containing analysis results. Its
methods live on the `AnalysisMixin` base class
(`src/spyglass/utils/mixins/analysis.py`).

**`create(nwb_file_name)` -> `analysis_file_name`** (line 200)

Opens the parent NWB file, creates a stripped-down copy (removes acquisition
data, processing modules, etc.), writes to disk. Returns the new filename.
Does NOT register the file in the table -- that must be done separately with
`add()`.

```python
analysis_file_name = AnalysisNwbfile().create(nwb_file_name)
```

**`add(nwb_file_name, analysis_file_name)`** (line 426)

Registers the file in the AnalysisNwbfile table by inserting a row with
`nwb_file_name`, `analysis_file_name`, and the resolved absolute path.

```python
AnalysisNwbfile().add(nwb_file_name, analysis_file_name)
```

**`add_nwb_object(analysis_file_name, nwb_object, table_name=...)`** (line 623)

Opens the analysis file in append mode, adds the NWB object to scratch space
(`nwbf.add_scratch()`), writes, and returns the `object_id`. Handles
DataFrames (converts to DynamicTable) and numpy arrays (converts to
ScratchData) automatically.

```python
obj_id = AnalysisNwbfile().add_nwb_object(
    analysis_file_name, my_timeseries, "clock_source"
)
```

**`build(nwb_file_name)`** (line 444)

Context manager that handles the CREATE -> POPULATE -> REGISTER lifecycle.
Recommended approach for new code:

```python
with AnalysisNwbfile().build(nwb_file_name) as builder:
    builder.add_nwb_object(my_data, "results")
    file = builder.analysis_file_name
# File automatically registered on exit
```

For direct HDF5 I/O (e.g., rewriting timestamps in place):

```python
with AnalysisNwbfile().build(nwb_file_name) as builder:
    with builder.open_for_write() as io:
        nwbf = io.read()
        # modify nwbf directly
        io.write(nwbf)
```

### IntervalList

Table for time intervals used in analysis
(`src/spyglass/common/common_interval.py`, line 23):

```python
@schema
class IntervalList(SpyglassIngestion, dj.Manual):
    definition = """
    # Time intervals used for analysis
    -> Session
    interval_list_name: varchar(170)
    ---
    valid_times: longblob  # numpy array with start/end times for each interval
    pipeline = "": varchar(64)
    """
```

Inserting a new interval:

```python
IntervalList.insert1(
    dict(
        nwb_file_name=nwb_file_name,
        interval_list_name="sync_valid_times_v1",
        valid_times=np.array([[start1, end1], [start2, end2]]),
        pipeline="neurokairos_v1",
    ),
    skip_duplicates=True,
)
```

The `Interval` class (line 319) provides operations on interval arrays
(intersection, union, consolidation, etc.). Use it via:

```python
from spyglass.common.common_interval import Interval

interval = Interval(valid_times_array, no_overlap=True)
interval.times  # consolidated numpy array
interval.as_dict  # dict ready for IntervalList.insert1()
```

To set the key fields for insertion:

```python
interval.set_key(
    nwb_file_name=nwb_file_name,
    interval_list_name="my_intervals",
    pipeline="neurokairos_v1",
)
IntervalList.insert1(interval.as_dict, skip_duplicates=True)
```

### SpyglassMixin

All spyglass tables inherit from `SpyglassMixin`
(`src/spyglass/utils/dj_mixin.py`, line 18). It composes:

- `CautiousDeleteMixin` -- permission-checked deletes
- `ExportMixin` -> `FetchMixin` -> `BaseMixin` -- NWB fetch, export tracking
- `HelperMixin` -- utility methods
- `PopulateMixin` -- populate with progress bars
- `RestrictByMixin` -- restrict_by convenience methods

Every table in spyglass-neurokairos should inherit from it:

```python
from spyglass.utils import SpyglassMixin

@schema
class IRIGDecoding(SpyglassMixin, dj.Computed):
    ...
```

### Merge Tables

The `Merge` class (`src/spyglass/utils/dj_merge_tables.py`, line 53) implements
a master+parts pattern with `merge_id: uuid` as primary key and `source:
varchar(32)` as secondary key.

```python
from spyglass.utils import Merge

schema_merge = dj.schema("neurokairos_merge")

@schema_merge
class NeuroKairosOutput(Merge, dj.Manual):
    definition = """
    merge_id: uuid
    ---
    source: varchar(32)
    """

    class IRIGDecoding(dj.Part):
        definition = """
        -> master
        ---
        -> neurokairos_v1_IRIGDecoding
        """

    class SynchronizedRecording(dj.Part):
        definition = """
        -> master
        ---
        -> neurokairos_v1_SynchronizedRecording
        """
```

Part tables inherit the master's `merge_id` as their sole primary key.

---

## Computed Table make() Pattern

Based on `SpikeSortingRecording.make()` (`spikesorting/v1/recording.py:199`):

```python
@schema
class IRIGDecoding(SpyglassMixin, dj.Computed):
    definition = """
    -> IRIGDecodingSelection
    ---
    -> AnalysisNwbfile
    clock_object_id: varchar(40)  # Object ID for ClockTable source in NWB
    """

    def make(self, key):
        # 1. Fetch upstream data
        sel_key = (IRIGDecodingSelection & key).fetch1()
        nwb_file_name = sel_key["nwb_file_name"]

        # 2. Decode IRIG -> ClockTable (using pynapple-irig)
        clock_table = self._decode_irig(sel_key)

        # 3. Create AnalysisNwbfile and write ClockTable
        analysis_file_name = AnalysisNwbfile().create(nwb_file_name)
        clock_obj_id = self._write_clock_to_nwb(analysis_file_name, clock_table)

        # 4. Register AnalysisNwbfile
        AnalysisNwbfile().add(nwb_file_name, analysis_file_name)

        # 5. Insert IntervalList entry (UTC-domain valid times from ClockTable)
        valid_times = self._derive_valid_times(clock_table)
        IntervalList.insert1(
            dict(
                nwb_file_name=nwb_file_name,
                interval_list_name=f"irig_decoded_{key['irig_decoding_id']}",
                valid_times=valid_times,
                pipeline="neurokairos_v1",
            ),
            skip_duplicates=True,
        )

        # 6. Insert into this table
        key["analysis_file_name"] = analysis_file_name
        key["clock_object_id"] = clock_obj_id
        self.insert1(key)
```

Or using the modern `build()` context manager:

```python
    def make(self, key):
        sel_key = (IRIGDecodingSelection & key).fetch1()
        nwb_file_name = sel_key["nwb_file_name"]
        clock_table = self._decode_irig(sel_key)

        with AnalysisNwbfile().build(nwb_file_name) as builder:
            clock_obj_id = self._write_clock_to_nwb(
                builder.analysis_file_name, clock_table
            )
            key["analysis_file_name"] = builder.analysis_file_name
        # File auto-registered on context exit

        key["clock_object_id"] = clock_obj_id
        self.insert1(key)
```

---

## DataJoint Table Definitions

### IRIGDecodingParams (Lookup)

```python
schema = dj.schema("neurokairos_v1")

@schema
class IRIGDecodingParams(SpyglassMixin, dj.Lookup):
    definition = """
    irig_params_name: varchar(64)
    ---
    irig_params: blob  # dict with decoder parameters
    """
    contents = [
        {"irig_params_name": "default", "irig_params": {
            "irig_format": "H",
            "sample_rate": 30000.0,
        }},
    ]
```

### IRIGDecodingSelection (Manual)

```python
@schema
class IRIGDecodingSelection(SpyglassMixin, dj.Manual):
    definition = """
    -> Session
    -> IRIGDecodingParams
    irig_channel_name: varchar(64)  # name of the IRIG channel in NWB
    ---
    irig_decoding_id: uuid  # unique ID for this decoding run
    """

    @classmethod
    def insert_selection(cls, key):
        if query := cls & key:
            return query.fetch(as_dict=True)
        key["irig_decoding_id"] = uuid.uuid4()
        cls.insert1(key, skip_duplicates=True)
        return key
```

### IRIGDecoding (Computed)

```python
@schema
class IRIGDecoding(SpyglassMixin, dj.Computed):
    definition = """
    -> IRIGDecodingSelection
    ---
    -> AnalysisNwbfile
    clock_object_id: varchar(40)  # NWB object ID for the serialized ClockTable
    """
```

### SynchronizedRecording (Computed)

```python
@schema
class SynchronizedRecording(SpyglassMixin, dj.Computed):
    definition = """
    -> IRIGDecoding
    source_object_name: varchar(128)  # name of the NWB object to synchronize
    ---
    -> AnalysisNwbfile
    sync_object_id: varchar(40)  # Object ID for the synchronized data in NWB
    """
```

### NeuroKairosOutput (Merge)

```python
schema_merge = dj.schema("neurokairos_merge")

@schema_merge
class NeuroKairosOutput(Merge, dj.Manual):
    definition = """
    merge_id: uuid
    ---
    source: varchar(32)
    """

    class IRIGDecoding(dj.Part):
        definition = """
        -> master
        ---
        -> neurokairos_v1.IRIGDecoding
        """

    class SynchronizedRecording(dj.Part):
        definition = """
        -> master
        ---
        -> neurokairos_v1.SynchronizedRecording
        """
```

---

## NWB Serialization Format for ClockTable

ClockTable has two arrays (source timestamps, reference timestamps) plus
metadata. Store in an AnalysisNwbfile as:

- **TimeSeries** named `"clock_source"`: source (device-time) timestamps as
  `data`, with `timestamps` set to the sample indices (or simply np.arange)
- **TimeSeries** named `"clock_reference"`: reference (UTC) timestamps as
  `data`, with matching `timestamps`
- **ScratchData** named `"clock_metadata"`: JSON string with ClockTable metadata
  (format version, device name, sample rate, etc.)

```python
import json
import numpy as np
from pynwb import TimeSeries
from pynwb.core import ScratchData

def clock_table_to_nwb(clock_table, analysis_file_name):
    """Serialize ClockTable to NWB objects in an AnalysisNwbfile."""
    anf = AnalysisNwbfile()

    source_ts = TimeSeries(
        name="clock_source",
        data=clock_table.source_timestamps,
        timestamps=np.arange(len(clock_table.source_timestamps), dtype=np.float64),
        unit="s",
        description="Source (device-time) timestamps from IRIG decoding",
    )
    source_id = anf.add_nwb_object(analysis_file_name, source_ts, "clock_source")

    ref_ts = TimeSeries(
        name="clock_reference",
        data=clock_table.reference_timestamps,
        timestamps=np.arange(len(clock_table.reference_timestamps), dtype=np.float64),
        unit="s",
        description="Reference (UTC) timestamps from IRIG decoding",
    )
    ref_id = anf.add_nwb_object(analysis_file_name, ref_ts, "clock_reference")

    metadata = ScratchData(
        name="clock_metadata",
        data=[json.dumps({
            "format_version": "1.0",
            "device_name": clock_table.device_name,
            "sample_rate": clock_table.sample_rate,
            "irig_format": clock_table.irig_format,
        })],
        description="ClockTable metadata as JSON",
    )
    meta_id = anf.add_nwb_object(analysis_file_name, metadata, "clock_metadata")

    return source_id  # use as the primary object_id reference


def clock_table_from_nwb(analysis_file_abs_path):
    """Deserialize ClockTable from an AnalysisNwbfile."""
    import pynwb

    with pynwb.NWBHDF5IO(analysis_file_abs_path, "r", load_namespaces=True) as io:
        nwbf = io.read()
        source = nwbf.scratch["clock_source"].data[:]
        reference = nwbf.scratch["clock_reference"].data[:]
        metadata = json.loads(nwbf.scratch["clock_metadata"].data[0])

    # Reconstruct ClockTable using pynapple-irig
    from pynapple_irig import ClockTable
    return ClockTable(
        source_timestamps=source,
        reference_timestamps=reference,
        **metadata,
    )
```

---

## Package Structure

```
spyglass-neurokairos/
├── src/spyglass_neurokairos/
│   ├── __init__.py
│   ├── v1/
│   │   ├── __init__.py
│   │   ├── irig_params.py          # IRIGDecodingParams (Lookup)
│   │   ├── irig_selection.py       # IRIGDecodingSelection (Manual)
│   │   ├── irig_decoding.py        # IRIGDecoding (Computed)
│   │   ├── sync_recording.py       # SynchronizedRecording (Computed)
│   │   └── irig_utils.py           # clock_table_to_nwb, clock_table_from_nwb, helpers
│   └── neurokairos_merge.py        # NeuroKairosOutput (Merge)
├── tests/
│   ├── conftest.py                 # Fixtures: synthetic ClockTable, mock NWB files
│   ├── test_nwb_roundtrip.py       # ClockTable <-> NWB serialization
│   ├── test_timestamp_rewrite.py   # Correctness of UTC timestamp mapping
│   └── test_pipeline.py            # Integration: params -> selection -> populate
├── pyproject.toml
└── CLAUDE.md                       # This file (or derived from it)
```

---

## Testing Strategy

### Synthetic ClockTable Fixtures

Generate deterministic test data -- no need for actual IRIG signal decoding
(that's tested in pynapple-irig):

```python
import numpy as np

def make_synthetic_clock_table(
    n_samples=60,
    source_start=0.0,
    source_step=1.0,
    drift_ppm=100.0,
    utc_epoch=1700000000.0,
):
    """Create a synthetic ClockTable with known drift for testing."""
    source = source_start + np.arange(n_samples) * source_step
    # Apply known drift: reference = source * (1 + drift_ppm/1e6) + offset
    scale = 1.0 + drift_ppm / 1e6
    reference = source * scale + utc_epoch
    return ClockTable(
        source_timestamps=source,
        reference_timestamps=reference,
        device_name="test_device",
        sample_rate=30000.0,
        irig_format="H",
    )
```

### Unit Tests

- **NWB roundtrip:** `clock_table_to_nwb()` -> `clock_table_from_nwb()` ->
  verify arrays match within float tolerance
- **Timestamp rewriting:** Known ClockTable + known input timestamps -> expected
  UTC timestamps (verify against manual computation)
- **Valid times derivation:** Verify correct intervals from various ClockTable
  shapes (gaps, concatenated files, edge cases)

### Integration Tests (require Docker MySQL)

- Full pipeline: insert params -> insert selection -> `IRIGDecoding.populate()`
  -> `SynchronizedRecording.populate()` -> verify corrected timestamps in output
  AnalysisNwbfile
- Merge table insertion and retrieval
- Verify a downstream pipeline (e.g., LFP filtering) produces correct results
  when fed synchronized vs. unsynchronized data

---

## Open Questions

- **Pipeline entry points:** Does spyglass's `SpikeSortingRecordingSelection`
  (and similar tables) accept an AnalysisNwbfile reference as input, or only raw
  NWB file names? If only raw NWB files, `SynchronizedRecording` may need to
  produce a file that can be referenced by `nwb_file_name` rather than
  `analysis_file_name`.

- **Cross-session alignment:** For setups where ephys and cameras are separate
  NWB files / sessions (ephys Session A + camera Session B), how are the
  synchronized recordings linked? May require a new "AlignedSession" or
  "RecordingGroup" concept.

- **ClockTable API:** The exact `ClockTable` constructor and method signatures
  depend on pynapple-irig's API. The serialization code above assumes
  `source_timestamps`, `reference_timestamps` attributes and a
  `synchronize(timestamps)` method. Verify against the actual pynapple-irig
  implementation.

- **Timestamp reference time:** When rewriting timestamps to UTC, should the
  NWB file's `timestamps_reference_time` be updated to a UTC epoch? Or should
  the timestamps be stored as seconds-since-some-UTC-epoch with the reference
  time updated accordingly? The answer depends on what downstream pipelines
  expect.
