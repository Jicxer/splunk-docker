# Splunk Study Lab

Hands-on Splunk administration practice using a Dockerized Splunk instance and a Kali VM as a log source. Built while prepping for a network security operations role focused on SIEM administration rather than analyst work.

## Environment Setup

### Requirements
- Docker installed on the host (or on Kali directly)
- ~4GB RAM free for the Splunk container
- A Kali VM able to reach the Docker host over the network

### Run Splunk

```bash
docker run -d -p 8000:8000 -p 8088:8088 -p 8089:8089 -p 9997:9997 \
  -e SPLUNK_START_ARGS=--accept-license \
  -e SPLUNK_GENERAL_TERMS=--accept-sgt-current-at-splunk-com \
  -e SPLUNK_PASSWORD=pass \
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

Log in at `http://<docker-host-ip>:8000` with `admin` and the password set above.

### Install the Universal Forwarder on Kali

```bash
sudo dpkg -i splunkforwarder-*.deb
sudo /opt/splunkforwarder/bin/splunk start --accept-license
sudo /opt/splunkforwarder/bin/splunk add forward-server <splunk-container-ip>:9997
```

Enable receiving on the Splunk side under Settings > Forwarding and receiving.

## Exercises

### 1. Forward a real log source
Point the Universal Forwarder at `/var/log/auth.log` on Kali and confirm events land in Splunk's default index.

### 2. Build a dedicated index
Create a `kali_logs` index instead of using `main`. Update the forwarder's `inputs.conf` to route into it with an appropriate sourcetype. Document the retention and sizing choices.

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
