# Nextflow Best Practices

My recommendations for Nextflow and nf-core best practices.

## Nextflow & nf-core: Beyond the Basics

This repository is intended to be a living guide to advanced Nextflow patterns.
The reader is expected to be familiar with programming in Nextflow and nf-core.
While [nf-core](https://nf-co.re/) provides a good foundation, it revolves around
community participation, and as a result, favours ease of composition rather than
efficiency. This repository demonstrates how to use the tools to make efficient
and user friendly pipelines.

## Key Concepts Demonstrated:

- **Multiple Entry Workflows:** Large workflows can become difficult to manage and
often users want to run individual stages or multiple stages similar to how one might
use Snakemake.
- **Module Patching:** How to use `nf-core modules patch` to optimize community modules
for performance without breaking update compatibility.
