# Script Exporter

Prometheus exporter that executes shell scripts and exposes their output as metrics.

## Features

- Multiple scripts with independent timeouts
- Per-script scraping via query parameter
- Basic authentication
- Hot reload (SIGHUP or HTTP)
- Self metrics (success, duration)
- No external dependencies


## Usage

```bash
# Basic
./script-exporter -config config.json

# With auth
./script-exporter -config config.json -auth-file auth.txt -listen-address :9103
```

## Options

| Flag | Default | Description |
|------|---------|-------------|
| `-config` | config.json | Config file path |
| `-listen-address` | :9103 | Listen address |
| `-auth-file` | - | Auth file (username:password) |
| `-metrics-path` | /metrics | Metrics endpoint |
| `-default-timeout` | 30s | Default script timeout |
| `-version` | - | Show version |
| `-help` | - | Show help |

## Endpoints

| Endpoint | Method | Auth | Description |
|----------|--------|------|-------------|
| `/metrics` | GET | Yes | All scripts |
| `/metrics?script=name` | GET | Yes | Single script |
| `/metrics?script=a,b` | GET | Yes | Multiple scripts |
| `/health` | GET | No | Health check |
| `/reload` | POST | Yes | Reload config |

## Config File (config.json)

```json
{
  "scripts": [
    {
      "name": "script_name",
      "path": "/path/to/script.sh",
      "timeout": "5s",
      "args": ["--flag", "value"],
      "env": ["VAR=value"]
    }
  ]
}
```
# OR 

```json
{
  "scripts": [
    {
      "name": "script1",
      "path": "s1.sh",
      "timeout": "5s"
    },
    {
      "name": "script2",
      "path": "s2.sh",
      "timeout": "10s"
    },
    {
      "name": "script3",
      "path": "s3.sh",
      "timeout": "30s"
    },
    {
      "name": "script4",
      "path": "s4.sh",
      "timeout": "15s"
    }
  ]
}

```

# OR

```json
{
  "scripts": [
    {
      "name": "alertmanager_metrics",
      "path": "/scripts/alertmanager_metrics.sh",
      "timeout": "5s",
      "env": [
        "ALERTMANAGER_URL=http://localhost:9093"
      ]
    },
    {
      "name": "disk_metrics",
      "path": "/scripts/disk_metrics.sh",
      "timeout": "10s"
    },
    {
      "name": "custom_script",
      "path": "/scripts/custom.sh",
      "timeout": "30s",
      "args": [
        "--verbose"
      ],
      "env": [
        "API_KEY=xxx"
      ]
    }
  ]
}
```



| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Unique script identifier |
| `path` | Yes | Script path (absolute or relative) |
| `timeout` | No | Timeout (default: 30s) |
| `args` | No | Command line arguments |
| `env` | No | Environment variables |

## Auth File (auth.txt)

```
admin:secretpassword
prometheus:scrape_pass
```

## Prometheus Configuration

```yaml
scrape_configs:
  # Script 1: every 5s
  - job_name: 'alertmanager-metrics'
    scrape_interval: 5s
    metrics_path: /metrics
    params:
      script: ['script1']
    static_configs:
      - targets: ['localhost:9103']
    basic_auth:
      username: 'prometheus'
      password: 'scrape_pass'

  # Script 2: every 30s
  - job_name: 'disk-metrics'
    scrape_interval: 30s
    metrics_path: /metrics
    params:
      script: ['script2']
    static_configs:
      - targets: ['localhost:9103']
    basic_auth:
      username: 'prometheus'
      password: 'scrape_pass'

  # Multiple scripts together
  - job_name: 'system-metrics'
    scrape_interval: 60s
    metrics_path: /metrics
    params:
      script: ['script3,script4']
    static_configs:
      - targets: ['localhost:9103']
```

## Self Metrics

```
script_exporter_success{script="name"} 1
script_exporter_duration_seconds{script="name"} 0.045
script_exporter_scrape_duration_seconds 0.123
```

## Hot Reload

```bash
# Via HTTP
curl -X POST -u admin:password http://localhost:9103/reload

# Via signal
kill -HUP $(pidof script-exporter)
```

## Script Output Format

Scripts must output valid Prometheus metrics:

```bash
#!/bin/bash
echo "# HELP my_metric Description"
echo "# TYPE my_metric gauge"
echo "my_metric{label=\"value\"} 123"
```

## Example Test

```bash
# Start exporter
./script-exporter -config config.json -auth-file auth.txt -listen-address :9103

# Test all scripts
curl -u admin:password http://localhost:9103/metrics

# Test specific script
curl -u admin:password 'http://localhost:9103/metrics?script=disk_metrics'

# Health check (no auth)
curl http://localhost:9103/health
```

