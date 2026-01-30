# Deployment Guide (No Go Required)

This document explains how to deploy the Oracle DB Prometheus exporter to a target host **without Go installed**.

> **Note:** The exporter uses a pure Go Oracle driver (`go-ora`), so no Oracle Instant Client or external libraries are required on the target host.

## Prerequisites

| Component | Build Host (GitHub Actions) | Target Host |
|-----------|---------------------------|-------------|
| Go 1.22+  | ✅ Provided               | ❌ Not Required |
| Oracle Client | ❌ Not Required        | ❌ Not Required |
| GLIBC     | N/A (static build)        | Any version |

## Quick Start: GitHub Actions Build (Recommended)

The easiest way to build the exporter is using GitHub Actions:

1. Fork/clone this repository
2. Go to **Actions** → **Build Linux Release** → **Run workflow**
3. Specify the version (default: `0.6.0`)
4. Download the artifact `oracledb_exporter_linux_amd64.tar.gz`

## Local Build on Linux

If you prefer to build locally:

### 1. Install Go 1.22+

```bash
wget https://go.dev/dl/go1.22.5.linux-amd64.tar.gz
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go1.22.5.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/go/bin
```

### 2. Build Static Binary

```bash
git clone https://github.com/iamseth/oracledb_exporter.git
cd oracledb_exporter

VERSION=0.6.0
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build \
  -ldflags "-X main.Version=${VERSION} -s -w" \
  -o oracledb_exporter
```

### 3. Create Deployment Package

```bash
mkdir -p dist
cp oracledb_exporter default-metrics.toml dist/
cd dist && tar -czvf oracledb_exporter_${VERSION}_linux_amd64.tar.gz oracledb_exporter default-metrics.toml
```

## Cross-Compile from macOS

Since the exporter uses a pure Go driver with no CGO dependencies, cross-compilation is straightforward:

```bash
VERSION=0.6.0
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build \
  -ldflags "-X main.Version=${VERSION} -s -w" \
  -o oracledb_exporter
```

> **Note:** Unlike the IBM DB2 exporter, no Docker or emulation is required for cross-compilation.

## Target Host Deployment

### Extract and Run

```bash
tar -xzvf oracledb_exporter_*.tar.gz
./oracledb_exporter --database.dsn="oracle://user:pass@hostname:1521/service"
```

### Verify Static Binary

```bash
# Should show "not a dynamic executable" for static build
ldd ./oracledb_exporter
# Output: not a dynamic executable

# Or check file type
file ./oracledb_exporter
# Output: ELF 64-bit LSB executable, x86-64, statically linked, ...
```

### systemd Service

Create `/etc/systemd/system/oracledb_exporter.service`:

```ini
[Unit]
Description=Oracle DB Prometheus Exporter
After=network.target

[Service]
Type=simple
User=prometheus
Group=prometheus
Environment="DATA_SOURCE_NAME=oracle://user:password@hostname:1521/service"
ExecStart=/opt/oracledb_exporter/oracledb_exporter
WorkingDirectory=/opt/oracledb_exporter
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Install and enable:

```bash
# Create directory and copy files
sudo mkdir -p /opt/oracledb_exporter
sudo cp oracledb_exporter default-metrics.toml /opt/oracledb_exporter/

# Create service user (if not exists)
sudo useradd -rs /bin/false prometheus 2>/dev/null || true
sudo chown -R prometheus:prometheus /opt/oracledb_exporter

# Enable and start service
sudo systemctl daemon-reload
sudo systemctl enable --now oracledb_exporter

# Check status
sudo systemctl status oracledb_exporter
```

## Connection String Format

The `go-ora` driver supports connection strings in these formats:

### Basic Format
```
oracle://user:password@hostname:port/service_name
```

### With SID (instead of service name)
```
oracle://user:password@hostname:port?SID=orcl
```

### With additional options
```
oracle://user:password@hostname:port/service?TRACE FILE=trace.log
```

## Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `DATA_SOURCE_NAME` | Oracle connection string | - |
| `DATA_SOURCE_NAME_FILE` | File containing connection string | - |
| `DEFAULT_METRICS` | Path to metrics file | `default-metrics.toml` |
| `CUSTOM_METRICS` | Path to custom metrics file | - |
| `TELEMETRY_PATH` | Metrics endpoint path | `/metrics` |
| `QUERY_TIMEOUT` | Query timeout in seconds | `5` |

## Deployment Checklist

- [ ] Deployment package includes both binary AND `default-metrics.toml`
- [ ] Oracle database is accessible from the target host (port 1521)
- [ ] Database user has SELECT permissions on required system views
- [ ] Firewall allows access to exporter port (default: 9161)
- [ ] (Optional) Prometheus is configured to scrape the exporter

## Comparison with IBM DB2 Exporter

| Aspect | oracledb_exporter | ibm-db2-prometheus-exporter |
|--------|-------------------|----------------------------|
| Driver | Pure Go (`go-ora`) | CGO (`go_ibm_db`) |
| CGO Required | ❌ No | ✅ Yes |
| External Libraries | ❌ None | ✅ clidriver (~25MB) |
| Build Complexity | Simple | Complex |
| Deployment Package | Single binary (~15MB) | Binary + libraries (~40MB) |
| GLIBC Dependency | None (static) | GLIBC 2.17+ |
| Cross-compile from macOS | ✅ Native Go | ❌ Requires Docker |
