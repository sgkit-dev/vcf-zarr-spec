This repo contains the [specification](vcf_zarr_spec.md) of the VCF Zarr format for storing VCF data in
[Zarr](https://zarr.readthedocs.io/) files. VCF Zarr is designed to enable efficient, scalable storage
and analysis of large-scale genetic variation data.

For details on the design rationale, performance characteristics, and efficiency gains when processing
large cohorts, see the published paper in *GigaScience*:

> **Analysis-ready VCF at Biobank scale using Zarr**  
> GigaScience, 2025.  
> https://doi.org/10.1093/gigascience/giaf049

The original preprint is also available on bioRxiv:  
https://doi.org/10.1101/2024.06.11.598241

To convert a VCF file to VCF Zarr, see
[vcf2zarr](https://sgkit-dev.github.io/bio2zarr/vcf2zarr/overview.html).

An implementation of this specification can be found in
[sgkit](https://pystatgen.github.io/sgkit/).

VCF Zarr is currently a draft specification, and we welcome input from a wide range of use cases
and perspectives. If you identify a specific issue with the specification (for example, unclear or
underspecified details), please open an
[issue](https://github.com/sgkit-dev/vcf-zarr-spec/issues).
For more open-ended discussion, please start a new thread on our
[discussions board](https://github.com/sgkit-dev/vcf-zarr-spec/discussions).
