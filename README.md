# Onda Dataset Format

**Onda** is a lightweight format for storing and manipulating sets of multi-sensor, multi-channel, LPCM-encodable, annotated, time-series recordings.

The latest tagged version is [v0.5.0](https://github.com/beacon-biosignals/OndaFormat/tree/v0.5.0).

This document contains:

- Onda's [Terminology](#terminology)
- Onda's [Design Principles](#design-principles)
- Onda's [Specification](#specification)
- [Potential Alternative Technologies/Approaches](#potential-alternatives)

Implementations:

- Julia: https://github.com/beacon-biosignals/Onda.jl

## Terminology

This document uses the term...

- ...**"LPCM"** to refer to [linear pulse code modulation](https://en.wikipedia.org/wiki/Pulse-code_modulation), a form of signal encoding where multivariate waveforms are digitized as a series of samples uniformly spaced over time and quantized to a uniformly spaced grid.

- ...**"signal"** to refer to the digitized output of an individual sensor or process. A signal is comprised of metadata (e.g. LPCM encoding, channel information, sample data path/format information, etc.) and associated multi-channel sample data.

- ...**"recording"** to refer a collection of one or more signals recorded simultaneously over a well-defined time period.

- ...**"annotation"** to refer to a piece of (meta)data associated with a specific time span within a specific recording.

## Design Principles

[\[back to top\]](#onda-dataset-format)

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
- ...enable metadata, annotations etc. to be stored and processed separately from raw sample data artifacts without significant file system overhead.
- ...enable extensibility without sacrificing interpretability. New signal encodings, annotations, sample data file formats, etc. should all be user-definable by design.
- ...be simple enough that a decent programmer (with Google access) should be able to fully interpret (and write performant parsers for) an Onda dataset without ever reading Onda documentation.

### Onda is not...

- ...a sample data file format. Onda allows dataset authors to utilize whatever file format is most appropriate for a given signal's sample data, as long as the author provides a mechanism to deserialize sample data from that format to a standardized interleaved LPCM representation.
- ...a transactional database. The majority of an Onda dataset's mandated metadata is stored in tabular manifests containing recording information, signal descriptions, annotations etc. This simple structure is tailored towards Onda's target regimes (see above), and is not intended to serve as a persistent backend for external services/applications.
- ...an analytics platform. Onda seeks to provide a data model that is purposefully structured to enable various sorts of analysis, but the format itself does not mandate/describe any specific implementation of analysis utilities.

## Specification

[\[back to top\]](#onda-dataset-format)

### Versioning

This specification document is versioned in accordance with [semantic versioning](https://semver.org/); version numbers take the form `major.minor.patch` where...

- ...increments to `major` correspond to changes/additions that are likely to break existing Onda readers
- ...increments to `minor` correspond to changes/additions that are unlikely to break existing Onda readers
- ...increments to `patch` correspond to purely textual changes, e.g. clarifying a phrase or fixing a typo

Note that, in accordance with the semantic versioning specification, `0.minor.patch` releases may include breaking changes:

> Major version zero (0.y.z) is for initial development. Anything MAY change at any time. The public API SHOULD NOT be considered stable.

### Overview

The Onda format describes three different types of files:

- `*.annotations` files: [Arrow files](https://arrow.apache.org/docs/format/Columnar.html#ipc-file-format) that contain annotation (meta)data associated with a dataset.
- `*.signals` files: Arrow files that contain signal metadata (e.g. LPCM encoding, channel information, sample data path/format, etc.) required to find and read sample data files associated with a dataset.
- sample data files: Files of user-defined formats that store the sample data associated with signals.

Note that `*.annotations` files and `*.signals` files are largely orthogonal to one another - there's nothing inherent to the Onda format that prevents dataset producers/consumers from separately constructing/manipulating/transferring/analyzing these files. Furthermore, there's nothing that prevents dataset producers/consumers from working with multiple files of the same type referencing the same set of recordings (e.g. splitting all of a dataset's annotations across multiple `*.annotations` files).

The Arrow tables contained in `*.annotations` and `*.signals` must have [attached custom metadata](https://arrow.apache.org/docs/format/Columnar.html#custom-application-metadata) containing the key `"onda_format_version"` whose value specifies the version of the Onda format that an Onda reader must support in order to properly read the file. This string takes the form `"vM.m.p"` where `M` is a major version number, `m` is a minor version number, and `p` is a patch version number.

Each of the aforementioned file types are further specified in the following sections. These sections refer to the [logical types defined by the Arrow specification](https://github.com/apache/arrow/blob/master/format/Schema.fbs), but note that Onda reader/writer implementations may employ Arrow extension type aliases (for example, to provide more first-class UUID support).

### `*.annotations` Files

An `*.annotations` file contains an Arrow table with the following columns in the following order:

1. `recording_uuid` (128-bit `FixedSizeBinary`): The UUID identifying the recording with which the annotation is associated.
2. `uuid` (128-bit `FixedSizeBinary`): The UUID identifying the annotation.
3. `start_nanosecond` (`Duration` w/ `NANOSECOND` unit): The annotation's start offset in nanoseconds from the beginning of the recording. The minimum possible value is `0`.
4. `stop_nanosecond` (`Duration` w/ `NANOSECOND` unit): The annotation's stop offset in nanoseconds (exclusive) from the beginning of the recording. This value must be greater than the annotation's corresponding `start_nanosecond`.
5. `value` (author-specified type): The value associated with the annotation. This column may be any type as specified by the author of the file.

An example of an `*.annotations` table (whose `value` column happens to contain strings):

| `recording_uuid`                     | `uuid`                               | `start_nanosecond` | `stop_nanosecond` | `value`                       |
|--------------------------------------|--------------------------------------|--------------------|-------------------|-------------------------------|
| `0xb14d2c6d8d844e46824f5c5d857215b4` | `0x81b17ea902504371954e7b8b167236a6` | `5e9`              | `6e9`             | `"this is a value"`           |
| `0xb14d2c6d8d844e46824f5c5d857215b4` | `0xdaebbd1b0cab4b89acdde51f9c9a1d7c` | `3e9`              | `7e9`             | `"this is a different value"` |
| `0x625fa5eadfb24252b58d1eb350fa7df6` | `0x11aeeb4b743149808b53547642652f0e` | `1e9`              | `2e9`             | `"this is another value"`     |
| `0xa5c01f0e50fe4acba065fcf474e263f5` | `0xbc0be95e3da2495391daba233f035acc` | `2e9`              | `3e9`             | `"wow what a great value"`    |

### `*.signals` Files

A `*.signals` file contains an Arrow table with the following columns in the following order:

1. `recording_uuid` (128-bit `FixedSizeBinary`): The UUID identifying the recording with which the annotation is associated.
2. `file_path` (`List` of `Utf8`): A string identifying the location of the signal's associated sample data file. This string must either be a [valid URI](https://en.wikipedia.org/wiki/Uniform_Resource_Identifier) or a relative file path (specifically, relative to the location of the `*.signals` file itself).
3. `file_format` (`List` of `Utf8`): A string identifying the format of the signal's associated sample data file. All Onda readers/writers must support the following file formats (and may define and support additional values as desired):
    - `"lpcm"`: signals are stored in raw interleaved LPCM format (see format description below).
    - `"lpcm.zst"`: signals stored in raw interleaved LPCM format and compressed via [`zstd`](https://github.com/facebook/zstd)
4. `type` (`List` of `Utf8`): A string identifying the signal's type. Valid `type` values are alphanumeric, lowercase, `snake_case`, and contain no whitespace, punctuation, or leading/trailing underscores.
5. `channel_names` (`List` of `List` of `Utf8`): A list of strings where the `i`th element is the name of the signal's `i`th channel name. A valid channel name...
    - ...conforms to the same format as signal types (alphanumeric, lowercase, `snake_case`, and contain no whitespace, punctuation, or leading/trailing underscores).
    - ...conforms to an `a-b` format where `a` and `b` are valid channel names. Furthermore, to allow arbitrary cross-signal referencing, `a` and/or `b` may be channel names from other signals contained in the recording. If this is the case, such a name must be qualified in the format `signal_name.channel_name`. For example, an `eog` signal might have a channel named `left-eeg.m1` (the left eye electrode referenced to the mastoid electrode from a 10-20 EEG signal).
6. `start_nanosecond` (`Duration` w/ `NANOSECOND` unit): The signal's start offset in nanoseconds from the beginning of the recording. The minimum possible value is `0`.
7. `stop_nanosecond` (`Duration` w/ `NANOSECOND` unit): The signal's stop offset in nanoseconds (exclusive) from the beginning of the recording. This value must be greater than the signal's corresponding `start_nanosecond`.
8. `sample_unit` (`List` of `Utf8`): The name of the signal's canonical unit as a string. This string should conform to the same format as signal types (alphanumeric, lowercase, `snake_case`, and contain no whitespace, punctuation, or leading/trailing underscores), should be singular and not contain abbreviations (e.g. `"uV"` is bad, `"microvolt"` is good; `"l/m"` is bad, `"liter_per_minute"` is good).
9. `sample_resolution_in_unit` (`FloatingPoint` w/ `DOUBLE` precision): The signal's resolution in its canonical unit. This value, along with the signal's `sample_type` and `sample_offset_in_unit` fields, determines the signal's LPCM quantization scheme.
10. `sample_offset_in_unit` (`FloatingPoint` w/ `DOUBLE` precision): The signal's zero-offset in its canonical unit (thus allowing LPCM encodings that are centered around non-zero values).
11. `sample_type`: The primitive scalar type used to encode each sample in the signal. Valid values are:
    - `"int8"`: signed little-endian 1-byte integer
    - `"int16"`: signed little-endian 2-byte integer
    - `"int32"`: signed little-endian 4-byte integer
    - `"int64"`: signed little-endian 8-byte integer
    - `"uint8"`: unsigned little-endian 1-byte integer
    - `"uint16"`: unsigned little-endian 2-byte integer
    - `"uint32"`: unsigned little-endian 4-byte integer
    - `"uint64"`: unsigned little-endian 8-byte integer
    - `"float32"`: 32-bit floating point number
    - `"float64"`: 64-bit floating point number
12. `sample_rate` (`FloatingPoint` w/ `DOUBLE` precision): The signal's sample rate.

An example `*.signals` table:

| `recording_uuid`                     | `file_path`                                        | `file_format`                                            | `type`     | `channel_names`                         | `start_nanosecond` | `stop_nanosecond` | `sample_unit` | `sample_resolution_in_unit` | `sample_offset_in_unit` | `sample_type` | `sample_rate` |
|--------------------------------------|----------------------------------------------------|----------------------------------------------------------|------------|-----------------------------------------|--------------------|-------------------|---------------|-----------------------------|-------------------------|---------------|---------------|
| `0xb14d2c6d8d844e46824f5c5d857215b4` | `"./relative/path/to/samples.lpcm"`                | `"lpcm"`                                                 | `"eeg"`    | `["fp1", "f3", "f7", "fz", "f4", "f8"]` | `10e9`             | `10900e9`         | `"microvolt"` | `0.25`                      | `3.6`                   | `"int16"`     | `256`         |
| `0xb14d2c6d8d844e46824f5c5d857215b4` | `"s3://bucket/prefix/obj.lpcm.zst"`                | `"lpcm.zst"`                                             | `"ecg"`    | `["avl", "avr"]`                        | `0`                | `10800e9`         | `"microvolt"` | `0.5`                       | `1.0`                   | `"int16"`     | `128.3`       |
| `0x625fa5eadfb24252b58d1eb350fa7df6` | `"s3://other-bucket/prefix/obj_with_no_extension"` | `"flac"`                                                 | `"audio"`  | `["left", "right"]`                     | `100e9`            | `500e9`           | `"scalar"`    | `1.0`                       | `0.0`                   | `"float32"`   | `44100`       |
| `0xa5c01f0e50fe4acba065fcf474e263f5` | `"./another-relative/path/to/samples"`             | `"custom_price_format:{\"parseable_json_parameter\":3}"` | `"price"`  | `["price"]`                             | `0`                | `3600e9`          | `"dollar"`    | `0.01`                      | `0.0`                   | `"uint32"`    | `50.75`       |

### Sample Data Files

All sample data is encoded as specified by the corresponding signal's `sample_type`, `sample_resolution_in_unit`, and `sample_offset_in_unit` fields, serialized to raw LPCM format, and formatted as specified by the signal's `file_format` field.

While Onda explicitly supports arbitrary choice of file format for serialized sample data via the `file_format` field, Onda reader/writer implementations should support (de)serialization of sample data from any implementation-supported format into the following standardized interleaved LPCM representation:

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

An individual value in a multichannel sample can be "encoded" from its representation in canonical units to its integer representation via:

```
encoded_value = (decoded_value - sample_offset_in_unit) / sample_resolution_in_unit
```

where the division is followed/preceded by whatever quantization strategy is chosen by the user (e.g. rounding/truncation/dithering etc). Complementarily, an individual value in a multichannel sample can be "decoded" from its integer representation to its representation in canonical units via:

```
decoded_value = (encoded_value * sample_resolution_in_unit) + sample_offset_in_unit
```

## Potential Alternatives

[\[back to top\]](#onda-dataset-format)

In this section, we describe several alternative technologies/solutions considered during Onda's design.

- HDF5: HDF5 was a candidate for Onda's de facto underlying storage layer. While featureful, ubiquitous, and technically based on an open standard, HDF5 is [infamous for being a hefty dependency with a fairly complex reference implementation](https://cyrille.rossant.net/moving-away-hdf5/). While HDF5 solves many problems inherent to filesystem-based storage, most use cases for Onda involve storing large binary blobs in domain-specific formats that already exist quite naturally as files on a filesystem. Though it was decided that Onda should not explicitly depend on HDF5, nothing inherently technically precludes Onda dataset content from being stored in HDF5 in the same manner as any other similarly structured filesystem directory. For practical purposes, however, Onda readers/writers may not necessarily automatically be able to read such a dataset unless they explicitly feature HDF5 support (since HDF5 support isn't mandated by the format).

- Avro: Avro was primarily considered as an alternative to one-sample-data-file-per-signal approach ultimately adopted by Onda, [initially motivated by Uber's use of the format in a manner that was extremely similar to an early Onda prototype's use of NPY](https://eng.uber.com/hdfs-file-format-apache-spark/). Unfortunately, it seems that most of the well-maintained tooling for Avro is Spark-centric; in fact, the overarching Avro project [has struggled (until very recently) to keep a dedicated set of maintainers engaged with the project](https://whimsy.apache.org/board/minutes/Avro.html). Avro's most desirable features, from the perspective of Onda, was its compression and "random" row access. However, early tests indicated that neither of those features worked particularly well for signals of interest compared to domain-specific seekable compression formats like FLAC.

- EDF/MEF/etc.: Onda was originally motivated by bulk electrophysiological dataset manipulation, a domain in which there are many different recording file formats that are all generally designed to support a one-file-per-recording use case and are constrained to certain domain-specific assumptions (e.g. specific bit depth assumptions, annotations stored within signal artifacts, etc.). Technically, since Onda itself is agnostic to choice of file formats used for signal serialization, one could store Onda sample data in MEF/EDF.

- BIDS: BIDS is an alternative option for storing neuroscience datasets. As mentioned above, Onda's original motivation is electrophysiological dataset manipulation, so BIDS appeared to be a highly relevant candidate. Unfortunately, BIDS restricts EEG data to [very specific file formats](https://bids-specification.readthedocs.io/en/stable/04-modality-specific-files/03-electroencephalography.html#eeg-recording-data) and also does not account for the plurality of LPCM-encodable signals that Onda seeks to handle generically.

- MessagePack: Before v0.5.0, the Onda format used MessagePack to store all signal/annotation metadata. See [this issue](https://github.com/beacon-biosignals/OndaFormat/issues/25) for background on the switch to Arrow.

- JSON: In early Onda implementations, JSON was used to serialize signal/annotation metadata. While JSON has the advantage of being ubiquitous/simple/flexible/human-readable, the performance overhead of textual decoding/encoding was greater than desired for datasets with lots of annotations. In comparison, switching to MessagePack yielded a ~3x performance increase in (de)serialization for practical usage. The subsequent switch from MessagePack to Arrow in v0.5.0 of the Onda format yielded even greater (de)serialization improvements.

- BSON: BSON was considered as a potential serialization format for signal/annotation metadata. Before v0.5.0 of the Onda format, MessagePack was chosen over BSON due to the latter's relative complexity compared to the former. After v0.5.0 of the Onda format, BSON remains less preferable than Arrow from a tabular/columnar data storage perspective.
