# dxt_examon — Parallel data extraction and backup for Examon HPC monitoring telemetry

`dxt_examon` is a Python tool for bulk **data extraction** (DXT) and daily backup of holistic
monitoring telemetry from the [Examon](https://github.com/EEESlab/examon) HPC monitoring
framework. It was built to archive operational time-series data from the **Marconi100**
supercomputer at [CINECA](https://www.cineca.it/), exporting the metrics from Examon's
KairosDB time-series backend into compressed, per-metric CSV files for offline analysis.

> Note: here **DXT** stands for *data extraction*. This project is unrelated to Darshan DXT
> (the I/O tracing extension); it does not trace or profile application I/O.

## What it does

- Connects to an Examon server through the `ExamonQL` query interface (Examon on top of a
  KairosDB time-series database).
- Reads a list of metric/plugin pairs from `M100_metrics.csv` (the Marconi100 metric catalog,
  ~26,900 metrics) and downloads each metric day by day.
- Filters queries to `cluster='marconi100'` for plugins that carry a cluster tag, and handles
  the special `job_table` case by querying the `job_info_marconi100` job accounting table.
- Saves each day of data as a gzip-compressed CSV under a `plugin/metric/` directory tree.
- Tracks progress per metric in a "follow file" (`<plugin>_<metric>_follow_dxt.csv`) that
  records the last extracted datetime, row count, and column count, so runs are incremental.
- Uses Python `multiprocessing` (a pool of worker processes) to extract many metrics in
  parallel.

## The problem it solves

HPC facilities like Marconi100 continuously produce large volumes of operational telemetry
(power, thermal, GPU, node, scheduler, and facility metrics) through monitoring plugins.
Examon stores this data in a KairosDB time-series database that is optimized for live
monitoring rather than long-term, offline, research-oriented access. `dxt_examon` bulk-exports
that telemetry into a plain, versioned, compressed CSV archive that can be analyzed with
standard data-science tooling (pandas), while automatically resuming and back-filling any
missing days.

## Data sources (Examon plugins)

The tool extracts metrics published by the following Marconi100 Examon plugins:

| Plugin | Description |
| --- | --- |
| `ganglia_pub` | Node and GPU metrics (e.g. NVIDIA GPU telemetry) |
| `ipmi_pub` | Out-of-band hardware sensors via IPMI |
| `slurm_pub` | Slurm scheduler / job state metrics |
| `nagios_pub` | Nagios health-check metrics |
| `schneider_pub` | Schneider electrical / power metrics |
| `vertiv_pub` | Vertiv cooling / facility metrics |
| `weather_pub` | Ambient weather metrics |
| `logics_pub` | Logics plugin metrics |
| `job_table` | Slurm job accounting (`job_info_marconi100`) |

## Repository layout

| File | Purpose |
| --- | --- |
| `dailyBackup_Parallel.py` | Daily incremental backup; wakes at a fixed hour and extracts the previous day's data for every metric in parallel. |
| `tryagain.py` | Gap-filler; compares the expected daily date range against each follow file and re-downloads only the missing days ("holes"). |
| `M100_metrics.csv` | Catalog of `metric,plugin` pairs to extract for Marconi100. |

## Requirements

- Python 3
- [`examon`](https://github.com/EEESlab/examon) client library (provides `examon.examon.Examon`
  and `ExamonQL`)
- `pandas`, `pytz`
- Access credentials to an Examon / KairosDB server

## Configuration

Both scripts read connection settings from a `config.ini` file (located one directory above
the scripts) with an `[examon_config]` section:

```ini
[examon_config]
KAIROSDB_SERVER = <examon-kairosdb-host>
KAIROSDB_PORT   = <port>
USER            = <username>
PWD             = <password>
```

Edit the data-output paths (`pth_data_files`, `loggin_path`) and the parallelism settings
(`processes`, `number_rows_chunks`) at the bottom of each script to match your environment.

## How to run

Incremental daily backup:

```bash
python dailyBackup_Parallel.py
```

Back-fill missing days across the full history:

```bash
python tryagain.py
```

Each run reads `M100_metrics.csv`, splits the metric list into chunks, and processes them
across a pool of worker processes, writing gzip CSVs and updating the follow files as it goes.

## License

Licensed under the [Apache License 2.0](LICENSE).
