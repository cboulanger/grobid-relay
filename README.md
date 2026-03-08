# grobid-relay

A WebSocket reverse-tunnel relay that makes a [GROBID](https://github.com/kermitt2/grobid) trainer service running inside an HPC cluster (Slurm/Apptainer) accessible from the outside world.

HPC job nodes typically cannot accept inbound connections. This relay runs on any internet-accessible host and proxies HTTP requests — including SSE streams — through a persistent WebSocket connection that the HPC node dials out to.

```text
[Your laptop / CI]         [This relay — internet-accessible]         [HPC job node]

  HTTP client ──────────► relay_server.py (port 8080) ◄── WS tunnel ── trainer_service.py
  curl, browser, scripts       proxies requests inward                 (port 8072, local)
```

The two scripts on the HPC side (`trainer_service.py` and `tunnel_client.py`) live in the GROBID repository under `grobid-home/scripts/`. The relay server in this repository is the counterpart that runs on your internet-accessible host.

## Installation

### 1. Clone and set up the Python environment

```bash
git clone https://github.com/you/grobid-relay.git
cd grobid-relay
uv sync
```

### 2. Configure the environment file

```bash
cp .env.template .env
```

Edit `.env` and set a strong secret token:

```env
RELAY_TOKEN=your-strong-secret-here
```

### 3. Configure and install the systemd service

```bash
cp grobid-relay.service.template grobid-relay.service
```

Edit `grobid-relay.service` and replace `/path/to/grobid-relay` with the absolute path to this directory (two occurrences):

```ini
[Service]
ExecStart=/home/cloud/grobid-relay/.venv/bin/python /home/cloud/grobid-relay/relay_server.py --port 8080
EnvironmentFile=/home/cloud/grobid-relay/.env
```

Then install and start the service:

```bash
sudo ln grobid-relay.service /etc/systemd/system/grobid-relay.service
sudo systemctl daemon-reload
sudo systemctl enable --now grobid-relay
```

Check status and logs:

```bash
sudo systemctl status grobid-relay
sudo journalctl -u grobid-relay -f
```

## Running manually (without systemd)

```bash
RELAY_TOKEN=mysecret python relay_server.py --port 8080
# or with explicit flags:
python relay_server.py --host 0.0.0.0 --port 8080 --token mysecret
```

### CLI options

| Option | Default | Description |
| ------ | ------- | ----------- |
| `--host ADDR` | `0.0.0.0` | Bind address. |
| `--port PORT` | `8080` | Bind port. |
| `--token SECRET` | `$RELAY_TOKEN` | If set, tunnel connections without a matching token are rejected. |

### Endpoints

| Endpoint | Description |
| -------- | ----------- |
| `WS /tunnel` | Tunnel connection from the trainer service on the HPC node |
| `GET /tunnel/status` | Check whether a tunnel is currently connected |
| `ANY /{path}` | Proxied to the trainer service through the tunnel |

## Putting the relay behind nginx (recommended)

Add these two location blocks inside your `server` block, **before** any catch-all `location /`:

```nginx
# WebSocket tunnel — the HPC node dials in here
location /tunnel {
    proxy_pass http://127.0.0.1:8080;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_read_timeout 3600s;
    proxy_send_timeout 3600s;
}

# REST + SSE endpoints — clients access the trainer service through here
location /relay/ {
    proxy_pass http://127.0.0.1:8080/;   # trailing slash strips the prefix
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_buffering off;                  # required for SSE streaming
    proxy_cache off;
    proxy_read_timeout 3600s;
    proxy_send_timeout 3600s;
    client_max_body_size 500M;
}
```

Apply: `sudo nginx -t && sudo systemctl reload nginx`

The HPC node then connects to `wss://your-domain/tunnel` and clients reach the trainer at `https://your-domain/relay/`.

> **Why two location blocks?** The `/tunnel` endpoint needs HTTP/1.1 upgrade headers and a long read timeout (the connection stays open for the entire job). The `/relay/` prefix handles HTTP and SSE traffic; `proxy_buffering off` is essential so SSE chunks are not held in nginx's buffer.

## GROBID integration

Two scripts on the GROBID/HPC side connect to the relay:

| File | Runs on | Purpose |
| ---- | ------- | ------- |
| `grobid-home/scripts/trainer_service.py` | HPC container | Trainer service; starts the tunnel client automatically when `--relay` is set |
| `grobid-home/scripts/tunnel_client.py` | HPC container | Dials out to the relay; forwards requests to the local trainer service |

### Starting the trainer service with the tunnel

Pass `--relay` (and optionally `--relay-token`) when starting the service inside the HPC container:

```bash
python grobid-home/scripts/trainer_service.py \
    --relay wss://relay.your-domain.com/tunnel \
    --relay-token mysecret
```

`RELAY_TOKEN` can be used instead of `--relay-token`:

```bash
export RELAY_TOKEN=mysecret
python grobid-home/scripts/trainer_service.py --relay wss://relay.your-domain.com/tunnel
```

Once connected, all trainer-service endpoints are accessible through the relay URL, including `GET /jobs/{job_id}/stream` (SSE streams work end-to-end).

**`trainer_service.py` tunnel options:**

| Option | Default | Description |
| ------ | ------- | ----------- |
| `--relay URL` | `""` | Relay WebSocket URL. If omitted, no tunnel is started. |
| `--relay-token SECRET` | `$RELAY_TOKEN` | Shared secret for authentication. |

### Running as an Apptainer / Slurm job

Pull the image once on the HPC login node (internet access required):

```bash
apptainer pull grobid-trainer.sif docker://you/grobid-trainer:latest
```

Paths inside the container follow the standard GROBID layout under `/opt/grobid/`. Bind-mount your scratch directories so training data and trained models survive across job runs.

Save the following as `train.sh` and adjust the paths and resource parameters to match your HPC cluster:

```bash
#!/bin/bash
#SBATCH --job-name=grobid-train
#SBATCH --time=12:00:00
#SBATCH --mem=16G
#SBATCH --gres=gpu:1          # omit if no GPU is needed

apptainer run \
  --nv \
  --bind /scratch/$USER/dataset:/opt/grobid/grobid-trainer/resources/dataset \
  --bind /scratch/$USER/models:/opt/grobid/grobid-home/models \
  --env RELAY_URL=wss://relay.your-domain.com/tunnel \
  --env RELAY_TOKEN="$RELAY_TOKEN" \
  grobid-trainer.sif
```

Submit the job from the login node:

```bash
export RELAY_TOKEN=mysecret
sbatch train.sh
```

Monitor it:

```bash
squeue --me               # check job state (PENDING → RUNNING)
scontrol show job <JOBID> # detailed info including the assigned node
```

Once the job is running the trainer service starts inside the container and the tunnel client dials out to the relay automatically. You can verify connectivity with:

```bash
curl https://your-domain/relay/tunnel/status
```


The container needs no inbound ports. `RELAY_URL` and `RELAY_TOKEN` are read by the container entrypoint and forwarded to `trainer_service.py` automatically.

### Running the tunnel client standalone

`tunnel_client.py` can also be run independently — useful for testing connectivity or wrapping a service started by other means:

```bash
python grobid-home/scripts/tunnel_client.py \
    --relay wss://relay.your-domain.com/tunnel \
    --token mysecret \
    --service http://localhost:8072
```

The client reconnects automatically with exponential back-off (up to 60 s) if the relay is temporarily unreachable.

### Accessing the trainer via the relay

If running the relay **directly** (no nginx, raw port 8080):

```bash
curl http://relay.your-domain.com:8080/health
curl -X POST http://relay.your-domain.com:8080/upload/segmentation \
  -F "file=@my-data.zip" -F "flavor=article/dh-law-footnotes"
curl -X POST http://relay.your-domain.com:8080/train/header \
  -H "Content-Type: application/json" \
  -d '{"mode": 2, "epsilon": 0.0001}'
# Stream live output (SSE — works transparently through the tunnel)
curl -N http://relay.your-domain.com:8080/jobs/a3f1bc7e/stream
```

If running **behind nginx** with the `/relay/` prefix:

```bash
curl https://your-domain/relay/health
curl -X POST https://your-domain/relay/upload/segmentation \
  -F "file=@my-data.zip" -F "flavor=article/dh-law-footnotes"
curl -X POST https://your-domain/relay/train/header \
  -H "Content-Type: application/json" \
  -d '{"mode": 2, "epsilon": 0.0001}'
curl -N https://your-domain/relay/jobs/a3f1bc7e/stream
```

## Blocking repeated unauthorized attempts with fail2ban

The relay logs every rejected connection:

```text
[relay] Unauthorized tunnel attempt from 1.2.3.4
```

**Behind nginx:** a rejected WebSocket upgrade appears as a `403` in the nginx access log. If you already have a fail2ban jail watching nginx for `4xx` errors, those attempts are caught automatically.

If no such jail exists, add one. Create `/etc/fail2ban/filter.d/nginx-403.conf`:

```ini
[Definition]
failregex = ^<HOST> -.*"(GET|POST|PUT|DELETE|PATCH|HEAD|OPTIONS|CONNECT) .* HTTP/.*" 403
journalmatch = _SYSTEMD_UNIT=nginx.service
```

And `/etc/fail2ban/jail.d/nginx-403.conf`:

```ini
[nginx-403]
enabled  = true
filter   = nginx-403
backend  = systemd
maxretry = 10
findtime = 60
bantime  = 3600
```

**Without nginx (raw port):** the relay logs the real client IP. Create `/etc/fail2ban/filter.d/grobid-relay.conf`:

```ini
[Definition]
failregex = \[relay\] Unauthorized tunnel attempt from <HOST>
journalmatch = _SYSTEMD_UNIT=grobid-relay.service
```

And `/etc/fail2ban/jail.d/grobid-relay.conf`:

```ini
[grobid-relay]
enabled  = true
filter   = grobid-relay
backend  = systemd
maxretry = 5
findtime = 60
bantime  = 3600
```

Apply: `sudo systemctl reload fail2ban`

## Security notes

- Set a strong `RELAY_TOKEN`. Without it the tunnel endpoint is unauthenticated.
- Run behind TLS (`wss://` not `ws://`) — use nginx/Caddy or pass `--ssl-certfile` / `--ssl-keyfile` to uvicorn.
- Only one tunnel connection is accepted at a time. A second client with the correct token replaces the first.
- The relay does not restrict which paths are forwarded. Access control is left to the caller (firewall, VPN, etc.).

## Idle resource usage

When no tunnel is connected the server is essentially sleeping — the asyncio event loop sits blocked on I/O with no timers or polling. Expect ~0% CPU and 30–60 MB RSS. It is safe to leave enabled permanently.
