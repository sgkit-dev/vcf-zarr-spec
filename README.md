This repo contains the [specification](vcf_zarr_spec.md) of the VCF Zarr format for storing VCF data in
[Zarr](https://zarr.readthedocs.io/) files. See the [preprint](https://www.biorxiv.org/content/10.1101/2024.06.11.598241v1)
for details on the rationale, and efficiency gains when processing large amounts of genetic variation data.

To convert a VCF file to VCF Zarr, see [vcf2zarr](https://sgkit-dev.github.io/bio2zarr/vcf2zarr/overview.html).

An implementation of this specification can be found in [sgkit](https://pystatgen.github.io/sgkit/).
