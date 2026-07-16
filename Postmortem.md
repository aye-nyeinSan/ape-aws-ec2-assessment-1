# Incident Postmortem: [ AWS EC2 Assessment 1: SRE Drill for EC2 Performance Degradation ]

## Executive Summary

* **Owner:** Rubi
* **Status:** Resolved
* **Impact:** `/health` api is unreachable externally from first deploy and server is crashed due to disk exhaustion.
* **Severity:** P1

## What Happened?

- Nginx isn't forwarding the app on its default port and getting 404(Notfound) from public IP while returning 200 status on local host
* The application was found writing logs at ~68 MB/min toward a 10 GiB target on an 8.7 GB disk, which would have tripped the app's 95%-disk health failure and crashed the box within hours.

## Timeline (All times in UTC)

* **17:00** - fix ownership for `/opt/storage-breaker` where the app server will run => `sudo chown -R ubuntu:ubuntu /opt/storage-breaker`
* **17:39** - curl -i <http://13.251.126.134/health> → 404, Server: nginx.
* **17:41** - curl -i <http://127.0.0.1:3000/health> → 200, server: uvicorn. App healthy in localhost
* **17:56** - Added proxy_pass to nginx location /; still 404 in the file (/etc/nginx/sites-available/default)
* **18:00** - Removed leftover try_files;
* **18:11** - Public /health → 200 {"status":"healthy"}. Reachability resolved from public ip
* **19:11** - `/var/log/storage-breaker/application.log` observed at 917 MB → 1015 MB in ~60s
* **19:20** - Run uvicorn server under systemd by using own custom service file (/etc/systemd/system/myapp.service) so that app can be restart if server crash.
* **20:00** - logtrim.service + .timer deployed so that log can be rotate if it reaches certain amount we defined, first clean run
* **20:30** - verified disk size trimming;  app unaffected, health stayed 200
* **20:31+** - Timer firing every 60s; disk stable at 31–35%

## Root Cause Analysis

### Root Cause

* **Reachability:** Nginx server didn't pass incoming traffic to uvicorn server. It served static files via try_files $uri $uri/ =404 and had no proxy_pass.
* **Disk Growth:** `app.py` defines `TARGET_LOG_BYTES = 10 GiB` written over `DURATION_HOURS = 2.5` in 64 KiB records to `/var/log/storage-breaker/application.log`, with `HEALTH_FAILURE_THRESHOLD_PERCENT = 95.0`. The write volume exceeds total disk capacity by design.

### Resolution

* nginx location '/' rewritten to proxy_pass '<http://127.0.0.1:3000>' (uvicorn server) with try_files removed and forwarding headers added.
* uvicorn moved under systemd (myapp.service) with Restart=always and enabled for boot persistence.
* Log growth bounded by a systemd oneshot + timer (logtrim.service / logtrim.timer) running find ... -size +500M -exec truncate -s 0 {} + every 60s.
