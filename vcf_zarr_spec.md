# VCF Zarr specification

***Version 0.3***

This document is a technical specification for VCF Zarr, a means of encoding VCF data in chunked-columnar form using the Zarr format.

This specification depends on definitions and terminology from [The Variant Call Format Specification, VCFv4.3 and BCFv2.2](https://samtools.github.io/hts-specs/VCFv4.3.pdf),
and [Zarr storage specification version 2](https://zarr.readthedocs.io/en/stable/spec/v2.html).

## Compatibility with VCF and BCF

Any VCF file that can be represented in BCF format can be represented in VCF Zarr.
Like BCF, VCF Zarr does not support the full set of VCF, since BCF is stricter than VCF around what is permitted in the header regarding field and contig information.

## Overall structure

A VCF Zarr store is composed of a top-level Zarr group that contains VCF header information stored as Zarr group attributes,
and VCF fields and sample information stored as Zarr arrays.

## VCF Zarr group attributes

The VCF Zarr store contains the following mandatory attributes:

| Key                | Value                                                                                |
|--------------------|--------------------------------------------------------------------------------------|
| `vcf_zarr_version` | `"0.3"`                                                                              |
| `vcf_header`       | The VCF header from `##fileformat` to `#CHROM` inclusive, stored as a single string. |

The following attributes are optional:

| Key      | Value                                                                                   |
|----------|-----------------------------------------------------------------------------------------|
| `source` | A string identifying the program (including a version number) writing the VCF Zarr data |

## VCF Zarr arrays

Each VCF field is stored in a separate Zarr array. This specification only mandates the path, shape, dimension names, and general dtype of each array. Other array metadata, including chunks, compression, layout order is not specified here.

### Zarr dtypes

This document uses a shorthand notation to refer to Zarr data types (dtypes). The following table shows the mapping to VCF types.

| Shorthand | Zarr dtypes                                              | VCF Type  |
|-----------|----------------------------------------------------------|-----------|
| `bool`    | `\|b1`                                                   | Flag      |
| `int`     | `<i1`, `<i2`, `<i4`, `<i8` or `>i1`, `>i2`, `>i4`, `>i8` | Integer   |
| `float`   | `<f4`, `<f8` or `>f4`, `>f8`                             | Float     |
| `char`    | `\|S1`                                                   | Character |
| `str`     | `\|O`                                                    | String    |

This specification does not mandate a byte order for numeric types: little-endian (e.g. `<i4`) or big-endian (`>i4`) are both permitted.

The `str` dtype is used to represent [variable-length strings](https://zarr.readthedocs.io/en/stable/tutorial.html#string-arrays). In this case a Zarr array filter with and `id` of `vlen-utf8` must be specified for the array.

### Missing and fill values

Missing values indicate the value is absent, and fill values are used to pad variable length fields. The following float values are based on the "signalling NaN" values used in BCF. Note that the BCF specification refers to fill values as "END_OF_VECTOR" values.

| Dtype     | Missing                                            | Fill                                               |
|-----------|----------------------------------------------------|----------------------------------------------------|
| `int  `   | -1                                                 | -2                                                 |
| `float  ` | NaN (0x7F800001 32-bit, 0x7FF0000000000001 64-bit) | NaN (0x7F800002 32-bit, 0x7FF0000000000002 64-bit) |
| `char`    | "."                                                | ""                                                 |
| `str`     | "."                                                | ""                                                 |

There is no need for missing or fill values for the `bool` dtype, since Type=Flag fields can only appear in INFO fields, and they always have Number=0.

In addition to encoding the missing and fill status in the values, some implementations may choose to store this information in accompanying Zarr arrays of the same shape. This can make masking out missing data easier for the user, since they don't have to know how missing and fill values are encoded for each type.

An array called `<name>` may have an accompanying array called `<name>_mask` with dtype `bool`, where true values indicate that the corresponding values are missing or fill values, and should be masked out. There may also be an accompanying array called `<name>_fill` with dtype `bool`, where true values indicate that the corresponding values are fill values.

### Array dimension names

Following [Xarray conventions](http://xarray.pydata.org/en/stable/internals/zarr-encoding-spec.html), each Zarr array has an attribute `_ARRAY_DIMENSIONS`, which is a list of strings naming the dimensions.

The reserved dimension names and their sizes are listed in the following table, along with the corresponding VCF Number value, if applicable.

| Dimension name        | Size                                                                             | VCF Number |
|-----------------------|----------------------------------------------------------------------------------|------------|
| `variants`            | The number of records in the VCF.                                                |            |
| `samples`             | The number of samples in the VCF.                                                |            |
| `ploidy`              | The maximum ploidy for any record in the VCF.                                    |            |
| `alleles`             | The maximum number of alleles for any record in the VCF.                         | R          |
| `alt_alleles`         | The maximum number of alternate non-reference alleles for any record in the VCF. | A          |
| `genotypes`           | The maximum number of genotypes for any record in the VCF.                       | G          |
| `contigs`             | The number of contigs in the VCF.                                                |            |
| `filters`             | The number of filters in the VCF.                                                |            |
| `parents`             | The number of unique parental categories used in the VCF header.                 |            |
| `region_index_values` | The number of chunks in the `variant_position` Zarr array.                       |            |
| `region_index_fields` | The number of fields in the index (6).                                           |            |

For fixed-size Number fields (e.g. Number=2) or unknown (Number=.), the dimension name can be any unique name that is not one of the reserved dimension names.

(*Implementation note: In general it is not possible to determine the size of some dimensions without a full pass through the VCF file being converted. Implementations may choose to fix the maximum size of some dimensions, although they should warn the user if this results in a lossy conversion.*)

### Fixed fields

The fixed VCF fields `CHROM`, `POS`, `ID`, `REF`, `ALT`, `QUAL`, and `FILTER` are stored as Zarr arrays according to the following table. Note that the `REF` and `ALT` fields are combined into a single Zarr array, with the `REF` first.

| VCF field(s) | Zarr array path    | Shape                 | Dimension names       | Dtype   |
|--------------|--------------------|-----------------------|-----------------------|---------|
| `CHROM`      | `variant_contig`   | `(variants)`          | `[variants]`          | `int`   |
| `POS`        | `variant_position` | `(variants)`          | `[variants]`          | `int`   |
| `ID`         | `variant_id`       | `(variants)`          | `[variants]`          | `str`   |
| `REF`, `ALT` | `variant_allele`   | `(variants, alleles)` | `[variants, alleles]` | `str`   |
| `QUAL`       | `variant_quality`  | `(variants)`          | `[variants]`          | `float` |
| `FILTER`     | `variant_filter`   | `(variants, filters)` | `[variants, filters]` | `bool`  |

Each value in the `variant_contig` array is a zero-based integer offset into the `contig_id` array.

The `variant_filter` array contains a true value at position `i` if the filter at position `i` in the `filter_id` array applies for a given variant. If no filters have been applied
for a variant, all values are false.

### INFO fields

Each INFO field is stored as a two-dimensional Zarr array at a path with name `variant_<field>`, of shape `(variants, <Number>)`, dimension names `[variants, <Number dimension name>]`. Dtypes, and missing and fill values are encoded as described above.

### FORMAT fields

Each FORMAT field is stored as a three-dimensional Zarr array at a path with name `call_<field>`, of shape `(variants, samples, <Number>)`, dimension names `[variants, samples, <Number dimension name>]`. Dtypes, and missing and fill values are encoded as described above.

A **Genotype (GT) field** is stored as a three-dimensional Zarr array at a path with name `call_genotype`, of shape `(variants, samples, ploidy)`, with an `int` dtype. Values encode the allele, with 0 for REF, 1 for the first alternate non-reference allele, and so on. A value of -1 indicates missing, and -2 indicates fill in mixed-ploidy datasets.

To indicate phasing, there is an optional accompanying Zarr array at a path with name `call_genotype_phased`, of shape `(variants, samples)`, with dtype `bool`. Values are true if a call is phased, false if unphased (or not present). If the array is not present then all calls are unphased.

### Contig information

Contig IDs are stored in a one-dimensional Zarr array at a path with name `contig_id`, of shape `(contigs)`, dimension names `[contigs]`, and with dtype `str`. The `contig_id` array plays the same role as the dictionary of contigs in BCF, providing a way of encoding a contig (in the `variant_contig` array) with a (zero-based) integer offset into the `contig_id` array.

Contig lengths are optional, and if present are stored in a one-dimensional Zarr array at a path with name `contig_length`, of shape `(contigs)`, dimension names `[contigs]`, and with dtype `int`.

### Filter information

Filters are stored in a one-dimensional Zarr array at a path with name `filter_id`, of shape `(filters)`, dimension names `[filters]`, and with dtype `str`. Filters must appear in the same order as specified in the header, except for `PASS`, which is always first.

Filter descriptions are stored in a one-dimensional Zarr array at a path with name `filter_description`, of shape `(filters)`, dimension names `[filters]`, and with dtype `str`.

### Sample information

Sample IDs are stored in a one-dimensional Zarr array at a path with name `sample_id`, of shape `(samples)`, dimension names `[samples]`, and with dtype `str`.

### Region index

To support efficient queries over variant regions, an optional two-dimensional Zarr array representing a region index may be stored at a path with name `region_index`, of shape `(region_index_values, region_index_fields)`, dimension names `[region_index_values, region_index_fields]`, and with the same `int` dtype as `variant_position`.

If `region_index` is present, the BCF `rlen` field is stored in a one-dimensional Zarr array at a path with name `variant_length`, of shape `(variants)`, dimension names `[variants]`, and with dtype `int`.

All Zarr arrays with a `variants` dimension must have the same chunk size in this dimension.

The region index must have a row for each distinct contig in each `variants` dimension chunk. The following properties of these chunk-contig pairs are stored as values in this column order:

* the `variants` dimension chunk index (zero-based)
* the contig ID (from `variant_contig`)
* the start position (from `variant_position`)
* the end position (from `variant_position`)
* the maximum end position (from `variant_position` combined with `variant_length`)
* the number of records

#### Example

To illustate how to build a region index, consider the following sample VCF Zarr dataset, which shows only the relevant `variant` fields, plus a chunk index for a chunk size of 3 in the `variants` dimension.

| Chunk index | `variant_contig` | `variant_position` | `variant_length` |
|-------------|------------------|--------------------|------------------|
| 0           | 0                | 111                | 1                |
| 0           | 0                | 112                | 1                |
| 0           | 1                | 14370              | 1                |
| 1           | 1                | 17330              | 1                |
| 1           | 1                | 1110696            | 1                |
| 1           | 1                | 1230237            | 1                |
| 2           | 1                | 1234567            | 1                |
| 2           | 1                | 1235237            | 1                |
| 2           | 2                | 10                 | 2                |

The corresponding `region_index` array is as follows:

| Chunk index | Contig ID | Start position | End position | Maximum end position | Number of  records |
|-------------|-----------|----------------|--------------|----------------------|--------------------|
| 0           | 0         | 111            | 112          | 112                  | 2                  |
| 0           | 1         | 14370          | 14370        | 14370                | 1                  |
| 1           | 1         | 17330          | 1230237      | 1230237              | 3                  |
| 2           | 1         | 1234567        | 1235237      | 1235237              | 2                  |
| 2           | 2         | 10             | 10           | 11                   | 1                  |

Using the region index to perform an overlap query is a two-step process. First, the set of query intervals is matched against the region index intervals (defined by the contig ID, start position, and either the end position or maximum end position, depending on the type of overlap). This gives a list of chunk indexes for the matching chunks, which can then be retrieved for the `variant_contig`, `variant_position`, and `variant_length` arrays before applying a second overlap query on each set of chunks to filter to the precise set of rows that overlap.

For the previous example, the query 1:1-20000 matches the second and third rows of the region index, corresponding to chunk indexes 0 and 1. Applying the same overlap query to each of these chunks returns the third and fourth rows of the original dataset (1:14370 from the first chunk and 1:17330 from the second).

## Changes

### Changes between VCF Zarr 0.2 and VCF Zarr 0.3

* Add an optional top-level attribute for `source`.
* Clarify type of `vcf_zarr_version` attribute.
* Add a new `parents` reserved dimension name.
* Add `filter_description` field.
* Add `region_index`.

### Changes between VCF Zarr 0.1 and VCF Zarr 0.2

* The `contigs` VCF Zarr group attribute was removed and replaced with a `contig_id` array and a `contigs` dimension name.
* Introduced a `contig_length` array for defining contig sequence lengths. 
* The `filters` VCF Zarr group attribute was removed and replaced with a `filter_id` array.
