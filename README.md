# Onda Dataset Format

**Onda** is a portable format for storing and manipulating sets of multi-sensor, multi-channel, LPCM-encodable, annotated, time-series recordings.

The latest tagged version is [v0.1.0](https://github.com/beacon-biosignals/OndaFormat/tree/v0.1.0).

This document contains:

- Onda's [Design Principles](#design-principles)
- Onda's [Specification](#specification)

## Design Principles

[\[back to top\]](#onda-dataset-format)

### Onda uses the term...

- ...**"LPCM"** to refer to [linear pulse code modulation](https://en.wikipedia.org/wiki/Pulse-code_modulation), a form of signal encoding where multivariate waveforms are digitized as a series of samples uniformly spaced over time and quantized to a uniformly spaced grid.

- ...**"signal"** to mean the digitized output of an individual sensor or process. A signal is comprised of an LPCM encoding, per-channel information, and a series of multi-channel samples represented as a matrix.

- ...**"recording"** to mean a collection of one or more signals recorded simultaneously over a well-defined time period.

### Onda is useful...

- ...when each signal in each recording in your dataset can fit in memory, but not all signals in each recording can fit in memory simultaneously.
- ...when each recording in your dataset can fit in memory, but not all recordings in your dataset can fit in memory simultaneously.
- ...when your dataset's signals benefit from sensor-specific encodings/compression codecs.
- ...as an intermediate target format for wrangling unstructured signal data before bulk ingestion into a larger data store.
- ...as an intermediate target format for local experimentation after bulk retrieval from a larger data store.
- ...as a format for sharing datasets comprised of several gigabytes to several terabytes of signal data.
- ...as a format for sharing datasets comprised of hundreds to hundreds of thousands of recordings.

### Onda is not...

- ...an analytics platform
- ...a file format
- ...a database

### Onda's design must...

- ...depend only upon technologies with standardized, implementation-agnostic specifications that are well-used across multiple application domains.
- ...support recordings where each signal in the recording may have a unique channel layout, physical unit resolution, bit depth and sample rate.
- ...be well-suited for ingestion into/retrieval from...
    - ...popular distributed analytics tools (e.g. Spark, TensorFlow).
    - ...traditional databases (e.g. PostgresSQL, Cassandra).
    - ...object-based storage systems (e.g. S3, GCP Cloud Storage).
- ...enable metadata, annotations etc. to be stored and processed separately from raw signal artifacts without significant file system overhead.
- ...enable extensibility without sacrificing interpretability. New signal encodings, annotations, compression formats, etc. should all be user-definable by design.
- ...be simple enough that a decent programmer (with Google access) should be able to fully interpret (and write performant parsers for) an Onda dataset without ever reading Onda documentation.

## Specification

[\[back to top\]](#onda-dataset-format)

### Versioning

This specification document is versioned in accordance with [semantic versioning](https://semver.org/); version numbers take the form `major.minor.patch` where...

- ...increments to `major` correspond to changes/additions that are likely to break existing Onda readers
- ...increments to `minor` correspond to changes/additions that are unlikely to break existing Onda readers
- ...increments to `patch` correspond to purely textual changes, e.g. clarifying a phrase or fixing a typo

### Directory Structure

An Onda dataset named `dataset_name` is comprised entirely of a filesystem directory named `dataset_name.onda` and that directory's contents.

This directory may contain any user-authored content, but **must** contain the following files/subdirectories:

```
dataset_name.onda/
    recordings.msgpack.zst
    samples/
    ⋮
```

Each of the above files/subdirectories is described in detail later, but here's a quick overview:

- `recordings.msgpack.zst`: a [zstd](https://github.com/facebook/zstd)-compressed [MessagePack](https://msgpack.org/index.html) file which contains a map whose entries represent individual recordings in the dataset.

- `samples/`: a directory containing all sample data associated with the dataset, grouped into subdirectories for each recording.

- `⋮`: any additional content as provided by the dataset author.

### `recordings.msgpack.zst`

Decompressing `recordings.msgpack.zst` yields a MessagePack Array with two elements. The first element is a MessagePack Map that serves as a header, while the second element is a MessagePack Map whose entries are `<uuid>: <recording object>` pairs.

The header object takes the same structure as the following example:

```
{
    "onda_format_version": "v0.1.0",
    "ordered_keys": false
}
```

Below is a detailed description for each field of the header object:

- `onda_format_version`: A string specifying the version of the Onda format that a reader must support in order to properly read this dataset. This string takes the form `"vM.m.p"` where `M` is a major version number, `m` is a minor version number, and `p` is a patch version number.

- `ordered_keys`: A boolean value. If `true`, readers may assume that all keys (except signal names) within each recording object in `recordings.msgpack.zst` are serialized in the exact order as the example given below. This ordering guarantee can facilitate increased parsing efficiency for implementations which choose to exploit it.

Each `<uuid>: <recording object>` pair in the second MessagePack Map takes the same structure as the following example:

```
"41459161-42bb-4e13-912f-6881ae356677": {
    "duration_in_nanoseconds": 1850078125000,
    "signals": {
        "eeg": {
            "channel_names": ["fp1", "f3", "c3", "p3", "f7", "t3", "t5",
                              "o1", "fz", "cz", "pz", "fp2", "f4", "c4",
                              "p4", "f8", "t4", "t6", "o2"],
            "sample_unit": "microvolt",
            "sample_resolution_in_unit": 0.25,
            "sample_type": "int16",
            "sample_rate": 256,
            "file_extension": "zst",
            "file_format_settings": {"level": 1}
        }
        ⋮
    },
    "annotations": [
        {
            "key": "epileptiform",
            "value": "spike",
            "start_nanosecond": 393500000000,
            "stop_nanosecond": 394500000000
        },
        ⋮
    ],
    "custom": ...
}
```

Below is a detailed description for each field of a recording object:

- `duration_in_nanoseconds`: The total duration of the recording in nanoseconds. This duration may be up to 1 nanosecond greater than the "actual" duration of the recording. All signals belonging to this recording MUST be of this duration, rounding up if sample rate/count do not divide evenly. For example, a 3-sample long signal at sampled at 22,222 Hz would be considered to have a duration of 135002 nanoseconds.

- `signals`: A map of `<name>: <signal object>` pairs representing the signals contained in the recording. The keys of the map are the signals' names as strings; valid signal names are alphanumeric, lowercase, `snake_case`, and contain no whitespace, punctuation, or leading/trailing underscores. The values of the map are objects with the following fields:
    - `channel_names`: An array of strings where the `i`th element is the name of the signal's `i`th channel name. A valid channel name...
        - ...conforms to the same format as signal names (alphanumeric, lowercase, `snake_case`, and contain no whitespace, punctuation, or leading/trailing underscores).
        - ...conforms to an `a-b` format where `a` and `b` are valid channel names. Furthermore, to allow arbitrary cross-signal referencing, `a` and/or `b` may be channel names from other signals contained in the recording. If this is the case, such a name must be qualified in the format `signal_name.channel_name`. For example, an `eog` signal might have a channel named `l-eeg.m1` (the left eye electrode referenced to the mastoid electrode from a 10-20 EEG signal).
    - `sample_unit`: The name of the signal's canonical unit as a string. This string should conform to the same format as signal names (alphanumeric, lowercase, `snake_case`, and contain no whitespace, punctuation, or leading/trailing underscores), should be singular and not contain abbreviations (e.g. `"uV"` is bad, while `"microvolt" is good`; `"l/m"` is bad, `"liter_per_minute"` is good).
    - `sample_resolution_in_unit`: The signal's resolution in its canonical unit as a floating point value. This value, along with the signal's `sample_type` field, determines the signal's LPCM quantization scheme.
    - `sample_type`: The primitive scalar type used to encode each sample in the signal. Valid values are:
        - `"int8"`: signed little-endian 1-byte integer
        - `"int16"`: signed little-endian 2-byte integer
        - `"int32"`: signed little-endian 4-byte integer
        - `"int64"`: signed little-endian 8-byte integer
        - `"uint8"`: unsigned little-endian 1-byte integer
        - `"uint16"`: unsigned little-endian 2-byte integer
        - `"uint32"`: unsigned little-endian 4-byte integer
        - `"uint64"`: unsigned little-endian 8-byte integer
    - `sample_rate`: The signal's sample rate as an unsigned integer.
    - `file_extension`: The extension of the signal's corresponding file name in the `recordings` directory, indicating the (potentially compressed) format to which the given signal was serialized. All Onda readers/writers must support the following file extensions (and may define and support additional values as desired):
        - `"raw"`: no compression is used; signals are stored as raw channel-major byte dumps.
        - `"zst"`: signals are compressed via [`zstd`](https://github.com/facebook/zstd)
    - `file_format_settings`: Either `nil`, or an object where each key-value pair corresponds to a configuration setting for the format indicated by the signal's `file_extension`. If an Onda reader/writer defines a new `file_extension` value, it must also define the valid `file_format_settings` values corresponding to that `file_extension` value. For the standard `"raw"` and `"zst"` file extensions, Onda readers/writers must support the following `file_format_settings` values:
        - `"raw"`: `nil`
        - `"zst"`: `nil`, or `{"level": i}` where 1 <= `i` <= 19
- `annotations`: An array of annotation objects. Each annotation is a key-value pair associated with a given time window in the corresponding recording and has the following fields:
    - `key`: The annotation's key as a string.
    - `value`: The annotation's value as a string.
    - `start_nanosecond`: The annotation's start offset in nanoseconds from the beginning of the recording. The minimum possible value is `0`.
    - `stop_nanosecond`: The annotation's stop offset in nanoseconds (inclusive) from the beginning of the recording. This value must be greater than or equal to the annotation's corresponding `start_nanosecond`.

- `custom`: Either `nil`, or a MessagePack value as specified by the dataset author. This field can be used to store domain-specific metadata for each recording.

Except for the `custom` and `file_format_settings` fields, `nil` values are entirely disallowed in recording objects.

### `samples/`

The `samples` directory has the following structure:

```
samples/
    <uuid>/<signal_1_name>.<signal_1.file_extension>
           <signal_2_name>.<signal_2.file_extension>
           ⋮
    <uuid>/<signal_1_name>.<signal_1.file_extension>
           <signal_2_name>.<signal_2.file_extension>
           ⋮
    ⋮
```

Each subdirectory in `samples` contains all sample data associated with the recording whose `uuid` field matches the subdirectory's name. Similarly, each file in a `recordings` subdirectory stores the sample data of the signal whose name matches the file's name. This sample data is encoded as specified by the signal's `sample_type` and `sample_resolution_in_unit` fields, serialized to raw LPCM format, and formatted as specified by the signal's `file_extension` field.

While Onda explicitly supports arbitrary choice of file format for serialized sample data via the `file_extension` and `file_format_settings` fields, Onda reader/writer implementations should support loading sample data stored in any supported format into the following standardized representation:

Given an `n`-channel signal, the byte offset for the `i`th channel value in the `j`th multichannel sample is given by `((i - 1) + (j - 1) * n) * byte_width(signal.sample_type)`. This layout can be expressed in the following table (where `w = byte_width(signal.sample_type)`):

| Byte Offset                 | Value                                |
|-----------------------------|--------------------------------------|
| 0                           | 1st channel value for 1st sample     |
| w                           | 2nd channel value for 1st sample     |
| ...                         | ...                                  |
| (n - 1) * w                 | `n`th channel value for 1st sample   |
| (n + 0) * w                 | 1st channel value for 2nd sample     |
| (n + 1) * w                 | 2nd channel value for 2nd sample     |
| ...                         | ...                                  |
| (2*n - 1) * w               | `n`th channel value for 2nd sample   |
| (2*n + 0) * w               | 1st channel value for 3rd sample     |
| (2*n + 1) * w               | 2nd channel value for 3rd sample     |
| ...                         | ...                                  |
| (3*n - 1) * w               | `n`th channel value for 3rd sample   |
| ...                         | ...                                  |
| ((i - 1) + (j - 1) * n) * w | `i`th channel value for `j`th sample |
| ...                         | ...                                  |

An individual value in a multichannel sample can be "encoded" from its representation in canonical units to its integer representation via division by the signal's `sample_resolution_in_unit` (followed/preceded by whatever quantization strategy is chosen by the user, e.g. rounding/truncation/dithering etc). Complementarily, an individual value in a multichannel sample can be "decoded" from its integer representation to its representation in canonical units via multiplication by the signal's `sample_resolution_in_unit`.
