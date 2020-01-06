# Onda Dataset Format

**Onda** is a lightweight format for storing and manipulating sets of multi-sensor, multi-channel, LPCM-encodable, annotated, time-series recordings.

The latest tagged version is [v0.2.3](https://github.com/beacon-biosignals/OndaFormat/tree/v0.2.3).

This document contains:

- Onda's [Design Principles](#design-principles)
- Onda's [Specification](#specification)
- [Potential Alternative Technologies/Approaches](#potential-alternatives)

Implementations:

- Julia: https://github.com/beacon-biosignals/Onda.jl

## Design Principles

[\[back to top\]](#onda-dataset-format)

### Onda uses the term...

- ...**"LPCM"** to refer to [linear pulse code modulation](https://en.wikipedia.org/wiki/Pulse-code_modulation), a form of signal encoding where multivariate waveforms are digitized as a series of samples uniformly spaced over time and quantized to a uniformly spaced grid.

- ...**"signal"** to mean the digitized output of an individual sensor or process. A signal is comprised of an LPCM encoding, per-channel information, and a series of multi-channel samples.

- ...**"recording"** to mean a collection of one or more signals recorded simultaneously over a well-defined time period.

### Onda is useful...

- ...when segments of a signal can fit in memory simultaneously, but an entire signal cannot.
- ...when each signal in each recording in your dataset can fit in memory, but not all signals in each recording can fit in memory simultaneously.
- ...when each recording in your dataset can fit in memory, but not all recordings in your dataset can fit in memory simultaneously.
- ...when your dataset's signals benefit from sensor-specific encodings/compression codecs.
- ...as an intermediate target format for wrangling unstructured signal data before bulk ingestion into a larger data store.
- ...as an intermediate target format for local experimentation after bulk retrieval from a larger data store.
- ...as a format for sharing datasets comprised of several gigabytes to several terabytes of signal data.
- ...as a format for sharing datasets comprised of hundreds to hundreds of thousands of recordings.

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

### Onda is not...

- ...a file format. Onda allows dataset authors to utilize whatever file format is most appropriate for a given signal, as long as the author provides a mechanism to deserialize sample data from that format to a standardized interleaved LPCM representation.
- ...a database. The majority of an Onda dataset's mandated metadata is stored in a single, monolithic JSON-like manifest containing recording information, signal descriptions, annotations etc. This simple structure is tailored towards Onda's target regimes (see above), and is not intended to serve as a persistent backend for external services/applications.
- ...an analytics platform. Onda seeks to provide a data model that is purposefully structured to enable various sorts of analysis, but the format itself does not mandate/describe any specific implementation of analysis utilities.

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
    "onda_format_version": "v0.2.0",
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
            "file_extension": "lpcm.zst",
            "file_options": nil
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
        - ...conforms to an `a-b` format where `a` and `b` are valid channel names. Furthermore, to allow arbitrary cross-signal referencing, `a` and/or `b` may be channel names from other signals contained in the recording. If this is the case, such a name must be qualified in the format `signal_name.channel_name`. For example, an `eog` signal might have a channel named `left-eeg.m1` (the left eye electrode referenced to the mastoid electrode from a 10-20 EEG signal).
    - `sample_unit`: The name of the signal's canonical unit as a string. This string should conform to the same format as signal names (alphanumeric, lowercase, `snake_case`, and contain no whitespace, punctuation, or leading/trailing underscores), should be singular and not contain abbreviations (e.g. `"uV"` is bad, while `"microvolt"` is good; `"l/m"` is bad, `"liter_per_minute"` is good).
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
        - `"lpcm"`: signals are stored in raw interleaved LPCM format (see format description below).
        - `"lpcm.zst"`: signals stored in raw interleaved LPCM format and compressed via [`zstd`](https://github.com/facebook/zstd)
    - `file_options`: Either `nil`, or an object where each key-value pair corresponds to a configuration setting for the file format indicated by the signal's `file_extension`. For Onda's standard `"lpcm"` and `"lpcm.zst"` file extensions, the only valid `file_options` value is simply `nil`. If an Onda reader/writer defines a new `file_extension` value, it must also define the valid `file_options` values corresponding to that `file_extension` value.
- `annotations`: A set of annotation objects stored as an array. As this array represents a set, it is not permitted to contain duplicate objects (as determined by value-equivalence). For practicality's sake, however, it is preferable for Onda readers to simply ignore duplicates rather than error upon encountering them. Each annotation is a key-value pair associated with a given time window in the corresponding recording and has the following fields:
    - `key`: The annotation's key as a string.
    - `value`: The annotation's value as a string.
    - `start_nanosecond`: The annotation's start offset in nanoseconds from the beginning of the recording. The minimum possible value is `0`.
    - `stop_nanosecond`: The annotation's stop offset in nanoseconds (inclusive) from the beginning of the recording. This value must be greater than or equal to the annotation's corresponding `start_nanosecond`.

- `custom`: Either `nil`, or a MessagePack value as specified by the dataset author. This field can be used to store domain-specific metadata for each recording.

Except for the `custom` and `file_options` fields, `nil` values are entirely disallowed in recording objects.

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

While Onda explicitly supports arbitrary choice of file format for serialized sample data via the `file_extension` and `file_options` fields, Onda reader/writer implementations should support (de)serialization of sample data from any implementation-supported format into the following standardized interleaved LPCM representation:

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

Values are stored in little-endian format.

An individual value in a multichannel sample can be "encoded" from its representation in canonical units to its integer representation via division by the signal's `sample_resolution_in_unit` (followed/preceded by whatever quantization strategy is chosen by the user, e.g. rounding/truncation/dithering etc). Complementarily, an individual value in a multichannel sample can be "decoded" from its integer representation to its representation in canonical units via multiplication by the signal's `sample_resolution_in_unit`.

## Potential Alternatives

[\[back to top\]](#onda-dataset-format)

In this section, we describe several alternative technologies/solutions considered during Onda's design.

- HDF5: HDF5 was a candidate for Onda's de facto underlying storage layer. While featureful, ubiquitous, and technically based on an open standard, HDF5 is [infamous for being a hefty dependency with a fairly complex reference implementation](https://cyrille.rossant.net/moving-away-hdf5/). While HDF5 solves many problems inherent to filesystem-based storage, most use cases for Onda involve storing large binary blobs in domain-specific formats that already exist quite naturally as files on a filesystem. Though it was decided that Onda should not explicitly depend on HDF5, nothing inherently technically precludes an Onda dataset from being stored in HDF5 in the same manner as any other similarly structured filesystem directory. For practical purposes, however, Onda readers/writers may not necessarily automatically be able to read such a dataset unless they explicitly feature HDF5 support (since HDF5 support isn't mandated by the format).

- Avro: Avro was primarily considered as an alternative to one-signal-per-file approach ultimately adopted by Onda, [initially motivated by Uber's use of the format in a manner that was extremely similar to an early Onda prototype's use of NPY](https://eng.uber.com/hdfs-file-format-apache-spark/). Unfortunately, it seems that most of the well-maintained tooling for Avro is Spark-centric; in fact, the overarching Avro project [has struggled (until very recently) to keep a dedicated set of maintainers engaged with the project](https://whimsy.apache.org/board/minutes/Avro.html). Avro's most desirable features, from the perspective of Onda, was its compression and "random" row access. However, early tests indicated that neither of those features worked particularly well for signals of interest compared to domain-specific seekable compression formats like FLAC.

- EDF/MEF/etc.: Onda was originally motivated by bulk electrophysiological dataset manipulation, a domain in which there are many different recording file formats that are all generally designed to support a one-file-per-recording use case and are constrained to certain domain-specific assumptions (e.g. specific bit depth assumptions, annotations stored within signal artifacts, etc.). Technically, since Onda itself is agnostic to choice of file formats used for signal serialization, one could store Onda sample data in MEF/EDF.

- BIDS: BIDS is an alternative option for storing neuroscience datasets. As mentioned above, Onda's original motivation is electrophysiological dataset manipulation, so BIDS appeared to be a highly relevant candidate. Unfortunately, BIDS restricts EEG data to [very specific file formats](https://bids-specification.readthedocs.io/en/stable/04-modality-specific-files/03-electroencephalography.html#eeg-recording-data) and also does not account for the plurality of LPCM-encodable signals that Onda seeks to handle generically.

- JSON: In early Onda implementations, JSON was used to serialize the `recordings` metadata file. JSON has the advantage of being ubiquitous, simple, flexible, and human-readable, but the performance overhead of textual decoding/encoding was greater than desired for datasets with lots of annotations. In comparison, switching to MessagePack yielded a ~3x performance increase in (de)serialization for practical usage.

- BSON: BSON was considered as a potential serialization format for the `recordings` metadata file. Ultimately, BSON's relative complexity and dissimilarity to JSON were both considered significant downsides compared to MessagePack.

- ProtoBuf/FlatBuffers/etc.: These formats were considered as potential serialization formats for the `recordings` metadata file. These tools were ultimately ruled out in favor of MessagePack, which is much easier to parse (as it doesn't require intermediate code generation and is quite similar to JSON) and was found to be nearly as performant for Onda's particular use case.

- Feather: Feather was briefly explored until it became clear that the format is planned to be deprecated in favor of ["Arrow files"](https://github.com/apache/arrow/blob/5e639059a0330cad256703516996fbed11c8fd83/site/faq.md#what-is-the-difference-between-apache-arrow-and-apache-parquet), which might be rebranded to ["Feather 2.0"](https://issues.apache.org/jira/browse/ARROW-5510) in the future. Future versions of Onda will likely reconsider Arrow as a potential serialization format for the `recordings` metadata file once Arrow has achieved a stable 1.0 release.

- Arrow/Parquet: Early versions of Onda stored `recordings` metadata in Arrow tables serialized to disk as Parquet. Internal usage of these early Onda versions revealed that the C++ implementations of Arrow/Parquet were extremely performant, but that [some features were incomplete](https://issues.apache.org/jira/browse/ARROW-1644) and other front-end bindings/implementations generally lagged behind the C++ implementation in terms of feature parity.
