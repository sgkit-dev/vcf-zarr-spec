# VCF Zarr specification

***Version 0.1***

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
| `vcf_zarr_version` | `0.1`                                                                                |
| `vcf_header`       | The VCF header from `##fileformat` to `#CHROM` inclusive, stored as a single string. |
| `contigs`          | A list of strings of the contig IDs in the same order as specified in the header.    |
| `filters`          | A list of strings of the filters in the same order as specified in the header, except for `PASS`, which is always first. |

The `contigs` attribute plays the same role as the dictionary of contigs in BCF, providing a way of encoding a contig (in the `variant_contig` array) with an integer offset into the `contigs` list.

## VCF Zarr arrays

Each VCF field is stored in a separate Zarr array. This specification only mandates the path, shape, dimension names, and dtype of each array. Other array metadata, including chunks, compression, layout order is not specified here.

### Zarr dtypes

This document uses a shorthand notation to refer to Zarr data types (dtypes). The following table shows the mapping to VCF types.

| Shorthand | Zarr dtypes    | VCF Type  |
|-----------|----------------|-----------|
| `bool`    | `\|b1`         | Flag      |
| `int`     | `<i1`, `<i2`, `<i4`, `<i8` or `>i1`, `>i2`, `>i4`, `>i8` | Integer   |
| `float`   | `<f4`, `<f8` or `>f4`, `>f8` | Float     |
| `char`    | `\|S1`         | Character |
| `str`     | `\|O`          | String    |

This specification does not mandate a byte order for numeric types: little-endian (e.g. `<i4`) or big-endian (`>i4`) are both permitted.

The `str` dtype is used to represent [variable-length strings](https://zarr.readthedocs.io/en/stable/tutorial.html#string-arrays). In this case a Zarr array filter with and `id` of `vlen-utf8` must be specified for the array.

### Missing and fill values

Missing values indicate the value is absent, and fill values are used to pad variable length fields. The following float values are based on the "signalling NaN" values used in BCF. Note that the BCF specification refers to fill values as "END_OF_VECTOR" values.

| Dtype     | Missing    | Fill          |
|-----------|------------|---------------|
| `int  `   | -1         | -2            |
| `float  ` | NaN (0x7F800001 32-bit, 0x7FF8000000000001 64-bit) | NaN (0x7F800002 32-bit, 0x7FF0000000000002 64-bit)    |
| `char`    | "."        | ""            |
| `str`     | "."        | ""            |

There is no need for missing or fill values for the `bool` dtype, since Type=Flag fields can only appear in INFO fields, and they always have Number=0.

In addition to encoding the missing and fill status in the values, some implementations may choose to store this information in accompanying Zarr arrays of the same shape. This make masking out missing data easier for the user, since they don't have to know how missing and fill values are encoded for each type.

An array called `<name>` may have an accompanying array called `<name>_mask` with dtype `bool`, where true values indicate that the corresponding values are missing or fill values, and should be masked out. There may also be an accompanying array called `<name>_fill` with dtype `bool`, where true values indicate that the corresponding values are fill values.

### Array dimension names

Following [Xarray conventions](http://xarray.pydata.org/en/stable/internals/zarr-encoding-spec.html), each Zarr array has an attribute `_ARRAY_DIMENSIONS`, which is a list of strings naming the dimensions.

The reserved dimension names and their sizes are listed in the following table, along with the corresponding VCF Number value, if applicable.

| Dimension name | Size                              | VCF Number |
|----------------|-----------------------------------|------------|
| `variants`     | The number of records in the VCF. | |
| `samples`      | The number of samples in the VCF. | |
| `ploidy`       | The maximum ploidy for any record in the VCF. | |
| `alleles`      | The maximum number of alleles for any record in the VCF. | R |
| `alt_alleles`  | The maximum number of alternate non-reference alleles for any record in the VCF. | A |
| `genotypes`    | The maximum number of genotypes for any record in the VCF. | G |
| `filters`      | The number of filters in the VCF. | |

For fixed-size Number fields (e.g. Number=2) or unknown (Number=.), the dimension name can be any unique name that is not one of the reserved dimension names.

(*Implementation note: In general it is not possible to determine the size of some dimensions without a full pass through the VCF file being converted. Implementations may choose to fix the maximum size of some dimensions, although they should warn the user if this results in a lossy conversion.*)

### Fixed fields

The fixed VCF fields `CHROM`, `POS`, `ID`, `REF`, `ALT`, `QUAL`, and `FILTER` are stored as Zarr arrays according to the following table. Note that the `REF` and `ALT` fields are combined into a single Zarr array.

| VCF field(s) | Zarr array path    | Shape                 | Dimension names       | Dtype   |
|--------------|--------------------|-----------------------|-----------------------|---------|
| `CHROM`      | `variant_contig`   | `(variants)`          | `[variants]`          | `int`   |
| `POS`        | `variant_position` | `(variants)`          | `[variants]`          | `int`   |
| `ID`         | `variant_id`       | `(variants)`          | `[variants]`          | `str`   |
| `REF`, `ALT` | `variant_allele`   | `(variants, alleles)` | `[variants, alleles]` | `str`   |
| `QUAL`       | `variant_quality`  | `(variants)`          | `[variants]`          | `float` |
| `FILTER`     | `variant_filter`   | `(variants, filters)` | `[variants, filters]` | `bool`  |

Each value in the `variant_contig` array is an integer offset into the `contigs` attribute list.

The `variant_filter` array contains a true value at position `i` if the filter at position `i` in the `filters` attribute list applies for a given variant. If not filters have been applied
for a variant, all values are false.

### INFO fields

Each INFO field is stored as a two-dimensional Zarr array at a path with name `variant_<field>`, of shape `(variants, <Number>)`, dimension names `[variants, <Number dimension name>]`. Dtypes, and missing and fill values are encoded as described above.

### FORMAT fields

Each FORMAT field is stored as a three-dimensional Zarr array at a path with name `call_<field>`, of shape `(variants, samples, <Number>)`, dimension names `[variants, samples, <Number dimension name>]`. Dtypes, and missing and fill values are encoded as described above.

A **Genotype (GT) field** is stored as a three-dimensional Zarr array at a path with name `call_genotype`, of shape `(variants, samples, ploidy)`, with an `int` dtype. Values encode the allele, with 0 for REF, 1 for the first alternate non-reference allele, and so on. A value of -1 indicates missing, and -2 indicates fill in mixed-ploidy datasets.

To indicate phasing, there is an optional accompanying Zarr array at a path with name `call_genotype_phased`, of shape `(variants, samples)`, with dtype `bool`. Values are true if a call is phased, false if unphased (or not present). If the array is not present then all calls are unphased.

### Sample information

Sample IDs are stored in a one-dimensional Zarr array at a path with name `sample_id`, of shape `(samples)`, dimension names `[samples]`, and with dtype `str`.
