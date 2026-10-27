# Self-hosting Pirate Weather

This guide covers running the ingest pipeline and API from this repository on your own hardware. The API itself (routes, parameters, response format) is documented in the [main repository](https://github.com/Pirate-Weather/pirateweather).

## What works and what doesn't

| Feature | Local instance |
| --- | --- |
| Forecast requests (`/forecast/...`) | Yes |
| Time machine / historic requests | No — local historic ingest output is never wired to the reader |
| Alerts | NWS alerts (US) only. `API/WMO_Alerts_Local.py` (global alerts) has no compose service |
| API key validation | Not enforced — any value works in the URL. The key is only used as an S3 credential in `TESTING` mode |
| Model coverage | Whatever you ingest. Stores for models you don't run are simply absent, and the API falls back per the [source priority rules](forecast-source-selection/README.md) |

## Architecture

```text
Ofelia (scheduler)
  └─> one-shot ingest container per model
        └─> downloads model output, processes it, writes chunked Zarr
              └─> FastAPI response container reads the Zarr stores
```

- Ingest services run to completion and exit; Ofelia re-runs them on a schedule.
- The API container loads all Zarr stores **once at startup**. If stores appear (or are corrected) after startup, the API must be restarted to see them.
- Scheduling, model list, and service wiring live in the compose files. Do not copy schedules or model lists into other docs — treat the compose files as the source of truth.

## Requirements

| Component | Memory | Disk | Notes |
| --- | --- | --- | --- |
| Ingest (all models, sequential) | ≥32 GB free | ≥200 GB working | Peak during processing of the largest global models |
| Ingest (regional subset, e.g. UK) | less | less | Scales with the models you keep; the [UK compose](../pirate-compose_oph.local.uk.yml) runs 3 global models |
| API response container | minimal | ~50 GB | Serves the Zarr stores; local NVMe/SSD strongly recommended (production serves from NVMe for IOPS) |

- Use **ext4** for the storage volumes; large ingest runs are significantly faster and more reliable than on NTFS mounts.
- The published container images (`public.ecr.aws/j9v4j3c7/...`) are ARM-only. On x86_64, build both images locally (commands in the quickstart below). The Dockerfiles are in [`Docker/`](../Docker/).

## Quickstart (full local stack)

```bash
git clone https://github.com/Pirate-Weather/pirate-weather-code.git
cd pirate-weather-code

# Build both images (required on x86_64; published images are ARM-only)
docker build -f Docker/pirate-ingest-dockerfile -t pirateingest:latest .
docker build -f Docker/pirate-api-dockerfile -t pirate-zarr-api:latest .

# Storage locations (any fast ext4 filesystem)
cp .env.example .env   # set PW_WEATHER_ROOT and PW_WORK_ROOT
mkdir -p /mnt/pirate-weather/Weather /mnt/pirate-weather/Work

docker compose -f pirate-compose_oph.local.yml up -d
```

The first cycle downloads and processes the initial batch of models and takes on the order of an hour; later cycles are scheduled by Ofelia. GFS alone downloads several GB — a partially complete cycle can take substantially longer on slower links, and transient download shortfalls trigger Herbie's retry loop (see Troubleshooting).

### Storage layout note

Keep both storage roots on the same filesystem if you can, but be aware that ingest scripts move files between them with `os.rename`, which fails across distinct mount points (`Invalid cross-device link`) even on the same physical disk. A single root with Work nested inside Weather avoids the issue entirely:

```text
<PW_WEATHER_ROOT>/          ← PW_WEATHER_ROOT, mounted at /data
<PW_WEATHER_ROOT>/work/     ← PW_WORK_ROOT, mounted at /tmp
```

See Troubleshooting for the failure signature if you keep them separate.

### Bridging ingest output to the API (required)

In `save_type=Download` mode the ingest jobs write versioned directories:

```text
<PW_WEATHER_ROOT>/Prod/<MODEL>/<INGEST_VERSION_STR>/<MODEL>.zarr
```

while the API reader opens flat stores:

```text
<save_dir>/<MODEL>.zarr
```

`INGEST_VERSION_STR` is defined in `API/constants/shared_const.py`; check the file for the value. Link the stores into the Weather root after the first ingest cycle completes:

```bash
cd "$PW_WEATHER_ROOT"
ln -s Prod/GFS/v30/GFS.zarr       GFS.zarr
ln -s Prod/GEFS/v30/GEFS.zarr     GEFS.zarr
ln -s Prod/ECMWF/v30/ECMWF.zarr   ECMWF.zarr
# repeat for each model you ingest
```

Then **restart the API container** — stores are read once at startup, so an API that started before the links existed will serve nothing.

## Regional subsets

The default local compose ingests nine models, six of which (NBM, SubH, HRRR, HRRR-6H, RTMA-RU, NWS alerts) cover only North America. For other regions, trim the compose to the models that cover you — see [`pirate-compose_oph.local.uk.yml`](../pirate-compose_oph.local.uk.yml) for a worked UK example (GFS + GEFS + ECMWF only).

Trimming a model means removing, in `pirate-compose_oph.local.yml`:

1. the ingest service block,
2. its `ofelia.job-run.<name>` labels, and
3. every reference to it in `depends_on` lists — Ofelia depends on all ingest services, and each ingest service depends on the whole chain before it. Removing a service without cleaning these up prevents the stack from starting.

Which models cover which region is documented in [`docs/forecast-source-selection/README.md`](forecast-source-selection/README.md). Note that several ingest scripts (`DWD_Mosmix`, `GDPS`, `GEPS`, `HRDPS`, `REPS`, `SILAM`, `RAQDPS`, `URMA`, `AIGFS`, `AIGEFS`, `ECMWF_AIFS`) exist in `API/` but have **no compose service** — `API/DWD_Mosmix_Local_Ingest.py` is the natural addition for European users and can be wired by copying any existing service block.

One script is not optional: `API/WMO_Alerts_Local.py` (global alerts) must run for the API to answer requests at all — the alerts store is read unguarded, so its absence 500s every request (see Troubleshooting). The UK compose wires it first in the chain because it completes in about a minute.

The API image layout is already wired correctly in the compose files here: the repository root is mounted at `/app` and the module path is `API.responseLocal:app`. If you write your own service definition, mount the repo root over `/app` rather than only the `API/` directory.

## Running only the API (response development)

To work on the response code without ingesting anything, run the API container in `TESTING` mode, which reads Zarr zip stores directly from production S3 using a Pirate Weather API key:

```bash
docker run -e STAGE=TESTING -e PW_API=<APIKEY> -e save_type=S3 \
  -p 8083:8083 public.ecr.aws/j9v4j3c7/pirate-alpine-zarr:dev \
  responseLocal:app --host 0.0.0.0 --port 8083
```

This is slower than local mode and consumes API calls, but requires no local data. To modify the response code, mount your checkout over `/app` as shown in the README.

## STAGE modes

| `STAGE` | Data source | Use |
| --- | --- | --- |
| `PROD` / `DEV` | Local stores at `<save_dir>/<MODEL>.zarr` | Full local self-host |
| `TESTING` | Zarr zip stores from production S3 (`PW_API` key used as S3 credential) | Response development without local ingest |
| `TIMEMACHINE` | Local `GFS.zarr` + `URMA_Hist.zarr` + ERA5 | Not functional locally; historic ingest output is never connected to it |

## Environment variables

Ingest-side variables are read by the scripts in `API/` and set per service in the compose files. API-side variables are read by `API/responseLocal.py`.

### Ingest

| Variable | Purpose |
| --- | --- |
| `wgrib2_path` | Path to the wgrib2 binary inside the ingest container (do not change) |
| `forecast_process_dir` | Working directory for intermediate processing files (map to `PW_WORK_ROOT`) |
| `forecast_path` | Final output root: `<forecast_path>/<INGEST_VERSION_STR>/<MODEL>.zarr` |
| `historic_path` | Historic output root (not consumed by the API in local mode) |
| `save_type` | `Download` (local files) or `S3` (upload zips) |
| `zarr_store_workers`, `zarr_async_concurrency` | Zarr write concurrency |
| `herbie_download_retries`, `herbie_retry_sleep_seconds` | Herbie download retry behaviour |
| `AWS_KEY`, `AWS_SECRET` | S3 credentials, only needed for `save_type=S3` |

### API

| Variable | Purpose |
| --- | --- |
| `STAGE` | See table above |
| `save_type` | `Download` (local) or `S3` |
| `save_dir` | Directory containing the flat `<MODEL>.zarr` stores (default `/tmp`) |
| `use_etopo` | Load the ETOPO elevation store from `save_dir`. If enabled without the store present, PROD/DEV requests fail — set to `FALSE` unless you have it |
| `TIMING` | Enable detailed timing output |
| `SKIP_ERA5`, `ERA5_CACHE_DIR` | ERA5 initialization for time-machine stages |
| `s3_bucket` | Bucket name for `TESTING`/`TM_TESTING` zip stores |
| `PW_API` | Pirate Weather API key (S3 credential in `TESTING` mode; not used for auth) |

`force_now` accepts boolean values only — timestamp strings are ignored, so it cannot pin a specific time.

## Pointing existing clients at your instance

The API serves the Dark Sky-style routes clients already expect:

```text
http://<host>:8083/forecast/<apikey>/<lat>,<lon>
```

- Any string works as `<apikey>`; the local API does not validate it.
- The [Home Assistant Pirate Weather integration](https://github.com/Pirate-Weather/pirate-weather-ha) supports a custom endpoint — set it to `http://<host>:8083` and the integration works unchanged.
- Models you did not ingest simply don't appear: missing models yield empty `alerts`, and `flags.sourceTimes` only contains the models you run.

## Troubleshooting

- **`docker compose up` hangs or Ofelia exits immediately** — one ingest service in the `depends_on` chain failed. Check each ingest container's logs; the chain runs strictly sequentially, so the first failure gates everything.
- **API starts but returns empty or errors** — the stores weren't present (or weren't linked) when the container started. Fix the layout, then restart the container; stores are only loaded at startup.
- **`<MODEL> not found` log lines at startup** — normal if you intentionally didn't ingest that model; the API falls back to the sources you do have.
- **Requests fail with a missing-ETOPO error** — set `use_etopo=FALSE` (note the lowercase name; `useETOPO` is silently ignored) or provide the ETOPO store in `save_dir`.
- **Ingest services with no output for a while** — check `docker stats`: active downloading shows high container NET I/O; idle CPU with steady memory means a retry backoff window, not a hang.

## Requirements and constraints

Behaviors of the current code that constrain how a local instance must be set up, with workarounds where they exist.

### Alerts store is required

The API reads the WMO alerts array without a guard for missing stores, so without a `WMO_Alerts.zarr` store every `/forecast` request fails with a 500 (`'NoneType' object is not subscriptable` in `grid_indexing.py`). Run `API/WMO_Alerts_Local.py` and link its store like any model store — it completes in about a minute and provides global alerts. With zero model stores ingested, requests return a 400 ("No forecast model sources are available").

Workaround without alerts: `?exclude=alerts` skips the alerts read, but standard clients do not send it by default.

### Work and Weather roots must share a mount point

The historic-loop ingest scripts move the temp Zarr from the Work volume to the Weather volume with `os.rename`, which fails across distinct mount points (`OSError: [Errno 18] Invalid cross-device link`) even when both are on the same physical disk — Docker bind mounts are separate mount points regardless of the underlying filesystem. Nest the Work root inside the Weather root (single mount; see [Storage layout note](#storage-layout-note)) or replace the `os.rename` calls with `shutil.move` (the approach used in discussion #650).

### Herbie downloads retry whole batches

The GRIB download validator treats a batch as failed when it is short by even one file ("Expected N downloaded ... paths but got N-1"), and the retry re-downloads the entire batch rather than only the missing files. Attempts and backoff are controlled by `herbie_download_retries` and `herbie_retry_sleep_seconds`; a persistently incomplete upstream file can stall a model for the full retry budget. A single retry cycle takes on the order of tens of minutes for GFS.
