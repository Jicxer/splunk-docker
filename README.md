# Splunk Study Lab

Hands-on Splunk admin practice using a Dockerized Splunk instance and a Kali VM as a log source. Built while prepping for a network security operations role that's focused on SIEM administration.

## Environment Setup

### Requirements
- Docker installed on the host (or on Kali directly)
- ~4GB RAM free for the Splunk container
- `rsyslog` installed and running on Kali. Current Kali installs log to the systemd journal by default, so `/var/log/auth.log` won't exist as a real file until rsyslog is added.

Check and install if needed:
```bash
dpkg -l | grep rsyslog
sudo apt update
sudo apt install rsyslog -y
sudo systemctl enable rsyslog --now
```

Confirm the file exists before continuing, since this matters for the forwarder mount step later:
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

Wait for the line showing the startup playbook has completed before moving on. Log in at `http://localhost:8000` with `admin` and the password set above.

### Run the Universal Forwarder (Docker)

Splunk publishes an official Universal Forwarder image, so there's no separate package download or account needed.

Create a shared network so the forwarder can reach the Splunk container by name:

```bash
docker network create splunklab
docker network connect splunklab splunk
```

Start the forwarder, mounting Kali's real log file into the container read only so it has something to watch. No port mapping needed here, since the forwarder only reaches out to Splunk over the shared network rather than needing to be reached from outside:

```bash
docker run -d \
  -v /var/log/auth.log:/var/log/auth.log:ro \
  -e SPLUNK_START_ARGS=--accept-license \
  -e SPLUNK_GENERAL_TERMS=--accept-sgt-current-at-splunk-com \
  -e SPLUNK_PASSWORD='UfPassword2026x' \
  -e SPLUNK_USER=root \
  --user root \
  --name uf \
  --network splunklab \
  splunk/universalforwarder:latest
```

`SPLUNK_USER=root` and `--user root` together stop the image from trying (and failing) to change file ownership on startup.

Point the forwarder at the Splunk container by name, and add a monitor input so it knows what to watch:

```bash
docker exec -it uf /opt/splunkforwarder/bin/splunk add forward-server splunk:9997 -auth admin:UfPassword2026x
docker exec -it uf /opt/splunkforwarder/bin/splunk add monitor /var/log/auth.log -auth admin:UfPassword2026x
```

Verify both took effect:
```bash
docker exec -it uf /opt/splunkforwarder/bin/splunk list forward-server -auth admin:UfPassword2026x
docker exec -it uf /opt/splunkforwarder/bin/splunk list monitor -auth admin:UfPassword2026x
```

Enable receiving on the Splunk side under Settings > Forwarding and receiving > Configure receiving, and confirm port 9997 is listed.

Note: the Universal Forwarder has no web interface. That's normal. It's a shipping agent only, not a search or admin console.

**Troubleshooting note:** if `/var/log/auth.log` doesn't exist on the host before running the `docker run -v` command above, Docker will quietly create an empty directory at that path instead of throwing an error. That breaks the mount with no obvious warning at container start. Confirm the file exists on the host first (see Requirements) before creating the forwarder container.

### Verify the pipeline end to end

Don't assume it's working. Confirm it:

```bash
docker ps                                    # both containers show Up
ssh localhost                                # generates a real auth event, fail is fine
tail -5 /var/log/auth.log                    # confirms it wrote on the host
docker exec -it uf tail -5 /var/log/auth.log # confirms the container sees the same file
```

Then in the Splunk web UI, Search & Reporting, time range set to All time:
```spl
index=* host=* | head 20
```
If the event you just generated shows up, the full chain is working: log write, forwarder, network, receiving port, indexing, search.

## Exercises

### 1. Forward a real log source (complete)
Added a monitor input inside the forwarder container for `/var/log/auth.log`, forwarded it to Splunk, and confirmed it's searchable in the web UI.

### 2. Build a dedicated index (complete)
Created a `kali_logs` index through Settings > Indexes, with `frozenTimePeriodInSecs = 604800` (7 days) and `maxTotalDataSizeMB = 500`, chosen deliberately for a lab rather than left at defaults. Updated the forwarder's monitor input to route `/var/log/auth.log` into `kali_logs` with `sourcetype = linux_secure` instead of the default `main` index.

Ran into two issues along the way, worth keeping a record of:
- Trying to `add monitor` a second time for the same file path failed with "one already exists." Splunk won't let you create a duplicate monitor stanza for a path that's already being watched. Fixed by running `remove monitor` first, then re-adding with the `-index` and `-sourcetype` flags.
- On a second attempt editing `inputs.conf` directly with `vi`, the new `index`/`sourcetype` lines ended up attached to the wrong stanza (`[splunktcp://9997]`, the network-receiving input) instead of a new `[monitor:///var/log/auth.log]` stanza. The existing monitor stanza got dropped from the file during editing rather than updated. Fixed by rewriting the file with both stanzas present as separate blocks, then restarting the forwarder.

### 3. Generate real traffic to detect
Run a controlled brute-force attempt against a local test SSH service (own VM only) to produce authentic failed-login events instead of synthetic test data.

### 4. Correlation search and alert
Write an SPL search that counts failed logins by source IP over a time window, then turn it into a scheduled alert with a defined threshold and notification action.

### 5. Custom sourcetype via config files
Hand-write a `props.conf` / `transforms.conf` pair to parse a log format instead of relying on Splunk's automatic detection. Include field extractions.

### 6. Automate an admin task
Script something against the Splunk REST API or the Python SDK, for example programmatic index creation or pulling alert results and generating a summary.

### 7. Dashboard panel
Build a single panel visualizing failed-login activity over time using `timechart`.

## What I Learned

**Docker and environment debugging**
- A bind mount (`-v host:container`) doesn't check that the host path exists first. If it doesn't, Docker quietly creates an empty directory there instead of erroring. That produced a mount that looked fine at container start but broke the moment the forwarder tried to read the "file," which was actually a directory.
- Orphaned Docker volumes from earlier failed container runs can carry broken state (like bad file ownership) across `docker rm` and recreation. `docker volume prune` clears volumes not attached to any running container, which is useful when a container keeps failing the same way even after the command itself is fixed.
- Modern Kali doesn't write `/var/log/auth.log` or other classic flat-text logs unless rsyslog is installed. Logging goes to the systemd journal (`journalctl`) by default instead. Any lab depending on flat log files needs rsyslog installed explicitly first.
- `SPLUNK_USER=root` combined with `--user root` was needed to stop the Universal Forwarder container from trying, and failing, to `chown` its own files at startup.

**Splunk indexing**
- An index is where data is physically stored and retained. A sourcetype is how that data is parsed and interpreted. They're independent settings, changing one doesn't affect the other.
- Retention (`frozenTimePeriodInSecs`) and size cap (`maxTotalDataSizeMB`) are two separate limits, and whichever gets hit first controls when old data ages out. Neither should be left at defaults without a reason. In production, retention is often driven by compliance requirements (NERC CIP log-retention rules, for example), not just convenience.
- A monitor input can't be duplicated for the same file path. Editing an existing input, either through `remove` + `add` or directly in `inputs.conf`, is required instead of adding a second one.
- `.conf` files are stanza-based, and each stanza is independent. Editing them by hand without care can silently merge or drop settings into the wrong stanza. Worth checking the file contents after any manual edit rather than assuming the change landed where intended.

## Notes

Each exercise builds on the one before it, ending with a working pipeline covering log ingestion, indexing, correlation logic, alerting, and automation, rather than a set of isolated demos.
