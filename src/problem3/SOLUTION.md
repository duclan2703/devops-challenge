# Problem 3 – NGINX Load Balancer VM at 99% Disk Usage

## Scenario recap

- **OS:** Ubuntu 24.04
- **Disk:** 64 GB total
- **Role:** Single service — **NGINX** as reverse proxy / load balancer to upstreams
- **Symptom:** Monitoring reports **~99% storage usage** (sustained)

---

## 1. How I would troubleshoot (order of operations)

### 1.0 Service health and blast radius *before* touching disk

First question is not “what’s using disk?” but **“is the load balancer failing users right now, and how big is the blast radius?”** Establish that *before* destructive or disruptive recovery steps.

| Check | What to look at | Why |
|--------|-----------------|-----|
| **Still serving?** | Synthetic checks, LB health in front of this node, **upstream error rate** (if you have it) | Confirms whether the alert is **predictive** (disk high but NGINX OK) vs **active incident** (writes failing, 502s climbing). |
| **NGINX / upstream** | Current **5xx rate** for this node vs peers, **active connections**, **upstream health** in `nginx -T` / status if enabled | If upstreams are already unhealthy, **reload/restart** timing matters more (see §5). |
| **Alert context** | Is the alert **disk** only, or also **latency** / **error**? | **I/O saturation** can look like disk pressure in dashboards but have a different fix (see ###1.1). |

Only after you know **customer impact** and **whether you’re in a degraded upstream situation** do you optimize the order of **disk vs coordination** (e.g. drain first).

### 1.1 Rule out I/O saturation (not the same as “full disk”)

On cloud VMs, **throttled EBS / PD volumes** can show **high iowait**, **latency spikes**, and **queue depth** without the filesystem being 99% full. That can be misread as “disk problem.”

| Step | Command / action | Why |
|------|------------------|-----|
| I/O wait | `iostat -xz 1` (or `iotop`) for a few intervals | **%util**, **await**, **queue** — if disk is saturated but `df` is fine, you’re in **throughput/limit** territory, not log deletion. |
| **Cloud console** | Volume metrics: **burst balance**, **IOPS/throughput caps** | Explains **steady-state** slowness vs capacity. |

If **I/O is saturated** but space is **not** the constraint: fix **volume type/size**, **application I/O** (e.g. log sync), or **instance** limits — not truncate.

### 1.2 Confirm the alert and scope (filesystem)

- **Validate** the metric (time range, **which mount**).
- **SSH** into the host (or **serial console** if SSH fails due to full disk).
- If the shell warns **“No space left on device”**, prefer **read-only** inspection first so you don’t make things worse.

### 1.3 Reserved block space (ext4) — why “99%” is confusing

On **ext4**, a portion of the filesystem is often **reserved for root** (default **~5%** on non-root volumes).

| Check | Command | Why it matters |
|-------|---------|----------------|
| Reserved blocks | `sudo tune2fs -l /dev/<device> \| grep -i reserved` | You may see **~99%** in `df` for normal users while **root** processes still have **headroom** — or the opposite: **root-only** space left while **nginx** (non-root) gets **ENOSPC**. |
| Who hits ENOSPC? | **Same** `df` + **which user** runs nginx (`www-data` / `nginx`) | Explains “monitoring says 99% but nginx died first” or “I can still write as root.” |

**Triage implication:** Before assuming “everything is full,” check **reserved block** behavior and **which UID** is failing writes — **not** only the headline `Use%` column.

### 1.4 Identify *what* is full

| Step | Command / action | Why |
|------|------------------|-----|
| Filesystem layout | `df -h` | See **which mount** is at 99%. |
| Inode usage | `df -i` | **100% inodes** with free space left → millions of tiny files (often logs). |
| Largest consumers (quick) | `sudo du -h --max-depth=1 / 2>/dev/null \| sort -hr \| head` | Find heavy top-level dirs. |
| Drill into suspects | `sudo du -h --max-depth=1 /var /var/log /var/lib/nginx 2>/dev/null \| sort -hr` | NGINX and logs usually live under `/var`. |
| Find huge files | `sudo find /var/log /var/lib/nginx /tmp -type f -size +100M 2>/dev/null` | Large single files (logs, cores, dumps). |
| Open but deleted files | `sudo lsof +L1` or `sudo lsof \| grep deleted` | Process still writing to a **deleted** file; space not freed until process restarts. |

### 1.5 Quick wins that are often *not* log-related (long-lived VMs)

Run these **after** you know impact (§1.0) and **before** aggressive log deletion — they’re low-risk, reversible, and common on aging VMs.

| Area | What to check | Example actions |
|------|----------------|-----------------|
| **`/tmp`** | Stale files, runaway jobs | `sudo find /tmp -type f -mtime +7 -ls` — review, then delete **only** what policy allows (e.g. `find … -delete` after confirmation). |
| **Core dumps** | `core` files under `/var/crash`, app dirs, or `/` | `sudo find / -xdev -name 'core' -size +10M 2>/dev/null` — confirm not needed for investigation, then remove or move off-box. |
| **APT locks / partial state** | Failed upgrades leaving locks | `ls /var/lib/dpkg/lock* /var/lib/apt/lists/lock` — if stuck from a **crashed** apt, **do not** blindly delete; follow Ubuntu recovery docs; **stale** `*.deb` partials in `/var/cache/apt/archives/` after `apt clean` are fair game. |
| **Snap / journal** | (See §2 D/E) | — |

### 1.6 Correlate with NGINX

- **Config:** `nginx -T` (full effective config) or `grep -R access_log /etc/nginx`.
- **Log paths:** Default `access.log` / `error.log` under `/var/log/nginx/` unless custom.
- **Traffic:** If monitoring has **request rate** or **upstream errors**, spikes explain log growth.

### 1.7 Check automation that should prevent this

- **logrotate:** `cat /etc/logrotate.d/nginx`, `logrotate -d /etc/logrotate.d/nginx` (debug dry-run).
- **journald:** `journalctl --disk-usage` (if services log heavily to journal).
- **snap:** `du -sh /var/lib/snapd` (sometimes large).
- **APT:** `du -sh /var/cache/apt` (old packages).

---

## 2. Likely root causes (NGINX LB context)

Below: **cause → why it happens on an LB → impact → recovery**

---

### A. Unbounded or oversized **access / error logs**

**Cause:** High request volume, verbose `access_log` format, **debug** logging, or **upstream errors** generating many lines; **no rotation**, **rotation misconfigured**, or **postrotate** failing (e.g. nginx not reopened).

**Impact:**

- **Disk full** → new writes fail (logs, temp files, package updates).
- **Risk of service crash** or **failed reloads** if NGINX or system cannot write.
- **Observability loss** if logging breaks mid-incident.

**Recovery (immediate) — safe order:**

1. **Do not truncate first** in production without a **deliberate** decision.
   - **Compliance / audit:** Logs may be **legal or contractual** retention; **destroying** them can be a **reportable** event.
   - **Forensics / security:** Confirm **no active investigation** or post-incident hold that requires **immutable** retention.
   - **Preferred:** **Copy off-box, then truncate locally** — e.g. **~30 seconds** on cloud:  
     `aws s3 cp /var/log/nginx/access.log s3://bucket/prefix/$(hostname)-access-$(date +%Y%m%d%H%M%S).log`  
     (or equivalent GCP `gsutil`, Azure Blob, internal log archive). **Then** free space locally.
2. **If you must truncate** after copy (or policy allows **no** retention):
   - `sudo truncate -s 0 /var/log/nginx/access.log` is **destructive** — treat it like **data loss**; get **second pair of eyes** or **change approval** in regulated environments.
3. **Prefer non-destructive** where possible:
   - **Force logrotate:** `sudo logrotate -f /etc/logrotate.d/nginx` then **`sudo nginx -s reopen`** (or `systemctl reload nginx`) so NGINX releases old file handles — see **§5** before reloading on a live LB.
4. **If `lsof` shows deleted-but-open huge log:** space returns only after the writer **reopens** — **`nginx -s reopen`** / reload after §5.

**Recovery (durable):**

- Ensure **logrotate** runs daily with **postrotate** sending `USR1` to nginx (or `nginx -s reopen`) per Ubuntu/nginx packages.
- **Reduce volume:** tune `access_log` (disable on very hot locations, use `buffer`, `flush`, or shorter format), set `error_log` to `warn` not `debug`.
- **Centralized logging** (Filebeat, Fluent Bit, Fluentd, Vector, etc.) — **does not replace** local rotation: if the **shipper is down** and you relied on it to “move” logs off disk, you’ve **moved the failure mode** without a safety net. **Always** keep a **local floor**: **logrotate** (or size-based rotation) **still runs** and caps local disk regardless of shipper health; monitor **shipper lag** and **disk** independently.

---

### B. **Logrotate** not running or failing silently

**Cause:** Cron/timer issues, **permission** errors, **disk already full** before rotate runs (chicken-and-egg), broken config.

**Impact:** Same as (A); logs grow until disk is full.

**Recovery:**

1. Fix disk enough to run tools (prefer **copy+truncate** or **rotate** as in A, not blind truncate).
2. `sudo logrotate -f /etc/logrotate.d/nginx` and verify.
3. Check `systemctl list-timers` / `/etc/cron.daily/logrotate`.
4. Fix permissions (log files owned by `www-data` or `nginx` as per distro).

---

### C. **Client body / proxy temp files** on disk

**Cause:** `client_body_temp_path`, `proxy_temp_path` under `/var/lib/nginx/` (or default) filling up if **large uploads**, **slow upstreams**, or **many concurrent** buffered requests.

**Impact:** Full disk; **502/504**-style failures if NGINX cannot buffer.

**Recovery:**

1. `du -sh /var/lib/nginx/*` — find temp dirs.
2. **Stop** abusive traffic (WAF, rate limit) if attack; **increase** `client_max_body_size` only if appropriate; **reduce** buffering or **stream** where possible.
3. Clear **stale** temp files only when NGINX is not using them (or after maintenance window / restart).
4. Point temp paths to a **dedicated volume** with monitoring.

---

### D. **Journald** using excessive `/var/log/journal`

**Cause:** Other services or **systemd** logging heavily; **no retention** limits.

**Impact:** Fills `/var` or `/`; unrelated to NGINX binary but same VM.

**Recovery:**

- `sudo journalctl --vacuum-time=7d` or `--vacuum-size=500M`
- Set `/etc/systemd/journald.conf`: `SystemMaxUse=`, `MaxRetentionSec=`
- Restart `systemd-journald` if needed.

---

### E. **Snap** packages or **APT** cache

**Cause:** Old snapshots, large revisions; **APT** not cleaned after upgrades.

**Impact:** Gradual growth; can tip a 64 GB disk into critical range.

**Recovery:**

- `sudo snap list --all` / remove old revisions per Ubuntu docs.
- `sudo apt clean` / `sudo apt autoremove`
- Optional: `sudo journalctl --vacuum-time=…` as above.

---

### F. **Old kernel packages** and unused images

**Cause:** Frequent `apt upgrade` without `autoremove`.

**Impact:** `/boot` full (sometimes **separate small partition**) — can break upgrades; **not** always NGINX-related but common on Ubuntu.

**Recovery:**

- `df -h /boot`
- `sudo apt autoremove --purge`
- Remove old kernels if multiple are installed.

---

### G. **Something else** on the VM (scope creep)

**Cause:** “Only NGINX” in intent, but **cron jobs**, **scripts**, **manual dumps**, **Docker** (if ever installed), **backup** staging to disk.

**Impact:** Depends on the process; same full-disk symptoms.

**Recovery:**

- `sudo du -x / | sort -h` / `ncdu` if installed — find unexpected trees.
- Remove or move data per policy; **prevent** recurrence with **separate volume** or **off-box** storage.

---

## 3. Impacts summary (when disk stays at ~99%)

| Impact area | Effect |
|-------------|--------|
| **NGINX** | Failed reloads; inability to write logs or temp files; possible **502/504** to clients. |
| **OS** | SSH/session failures; **apt** cannot install packages; **cannot create small files** even if `df` shows a few MB free (fragmentation / reserved blocks). |
| **Reserved blocks (ext4)** | **root** may still write when **non-root** (nginx) sees **ENOSPC** — or the reverse; triage **which user** fails writes (§1.3). |
| **Security / compliance** | Audit gaps if logs stop; **destroying** logs without policy can **violate** retention or investigations. |
| **Recovery time** | Until space is freed and rotation/shipping is fixed, **risk of recurrence** within hours under load. |

---

## 4. Prevention (after incident) — alerts, ownership, and failure modes

### 4.1 What fires the alert (not just “>80%”)

- **Table-stakes:** Per-mount **disk %** and **inode %** (e.g. warn **80%**, page **90%**). Implement in **CloudWatch**, **Datadog**, **Prometheus + Alertmanager**, or your cloud’s monitoring — the important part is **which signal** and **who** gets paged.
- **Also:** **shipper** health (Filebeat/Fluent Bit/Vector **not running**, **lag**), **logrotate** last success (custom check or log age), **I/O wait** sustained **with** volume metrics.
- **Ownership:** On-call runbook names **who** escalates for **compliance** vs **infra** when logs must be **preserved** vs **emergency freed**.

### 4.2 On-call decision tree (short)

1. **Is traffic / 5xx degraded?** → If yes, page **on-call app** + consider **drain** (§5).
2. **Is it I/O saturation vs capacity?** → `iostat` + cloud volume metrics → **resize** / **volume type** vs **free space**.
3. **Is it capacity?** → `du` / `find` → **copy off-box** (S3/GCS) **then** rotate/truncate per policy.
4. **Shipper down?** → **Do not** assume “logs will leave”; **local logrotate** still must cap disk; fix shipper + **backlog** risk.

### 4.3 Centralized logging ≠ “disk solved”

- **Failure mode:** Shipper stopped → logs **never** leave → **local disk** still fills.
- **Mitigation:** **Independent** local **rotation/size caps** + **alerts** on **shipper** health + **disk**; optional **dual path** (ship + archive).

### 4.4 Engineering hygiene

- **NGINX:** minimal access logging on hot paths; **rate limits**; **temp paths** on monitored volume.
- **Runbooks:** **copy-first** truncate, **reload** procedure (§5), **forensic** hold checklist.
- **IaC / config management:** enforce `journald` limits, `logrotate`, **reserved block** awareness in dashboards (optional note field).

---

## 5. NGINX reload / restart on a **live** load balancer

**Reload** (`nginx -s reload` / `systemctl reload nginx`) is usually **graceful**: in-flight connections **drain**, workers **roll** to new config.

**But** in an incident:

| Risk | Detail |
|------|--------|
| **Brief window** | New config or upstream state can cause **momentary** mis-routing or **failed** upstream picks if state is wrong. |
| **Degraded upstreams** | If backends are **already unhealthy**, **reload** can **overlap** with **connection churn** — **increase** error rate briefly. |
| **Clustered LBs** | **keepalived**, **anycast**, or **multiple** NGINX nodes: you may want **drain** this node **from the outer LB** (or **lower weight**) **before** reload so **users** aren’t on the node you’re touching. |

**“Can I do this right now without asking anyone?”**

- **No** if: **change window** policy, **compliance** sign-off for log destruction, **upstream** is on fire, or **you’re the only** node in a **single-point** path.
- **Yes** with **care** if: **copy-first** done, **peer** nodes healthy, **reload** is **lower risk** than **OOM** / **ENOSPC** taking the node down.

**Default:** Prefer **`nginx -s reopen`** / **logrotate** after **copy** when the goal is **only** releasing log file handles — **narrower** blast radius than full **reload** if your package supports it.

---

## 6. Quick reference

| Symptom | First check |
|---------|-------------|
| Dashboard “disk” + latency | **iostat** + cloud volume — **throttling** vs **full** |
| **99%** but confusing who fails | **tune2fs** reserved blocks, **user** running nginx |
| `/var/log/nginx/*.log` huge | **copy to S3/GCS first**, then rotate/truncate; logrotate + `reopen` |
| **Shipper** down | Fix shipper **and** ensure **local** rotation still caps disk |
| `df -i` 100% | many small files → `find` + count + cleanup |
| Space not freed after delete | `lsof` deleted files → **reopen** / reload per **§5** |
| `/var/lib/nginx` huge | client/proxy temp paths, upstream slowness |
| `/tmp`, cores, apt | §1.5 quick wins |

---

**Bottom line:** On a busy **NGINX LB**, **log growth** is the most common cause; **I/O limits** and **reserved blocks** explain many **false** “disk” stories; **recovery** is **copy-first**, **compliance-aware**, and **reload** is a **production** decision — not a casual shell habit.
