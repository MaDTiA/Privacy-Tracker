<div align="center">

# 🛡️ Privacy Tracker

**Self-hosted UDP BitTorrent tracker · No IP logging · Flask web UI · REST API**

[![Python](https://img.shields.io/badge/Python-3.x-3776ab?logo=python&logoColor=white)](https://python.org)
[![Flask](https://img.shields.io/badge/Flask-web%20layer-000000?logo=flask)](https://flask.palletsprojects.com)
[![Protocol](https://img.shields.io/badge/UDP-BEP%2015-orange)](http://bittorrent.org/beps/bep_0015.html)
[![IPv6](https://img.shields.io/badge/IPv6-supported-blueviolet)](#)
[![License](https://img.shields.io/badge/license-MIT-22c55e)](#license)

</div>

---

## What it is

A **self-hosted BitTorrent tracker** written in Python. It speaks the **UDP tracker protocol (BEP 15)**, tracks peers for any info-hash, and exposes a web interface for stats and torrent search — without logging a single client IP address to disk.

Public trackers log every announce request, revealing which torrents your peers are downloading. Privacy Tracker gives you a tracker you control, on your own server, that never persists IP addresses — suitable for privacy-conscious communities and private groups.

---

## Use cases

- 🔒 Private torrent community or closed-group tracker
- 🔗 Self-hosted fallback tracker embedded in your magnet links
- 🧪 Developer tool — test BitTorrent clients locally without relying on public trackers
- 📊 Research — monitor swarm sizes for specific info-hashes
- ⚡ Reliability layer — add a known-good tracker to your magnet URIs

---

## Official Nodes

Add these to your torrent client or magnet links for the best availability:

| Node | Announce URL |
|------|-------------|
| 🟢 coeus | `udp://coeus.torrentonline.cc:42069/announce` |
| 🟢 whybother | `udp://whybother.torrentonline.cc:42069/announce` |
| 🟢 obey | `udp://obey.torrentonline.cc:42069/announce` |
| 🟢 archive | `udp://archive.torrentonline.cc:42069/announce` |

**Live node status:** [public-stats.trackerstatus.live](https://public-stats.trackerstatus.live/)

**Example magnet snippet:**
```
&tr=udp://coeus.torrentonline.cc:42069/announce
&tr=udp://whybother.torrentonline.cc:42069/announce
&tr=udp://obey.torrentonline.cc:42069/announce
&tr=udp://archive.torrentonline.cc:42069/announce
```

---

## Stack & architecture

| Component | Tech |
|-----------|------|
| Tracker daemon | Python 3 · `socket` · `struct` · `threading` |
| Web layer | Flask · Markupsafe · Werkzeug |
| External search | BTDigg API · ITorrents |
| Persistence | JSON files (`stats.json` · `names.json`) |
| Process management | `systemd` (`Restart=always`) |
| Network | IPv4 + optional IPv6 (UDP) |

Single `.py` file. No database required. Designed for a **1-core VPS** with low memory footprint. The script embeds both the UDP tracker daemon and the Flask web app, auto-generates the inner daemon file, and manages both via one `systemd` unit.

---

## Quick start

```bash
# 1. Install dependencies
pip install flask markupsafe werkzeug

# 2. Create config directory
mkdir -p /opt/tracker
# Place your config.json in /opt/tracker/ with port, password, admin path

# 3. Run directly (development)
python tracker_v15_final.py

# 4. Run as systemd service (production)
sudo systemctl enable --now tracker
```

**Web UI:**
```
https://yourdomain.com/stats          # public stats + torrent search
https://yourdomain.com/ADMINPATH      # admin panel
```

---

## REST API

| Endpoint | Description |
|----------|-------------|
| `GET /api/stats` | Global stats — torrents, peers, queries, uptime |
| `GET /api/torrent/<hash>` | Single torrent stats by info-hash |
| `GET /api/search?q=QUERY&source=all` | Search local tracker + BTDigg |
| `GET /api/list?limit=50&offset=0` | Paginated torrent list sorted by peers |
| `GET /api/resolve/<hash>` | Full hash resolution — cache + ITorrents + BTDigg |
| `GET /api/lookup?hash=HASH` | Fast local name lookup |
| `GET /api/name2hash?q=NAME` | Reverse name-to-hash search |

API keys with per-key hourly rate limits are managed from the admin panel. Pass the key via `?key=` query param or `X-API-Key` header.

---

## Changelog

---

### [v15] — April 2026 · `Latest`

> **Resilience & Resource Control** — circuit breakers, non-blocking external HTTP, smarter auto-refresh, BTDigg bandwidth optimisation.

<br>

#### ✨ New — Circuit Breaker (`CB` class)

New `CB` class wraps all calls to ITorrents and BTDigg. Once a service fails **3 times in a row**, the circuit opens and calls are skipped immediately — no hanging threads, no timeout pile-up in the worker pool.

```python
cbit = CB("ITorrents", threshold=3, resetsecs=120)  # resets every 2 min
cbbt = CB("BTDigg",    threshold=3, resetsecs=300)  # resets every 5 min
```

Thread-safe via internal `threading.Lock`. Self-healing: circuit closes automatically on the next successful call (`cb.ok()`).

<br>

#### ✨ New · ⚡ Perf — Non-blocking external HTTP via Thread Pool

All outbound HTTP (BTDigg, ITorrents) now runs in a dedicated `ThreadPoolExecutor` instead of blocking Flask worker threads directly. In v14, a slow upstream would block the entire worker thread for up to 15 s, stalling all concurrent requests.

```python
# Capped at 4 workers for single-core VPS
extpool = ThreadPoolExecutor(max_workers=4, thread_name_prefix="ext")

def btdigsearch(query, limit=10):
    if cbbt.isopen(): return []
    try:
        fut = extpool.submit(btdigsearchinner, query, limit)
        res = fut.result(timeout=10)   # hard 10 s wall-clock deadline
        cbbt.ok()
        return res
    except FutureTimeoutError:
        cbbt.fail()
        return []
```

<br>

#### ⚡ Perf — BTDigg: timeout `15s → 8s`, buffer `512 KB → 256 KB`

BTDigg result markup appears in the first ~100 KB of the response. Reading 512 KB was wasteful in both bandwidth and parse time. Combined with the new 10 s wall-clock cap, worst-case user-visible BTDigg latency drops from **~15 s → ~10 s**.

```diff
- with urllib.request.urlopen(req, timeout=15) as r:
-     html = r.read(512 * 1024).decode('utf-8', errors='replace')
+ with urllib.request.urlopen(req, timeout=8) as r:
+     html = r.read(256 * 1024).decode('utf-8', errors='replace')
```

<br>

#### 🔄 Changed · 🐛 Fix — Homepage: hard `<meta refresh>` → 3-shot JS refresh

The homepage no longer reloads unconditionally every 15 s. A JS session counter stops after **3 auto-reloads** and any user interaction (keydown / mousedown) cancels the next reload immediately — no more interrupted reading.

```diff
- <meta http-equiv="refresh" content="15">
+ <script>
+   var MAX=3, IV=15000, F="arf", C="arc";
+   var cnt = was ? parseInt(sessionStorage.getItem(C)||0, 10) : 0;
+   if (cnt < MAX) {
+     var tid = setTimeout(function(){
+       sessionStorage.setItem(F,1);
+       sessionStorage.setItem(C, String(cnt+1));
+       location.reload();
+     }, IV);
+     function cancel(){ clearTimeout(tid); }
+     document.addEventListener("keydown",  cancel, {once:true});
+     document.addEventListener("mousedown", cancel, {once:true});
+   }
+ </script>
```

<br>

#### 🔄 Changed — Stats page: infinite `setInterval` → 3-shot refresh

`setInterval(loadDef, 30000)` fired indefinitely as long as the stats tab was open, generating continuous `/api/stats` background traffic. The same 3-shot session-counter pattern now caps total background requests to 3 per session.

```diff
- setInterval(function(){
-   if(document.getElementById('utabs').style.display==='none') loadDef;
- }, 30000);
+ // 3-shot pattern (MAX=3, IV=30 000ms)
+ // sessionStorage counters "arsf" / "arsc"
+ // cancelled by any user keydown or mousedown
```

The **"Active Torrents"** heading also cleaned up: static `(auto-refresh 30s)` text replaced with a dynamic `<span id="arstatslbl">` updated by JS.

---

### [v14] — Initial public release · `Baseline`

> First public release. The foundation from which all future changes are tracked.

<details>
<summary>View full feature list</summary>

- **UDP BitTorrent tracker (BEP 15):** IPv4 + optional IPv6, connection / announce / scrape, low-memory mode
- **Peer cleanup daemon** + periodic stats persistence to `stats.json`
- **`systemd` auto-restart** via `os.exit(0)` + `Restart=always`; configurable interval from admin panel
- **Flask web layer:** public stats page with live torrent list, unified search (local + BTDigg), hash resolution via ITorrents & BTDigg, legacy `?search=` query param support
- **Full REST API** — stats, torrent, search, list, resolve, lookup, name2hash
- **Admin panel:** password login, API key management with per-key hourly rate limits, ad slot injection (5 slots), Google Analytics code injection, tracker-list editor, public-stats toggle
- **Brute-force login protection:** 5 attempts → 15-minute lockout
- **Config & stats hot-reload** with TTL cache (30 s config / 10 s stats)
- Homepage hard auto-refresh via `<meta refresh="15">`; stats torrent list polled every 30 s via `setInterval`

</details>

---

## License

MIT
