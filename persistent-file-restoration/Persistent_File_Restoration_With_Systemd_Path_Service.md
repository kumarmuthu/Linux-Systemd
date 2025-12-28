# Persistent File Restoration on Linux Using systemd Path + Service Units

A practical pattern to ensure critical files are restored automatically using systemd path and service units, without cron or daemons.

### Using systemd Path Units (Without Cron or Daemons)

## Why this problem exists

If you work with Linux automation, CI/CD, or managed systems, you’ve likely seen this happen:

* A reboot occurs
* A vendor service runs
* An automation job re-applies configuration
* Suddenly:

  * SSH keys disappear
  * JSON / manifest files reset
  * Access breaks unexpectedly

This usually isn’t a bug.

It’s automation doing exactly what it was designed to do.

The real issue is:

> **Critical files are overwritten occasionally — not continuously.**

So why do we often solve this with tools that run all the time?

---

## Common solutions (and why they are not ideal)

### ❌ Cron jobs

```bash
*/5 * * * * restore_files.sh
```

Problems:

* Runs even when nothing changed
* Wastes CPU and disk I/O
* Slow to react
* Adds operational noise

---

### ❌ Custom daemons or loops

Some teams write background processes that:

* Sleep
* Check files
* Restore if missing

Problems:

* Always running
* More complexity
* Needs monitoring
* Breaks during upgrades

---

## A simpler idea: react only when a file changes

Linux already provides the right tool:

> **systemd path units**

Instead of checking files repeatedly, systemd can **react only when a file changes**.

No polling
No loops
No cron
Almost zero CPU overhead

---

## The core pattern

This solution uses **two systemd units**:

1. **Path unit** – watches a file
2. **Service unit** – runs only when the file changes

Conceptually:

```
Target file changes
        ↓
systemd detects it
        ↓
One-shot service runs
        ↓
File is restored
```

---

## A critical concept: persistent storage is the source of truth

This is the most important idea in this pattern.

> **The watched (target) file is NOT the source of truth.**

The **persistent file is**.

---

## Why this matters

Automation, upgrades, or vendors may:

* Overwrite files
* Truncate them
* Recreate them from templates
* Replace them entirely

So instead of trying to “protect” the target file, we do this:

> **Always rebuild the target file from a persistent copy.**

---

## The data flow (always one-way)

```
Persistent storage (source of truth)
        │
        │  (latest approved content)
        ▼
Target file (can be overwritten anytime)
```

### Never reverse this automatically

```bash
# Correct direction
cp /var/lib/persist/important.json /etc/example/important.json
```

```bash
# ❌ Wrong direction (never automate this)
cp /etc/example/important.json /var/lib/persist/important.json
```

Updates to persistent storage must be:

* Manual
* Approved
* Or explicitly deployed by automation

Not inferred from a possibly corrupted target file.

---

## Example scenario

Let’s protect a file that automation often overwrites:

```
Target file:
/etc/example/important.json
```

We store the trusted version here:

```
Persistent file:
/var/lib/persist/important.json
```

Whenever the target file changes, it is automatically restored from persistent storage.

---

## Step 1: Restore script (single responsibility)

📄 `/usr/local/bin/restore-important-file.sh`

```bash
#!/bin/bash
set -e

SOURCE="/var/lib/persist/important.json"
TARGET="/etc/example/important.json"

mkdir -p "$(dirname "$TARGET")"

if [ ! -f "$SOURCE" ]; then
    echo "Persistent file missing; nothing to restore"
    exit 0
fi

cp "$SOURCE" "$TARGET"
chmod 600 "$TARGET"

echo "important.json restored successfully"
```

Make it executable:

```bash
chmod +x /usr/local/bin/restore-important-file.sh
```

---

## Step 2: systemd service unit (runs only when triggered)

📄 `/etc/systemd/system/important-file-restore.service`

```ini
[Unit]
Description=Restore important file from persistent storage

[Service]
Type=oneshot
ExecStart=/usr/local/bin/restore-important-file.sh
```

---

## Step 3: systemd path unit (the trigger)

📄 `/etc/systemd/system/important-file-restore.path`

```ini
[Unit]
Description=Watch important.json for changes

[Path]
PathModified=/etc/example/important.json
PathChanged=/etc/example/important.json

[Install]
WantedBy=multi-user.target
```

---

## Step 4: Enable and start

```bash
systemctl daemon-reload
systemctl enable important-file-restore.path
systemctl start important-file-restore.path
```

That’s it.

No daemon.
No cron.
No loop.

---

## Step 5: Monitoring and Debugging (journal / status commands)

Once enabled, you can inspect and debug the behavior using standard systemd tools.

### Check path unit status

```bash
systemctl status important-file-restore.path
```

This shows:

* Whether the path unit is active
* Which paths are being watched
* When it last triggered

---

### Check service execution status

```bash
systemctl status important-file-restore.service
```

Useful to verify:

* Whether the restore service ran
* Exit status
* Last execution time

---

### View detailed logs (journal)

```bash
journalctl -u important-file-restore.service
```

Follow logs in real time:

```bash
journalctl -u important-file-restore.service -f
```

---

### Verify path triggers

```bash
journalctl -u important-file-restore.path
```

This confirms that systemd detected the file change and triggered the service.

---

## What actually happens at runtime

1. The kernel reports a file change (inotify)
2. systemd notices it
3. The restore service runs once
4. The target file is rebuilt from persistent storage
5. CPU usage returns to zero

---

## Why this is ideal for CI/CD and automation

✔ Event-driven (not polling)
✔ Near-zero CPU usage
✔ Native systemd solution
✔ Deterministic recovery
✔ Survives reboots
✔ Easy to audit

Perfect for:

* SSH keys
* Manifest files
* License files
* JSON metadata
* System configuration fragments

---

## Mental model to remember

* **Persistent file** → database
* **Target file** → cache

Caches can be wiped.
Databases must not.

---

## When to use this pattern

Use it when:

* Files change occasionally
* Recovery is idempotent
* You want immediate correction

Avoid it when:

* Files change constantly
* You need bidirectional sync
* Version history is required (use Git)

---

## Final takeaway

> **Don’t repeatedly check if something broke.
> Let the system react when it actually breaks.**

systemd path units are one of Linux’s most underused features — simple, efficient, and perfectly suited for protecting critical files in automated environments.

---

## Author

Created and maintained by **[kumarmuthu](https://github.com/kumarmuthu)**.

---


