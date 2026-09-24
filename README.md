# configs

Nextflow configs for running [JASEN](https://github.com/genomic-medicine-sweden/jasen) on our local servers with the local executor.

| Config | Server | CPU | RAM | CPUs used | Memory used |
| --- | --- | --- | --- | --- | --- |
| `jasen/thehorse.config` | thehorse | Intel Core i7-14700F (28 threads) | 125 GB | 28 | 112 GB |
| `jasen/thedonkey.config` | thedonkey | Intel Core i5-13600KF (20 threads) | 188 GB | 15 | 141 GB |

Each config sets `process.executor = 'local'`, caps the total CPUs and memory Nextflow uses at once (`executor`), and shrinks any task that requests more than the server has (`process.resourceLimits`).

`thehorse.config` is a bare resource cap at ~90% of RAM and nothing else, so it works with any Nextflow pipeline. `thedonkey.config` is a full JASEN site config: it also sets the work dir, database paths, publish target, Singularity and the execution reports. Its caps sit ~25% below the machine totals because thedonkey is shared with two colleagues and has no scheduler to arbitrate between them — the `$local` executor scope is the only thing preventing three runs from colliding.

## Prerequisites

- JASEN installed with its containers, references and databases (`make install` in the JASEN repo, see the [installation docs](https://jasen.readthedocs.io/en/latest/install.html))
- Nextflow v24 or later
- Singularity

`thedonkey.config` expects the databases and containers on the cache NVMe rather than inside the JASEN repo, so install them there:

```bash
make install ASSETS_DIR=/mnt/cache/dbs CONTAINERS_DIR=/mnt/cache/singularity
```

## Running JASEN

On thehorse, add the config with `-c` alongside the usual species, platform and container profiles:

```bash
nextflow run /path/to/jasen/main.nf \
    -c /path/to/configs/jasen/thehorse.config \
    -profile staphylococcus_aureus,illumina,singularity \
    -work-dir /path/to/work \
    --csv samplelist.csv \
    --outdir /path/to/results
```

On thedonkey the config carries the work dir, databases, publish target and Singularity itself, so only the species and platform profiles are needed. Run it inside tmux so a dropped VPN connection doesn't kill the run:

```bash
tmux new -s jasen 'nextflow run /path/to/jasen/main.nf -c /path/to/configs/jasen/thedonkey.config -profile staphylococcus_aureus,illumina -resume --csv samplelist.csv'
```

Set these in the shell before launching on thedonkey:

```bash
export NXF_SINGULARITY_CACHEDIR=/mnt/cache/singularity
export NXF_WORK=/media/shortterm/ryan/work
export NXF_OPTS='-Xms1g -Xmx4g'
```

`NXF_SINGULARITY_CACHEDIR` has to be exported rather than set in the config, because Nextflow reads it before the config is parsed. Raise `-Xmx` to 8g for batches of more than a few hundred samples.

To confirm the config is applied before a run:

```bash
nextflow -c /path/to/configs/jasen/thedonkey.config config /path/to/jasen -flat -profile staphylococcus_aureus,illumina | grep -E "executor|resourceLimits|_db|workDir|outdir"
```

## Kraken

Kraken is off by default. Enable it with `--use_kraken true`, and on thehorse also pass `--kraken_db /path/to/kraken_db`; thedonkey's config already points at `/mnt/cache/dbs/kraken2/standard_2025-06`.

The standard kraken2 database needs roughly its own size in RAM (~80 GB). JASEN requests 100 GB for the process, which fits under both caps, but on thedonkey it leaves only 41 GB for everything else — `maxForks = 1` on `kraken`, `kraken_batch` and `tbprofiler_mergedb` keeps two of them from ever running at once.

Leave `use_kraken_batch` off on thehorse: batch mode copies the database into `/dev/shm`, which needs 80–100 GB and thehorse has 63 GB. Check `df -h /dev/shm` on thedonkey before enabling it there.
