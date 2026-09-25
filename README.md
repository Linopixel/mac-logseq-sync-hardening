# macOS Sync Daemon Hardening for Logseq DB

[![macOS Monterey](https://img.shields.io/badge/macOS-Monterey%2012-blue)](https://www.apple.com/macos)
[![Java](https://img.shields.io/badge/Java-21%20Temurin-orange)](https://adoptium.net/)
[![Python](https://img.shields.io/badge/Python-3.14-blue)](https://www.python.org/)
[![Status](https://img.shields.io/badge/Service-Production--Ready-brightgreen)](#)

Automated, crash-resilient, and self-healing local synchronization daemon for **Logseq DB** running on macOS via `launchd`.

This repository provides an enterprise-grade setup to harden the local Logseq sync server (`db_sync_server.py`), overcoming stale PID locks, forced system reboots, and non-interactive `launchd` environment restrictions.

---

## 🛠️ Key Architectural Solutions

### 1. Kernel-Level Concurrency Control (`fcntl.flock`)
- **Inode Lock Ownership**: Replaced fragile PID-file validation and `os.kill` polling with atomic file descriptor locking (`fcntl.LOCK_EX | fcntl.LOCK_NB`).
- **Reboot & Crash Resilience**: Eliminated `atexit` cleanups. If the system panics or undergoes a forced reboot, the Unix kernel automatically releases the file descriptor lock, eliminating *stale PID* locks upon startup.
- **Graceful Duplicate Handling**: Secondary instances or race-condition invocations exit gracefully (`sys.exit(0)`), preventing aggressive restart loops in `launchd`.

### 2. Isolated `launchd` Environment
- Explicitly injects non-interactive environment variables (`JAVA_HOME` for Eclipse Temurin JDK 21 and complete system `PATH`) into the `.plist` agent, eliminating dependency on interactive `.zshrc` or Terminal sessions.

---

## 📁 File & Path Mapping

> **Note**: Replace `<username>` with your actual macOS short username (or `$USER`).

| Item | Path / Location |
| :--- | :--- |
| **Sync Script** | `/Users/<username>/logseq-code/cli-e2e/scripts/db_sync_server.py` |
| **Compiled Adapter** | `/Users/<username>/logseq-code/deps/db-sync/node-adapter.js` |
| **LaunchAgent Manifest** | `~/Library/LaunchAgents/com.logseq.sync.plist` |
| **Data & PID Directory** | `/Users/<username>/logseq-server/` |
| **Daemon Logs** | `/Users/<username>/logseq-server/launchd_stdout.log`<br>`/Users/<username>/logseq-server/launchd_stderr.log` |

---

## ⚙️ Prerequisites

- **macOS**: Monterey 12 (Intel x64 / Apple Silicon) or newer
- **Java Runtime**: Eclipse Temurin JDK 21
- **Node.js**: v24+ & `pnpm`

---

## 🚀 Installation & Deployment

### 1. Build the Node Adapter
Ensure the Clojure/JS sync adapter is compiled against Java 21:

```bash
cd ~/logseq-code/deps/db-sync
export JAVA_HOME=$(/usr/libexec/java_home -v 21)
pnpm install
pnpm build:node-adapter
```

### 2. Configure & Deploy the LaunchAgent
Copy `com.logseq.sync.plist` to your local `LaunchAgents` folder, update all `/Users/<username>/` paths to match your system account, and apply the correct file permissions:

```bash
cp com.logseq.sync.plist ~/Library/LaunchAgents/
chmod 644 ~/Library/LaunchAgents/com.logseq.sync.plist
plutil ~/Library/LaunchAgents/com.logseq.sync.plist
```

### 3. Register and Start the Daemon

```bash
launchctl load -w ~/Library/LaunchAgents/com.logseq.sync.plist
```

---

## 📊 Operation & Monitoring

### Check Daemon Status
```bash
launchctl list | grep com.logseq.sync
```
*Expected Output:* `<PID> 0 com.logseq.sync` (A positive PID and exit code `0`).

### Inspect Real-Time Logs
```bash
tail -f ~/logseq-server/launchd_stdout.log
tail -f ~/logseq-server/server.log
```

---

## 🌐 Endpoint Mapping

| Client | Protocol / Address |
| :--- | :--- |
| **Logseq Desktop (Local)** | `http://127.0.0.1:8080` |
| **Logseq Mobile (LAN)** | `http://<your-local-ip>:8787` (or direct API via `:8080`) |

---

## 📄 License
MIT License.
