# Splunk Study Lab

Hands-on Splunk administration practice using a Dockerized Splunk instance and a Kali VM as a log source. Built while prepping for a network security operations role focused on SIEM administration rather than analyst work.

## Environment Setup

### Requirements
- Docker installed on the host (or on Kali directly)
- ~4GB RAM free for the Splunk container
- `rsyslog` installed and running on Kali (many current Kali installs log only to the systemd journal by default; `/var/log/auth.log` will not exist as a real file without it)

Check and install if needed:
```bash
dpkg -l | grep rsyslog
sudo apt update
sudo apt install rsyslog -y
sudo systemctl enable rsyslog --now
```

Confirm the file exists before continuing — this matters for the forwarder mount step later:
```bash
ls -l /var/log/auth.log
```

### Run Splunk

```bash
docker run -d -p 8000:8000 -p 8088:8088 -p 8089:8089 -p 9997:9997 \
  -e SPLUNK_START_ARGS=--accept-license \
  -e SPLUNK_GENERAL_TERMS=--accept-sgt-current-at-splunk-com \
  -e SPLUNK_PASSWORD='SplunkLab2026x' \
  --name splunk \
  splunk/splunk:latest
```

| Port | Purpose |
|------|---------|
| 8000 | Web UI |
| 8088 | HTTP Event Collector |
| 8089 | REST/management API |
| 9997 | Forwarder receiving port |

Watch startup logs until initialization finishes:

```bash
docker logs -f splunk
```

Wait for the line indicating the startup playbook has completed before continuing. Log in at `http://localhost:8000` with `admin` and the password set above.

### Run the Universal Forwarder (Docker)

Splunk publishes an official Universal Forwarder image, so no separate package download or account is required.

Create a shared network so the forwarder can reach the Splunk container by name:

```bash
docker network create splunklab
docker network connect splunklab splunk
```

Start the forwarder, mounting Kali's real log file into the container read-only so it has something to watch:

```bash
docker run -d -p 9997:9997 \
  -v /var/log/auth.log:/var/log/auth.log:ro \
  -e SPLUNK_START_ARGS=--accept-license \
  -e SPLUNK_GENERAL_TERMS=--accept-sgt-current-at-splunk-com \
  -e SPLUNK_PASSWORD='UfPassword2026x' \
  --name uf \
  --network splunklab \
  splunk/universalforwarder:latest
```

Point the forwarder at the Splunk container by name:

```bash
docker exec -it uf /opt/splunkforwarder/bin/splunk add forward-server splunk:9997 -auth admin:UfPassword2026x
```

Enable receiving on the Splunk side under Settings > Forwarding and receiving.

Note: the Universal Forwarder has no web interface. That's expected — it is a shipping agent only, not a search or admin console.

## Exercises

### 1. Forward a real log source
Add a monitor input inside the forwarder container for the mounted `/var/log/auth.log` and confirm events land in Splunk's default index.

### 2. Build a dedicated index
Create a `kali_logs` index instead of using `main`. Update the forwarder's `inputs.conf` (inside the `uf` container, under `/opt/splunkforwarder/etc/system/local/`) to route into it with an appropriate sourcetype. Document the retention and sizing choices.

### 3. Generate real traffic to detect
Run a controlled brute-force attempt against a local test SSH service (own VM only) to produce authentic failed-login events rather than synthetic test data.

### 4. Correlation search and alert
Write an SPL search that counts failed logins by source IP over a time window, then convert it into a scheduled alert with a defined threshold and notification action.

### 5. Custom sourcetype via config files
Hand-write a `props.conf` / `transforms.conf` pair to parse a log format instead of relying on Splunk's automatic detection. Include field extractions.

### 6. Automate an admin task
Script something against the Splunk REST API or the Python SDK — for example, programmatic index creation or pulling alert results and generating a summary.

### 7. Dashboard panel
Build a single panel visualizing failed-login activity over time using `timechart`.

## Notes

Each exercise builds on the previous one, ending with a working pipeline: log ingestion, indexing, correlation logic, alerting, and automation, rather than isolated demos.
