# Incident Postmortem: AWS EC2 Assessment 1 — SRE Drill for EC2 Performance Degradation

---

## Table of Contents

- [Executive Summary](#executive-summary)
- [What Happened](#what-happened)
- [Root Cause Analysis](#root-cause-analysis)
  - [Reachability](#reachability)
  - [Disk Growth](#disk-growth)
- [Resolution](#resolution)
  - [1. Restore External Reachability (Nginx Reverse Proxy)](#1-restore-external-reachability-nginx-reverse-proxy)
  - [2. Deploy Under a Service Account and systemd](#2-deploy-under-a-service-account-and-systemd)
  - [3. Resolve Disk Exhaustion](#3-resolve-disk-exhaustion)
- [Preventive Measures](#preventive-measures)

---

## Executive Summary

- **Owner:** Rubi
- **Status:** Resolved
- **Impact:** The `/health` API was unreachable externally from the first deploy, and the server was at risk of crashing due to disk exhaustion.

---

## What Happened

- Nginx was not forwarding traffic to the app on its port. Requests to the public IP returned `404 (Not Found)`, while the same request to `localhost` returned `200`.
- The application was writing logs at ~68 MB/min toward a 10 GiB target on an 8.7 GB disk. This would have tripped the app's 95%-disk health failure and crashed the box within hours.

---

## Root Cause Analysis

### Reachability

Nginx did not pass incoming traffic to the uvicorn server. It served static files via `try_files $uri $uri/ =404` and had **no `proxy_pass`** directive, so external requests never reached the application.

### Disk Growth

`app.py` defines `TARGET_LOG_BYTES = 10 GiB` written over `DURATION_HOURS = 2.5` in 64 KiB records to `/var/log/storage-breaker/application.log`, with `HEALTH_FAILURE_THRESHOLD_PERCENT = 95.0`. The intended write volume exceeds total disk capacity **by design** — this is the "storage breaker" behavior the drill is testing.

---

## Resolution

### 1. Restore External Reachability (Nginx Reverse Proxy)

Confirmed the app was healthy locally but unreachable externally:

```bash
curl -i http://13.251.126.134/health   # 404, Server: nginx
curl -i http://127.0.0.1:3000/health   # 200, Server: uvicorn
```

Since the app was healthy on localhost, the problem was in Nginx. Inspected the service and its config:

```bash
systemctl status nginx   # check nginx service
cd /etc/nginx/
sudo nginx -T            # review the active nginx config
```

The active config (`/etc/nginx/sites-enabled/default`) showed `location /` serving static files with `try_files` and no route to the uvicorn service. Rewrote `location /` to reverse-proxy to uvicorn, removed `try_files`, and added forwarding headers:

```nginx
location / {
    proxy_pass http://127.0.0.1:3000;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

Reloaded Nginx:

```bash
sudo systemctl reload nginx
```

### 2. Deploy Under a Service Account and systemd

Created a dedicated, non-login service account so the deployment does not depend on the `ubuntu` user, and gave it ownership of the log folder:

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin assignment
sudo chown -R assignment:assignment /var/log/storage-breaker
```

Moved the app to `/opt` (the standard location for server apps), rebuilt the virtualenv, and set ownership:

```bash
sudo mv /home/ubuntu/ape-aws-ec2-assessment-1 /opt/assignment
cd /opt/assignment
sudo rm -rf .venv
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
sudo chown -R assignment:assignment /opt/assignment
```

Ran uvicorn under systemd with a custom unit so it restarts automatically if the server crashes:

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My FastAPI app
After=network.target

[Service]
User=assignment
Group=assignment
WorkingDirectory=/opt/assignment
ExecStart=/opt/assignment/.venv/bin/uvicorn main:app --host 127.0.0.1 --port 3000
Restart=always

[Install]
WantedBy=multi-user.target
```

Enabled and started the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now myapp
sudo systemctl status myapp
```

### 3. Resolve Disk Exhaustion

The `/health` endpoint began returning `503`, so I checked disk usage:

```bash
df -h /
```

The disk was nearly full — I could not even install the AWS CLI because there was no free space. Applied a layered fix:

**Bound log growth with a systemd timer.** A oneshot service (`logtrim.service`) plus timer (`logtrim.timer`) truncates oversized logs every 60 seconds:

```bash
find /var/log/storage-breaker -size +500M -exec truncate -s 0 {} +
```

**Grew the EBS volume** from 10 GB to 15 GB (modifying the existing attachment rather than attaching a second volume), then verified and extended the partition and filesystem:

```bash
lsblk                              # confirm the new size is visible
sudo growpart /dev/nvme0n1 1       # extend the partition into new space
# extend the filesystem, then verify
df -h /                            # usage back to ~75%
```

**Archived logs to S3 and reclaimed space:**

```bash
tar -czf - /var/log/storage-breaker | aws s3 cp - s3://log-bucket-assignment/ec2-logs/$(hostname)/final-$(date +%s).tar.gz
sudo truncate -s 0 /var/log/storage-breaker/application.log
df -h /
```

---

## Preventive Measures

- **Auto-restart:** uvicorn runs under systemd (`myapp.service`) with `Restart=always` and is enabled for boot persistence.
- **Bounded log growth:** `logtrim.service` + `logtrim.timer` truncate any log over 500 MB every 60 seconds.
- **Log archival:** logrotate ships logs to S3 (triggered around 80% disk usage).
- **Disk guard:** `disk_guard.sh` runs under cron every 5 minutes:

  ```bash
  */5 * * * * /opt/disk_guard.sh
  ```

- **Headroom:** EBS volume resized from 10 GB to 15 GB to buy recovery margin.
