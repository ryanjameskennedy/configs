# configs

Nextflow configs for running [JASEN](https://github.com/genomic-medicine-sweden/jasen) on our local servers with the local executor.

| Config | Server | CPU | RAM | CPUs used | Memory used |
| --- | --- | --- | --- | --- | --- |
| `jasen/thehorse.config` | thehorse | Intel Core i7-14700F (28 threads) | 125 GB | 28 | 112 GB |
| `jasen/mp_bioinfo.config` | mp-bioinfo | Intel Core i5-13600KF (20 threads) | 188 GB | 20 | 169 GB |

Each config sets `process.executor = 'local'`, caps the total CPUs and memory Nextflow uses at once (`executor`), and shrinks any task that requests more than the server has (`process.resourceLimits`). Memory is set to ~90% of RAM to leave headroom for the OS and Nextflow.

## Prerequisites

- JASEN installed with its containers, references and databases (`make install` in the JASEN repo, see the [installation docs](https://jasen.readthedocs.io/en/latest/install.html))
- Nextflow v24 or later
- Apptainer

## Running JASEN

Add the server's config with `-c` alongside the usual species, platform and container profiles:

```bash
nextflow run /path/to/jasen/main.nf \
    -c /path/to/configs/jasen/thehorse.config \
    -profile staphylococcus_aureus,illumina,apptainer \
    -work-dir /path/to/work \
    --csv samplelist.csv \
    --outdir /path/to/results
```

On mp-bioinfo, swap in `-c /path/to/configs/jasen/mp_bioinfo.config`. Add `-resume` to rerun from cached tasks.

To confirm the config is applied before a run:

```bash
nextflow -c /path/to/configs/jasen/thehorse.config config /path/to/jasen -flat -profile staphylococcus_aureus,illumina,apptainer | grep -E "executor|resourceLimits"
```

## Kraken

Kraken is off by default. Enable it with `--use_kraken true --kraken_db /path/to/kraken_db`. The standard kraken2 database needs roughly its own size in RAM (~80 GB), which fits within both configs.

Leave `use_kraken_batch` off on thehorse: batch mode copies the database into `/dev/shm`, which needs 80–100 GB and thehorse has 63 GB. Check `df -h /dev/shm` on mp-bioinfo before enabling it there.
