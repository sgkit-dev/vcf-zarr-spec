This repo contains the [specification](vcf_zarr_spec.md) of the VCF Zarr format for storing VCF data in
[Zarr](https://zarr.readthedocs.io/) files. See the [preprint](https://www.biorxiv.org/content/10.1101/2024.06.11.598241v1)
for details on the rationale, and efficiency gains when processing large amounts of genetic variation data.

To convert a VCF file to VCF Zarr, see [vcf2zarr](https://sgkit-dev.github.io/bio2zarr/vcf2zarr/overview.html).

An implementation of this specification can be found in [sgkit](https://pystatgen.github.io/sgkit/).

VCF Zarr is a draft specification, and we hope to gain input from a wide range of use-cases 
and perspectives. If there's a specific problem with the specification (for example, lack of 
clarity on some details) please open an [issue](https://github.com/sgkit-dev/vcf-zarr-spec/issues)
to discuss. For more open-ended discussiones, please start a new thread on our 
[discussions board](https://github.com/sgkit-dev/vcf-zarr-spec/discussions).
