# NebulaStream Live-Metrics Demo

A self-contained NebulaStream demo you can launch with a single command:

```bash
docker compose -f github.com/ls-1801/nebulastream-demo.git up
```

> Requires Docker Compose **v2.23.1+** (for inline `configs` content). If your
> Compose is older, or the remote-git form is unavailable, clone first:
>
> ```bash
> git clone https://github.com/ls-1801/nebulastream-demo.git
> cd nebulastream-demo
> docker compose up
> ```

## What it does

```
metrics-feeder  --(TCP, JSON lines)-->  worker  --(File sink, CSV)-->  ./output
   (python)                       (nebulastream/worker:latest)
                                          ^
                                          | gRPC: start / status / stop
                                    coordinator
                                 (nebulastream/nes-cli:latest)
```

Three services, all wired up in [`docker-compose.yml`](docker-compose.yml):

| Service          | Image                          | Role |
|------------------|--------------------------------|------|
| `metrics-feeder` | `python:3.12-slim`             | A tiny TCP server that emits one JSON object per second with its own CPU%, memory, and network-RX. |
| `worker`         | `nebulastream/worker:latest`   | The NebulaStream engine. Connects out to the feeder as a TCP source, runs a 5-second tumbling-window aggregation, and writes results to `/output`. |
| `coordinator`    | `nebulastream/nes-cli:latest` (+ `jq`) | Drives the query with `nes-cli`: **start** → trap a **stop** hook on the returned query id → **status**-monitor in a loop. Built from a small inline Dockerfile that adds `jq`. |

## The query

```sql
SELECT
  start, end,
  AVG(CPU_PCT)    AS avg_cpu_pct,
  MAX(MEM_BYTES)  AS max_mem_bytes,
  AVG(NET_RX_BPS) AS avg_net_rx_bps
FROM metrics
WINDOW TUMBLING(TS, SIZE 5 SEC)
INTO metrics_window
```

## What you'll see

- **`coordinator` logs**: the assigned query id, then a one-line `status` of the
  global query (e.g. `Running`) every 5 seconds.
- **`worker` logs**: engine startup and query execution.
- **`./output/metrics-windows.csv`**: one CSV row per 5-second window —
  `start,end,avg_cpu_pct,max_mem_bytes,avg_net_rx_bps`. The first row appears
  after ~6–10 seconds (a window only emits once a later timestamp arrives).

```bash
# follow just the coordinator's start/status output
docker compose logs -f coordinator

# watch the windowed results
tail -f output/metrics-windows.csv
```

## Stopping

```bash
docker compose down
```

On shutdown the coordinator receives `SIGTERM`, its trap runs
`nes-cli stop <query-id>`, and the query is retired cleanly before the
containers go away.

## How it fits together (notes)

- The coordinator's base image entrypoint is `nes-cli`; the compose file
  overrides it with a bash script (embedded as a `config`). `nes-cli start`
  prints the persisted query id on stdout, which the script captures and reuses
  for `stop`/`status`. The id mapping is kept in `XDG_STATE_HOME=/state` inside
  that container.
- NebulaStream upper-cases identifiers, so the feeder emits **upper-case** JSON
  keys (`TS`, `CPU_PCT`, `MEM_BYTES`, `NET_RX_BPS`) to match the schema.
- The TCP source is a **client**: the worker dials `metrics-feeder:5000`, so the
  python service is a TCP **server**. Records are newline-delimited (the TCP
  source's default tuple delimiter).
- Everything (topology, feeder script, coordinator script) is embedded in
  `docker-compose.yml` under `configs:`, which is why the single remote
  `-f github.com/...git` command needs no local files. Edit those blocks to
  change the schema, query, sink, or emit rate.
