# CLI Self-Update Runbook

> Step-by-step procedure for how a CLI tool (`mycli`) updates itself from GitHub Releases.
> Each step includes: what happens, which command/API is used, what can go wrong, and how to recover.

---

## Step 1: Check — Read local version

**What happens:** The CLI reads its own current version from the source of truth.

**Command/API:**
```python
from mycli import __version__  # e.g. "1.4.2"
```
Alternative: parse `pyproject.toml`:
```bash
grep '^version =' pyproject.toml | head -1 | cut -d'"' -f2
```

**What can go wrong:**
- `__version__` not defined → `ImportError`
- `pyproject.toml` missing or not installed via editable mode

**Recovery:**
- Fall back to reading `pyproject.toml` directly via `tomllib` (Python 3.11+)
- If neither source exists, assume version `"0.0.0"` and force update check

---

## Step 2: Fetch — Get latest release metadata from GitHub

**What happens:** Query the GitHub Releases API to find the latest published version.

**Command/API:**
```bash
curl -s https://api.github.com/repos/owner/mycli/releases/latest
```
```python
import requests
resp = requests.get(
    "https://api.github.com/repos/owner/mycli/releases/latest",
    headers={"Accept": "application/vnd.github+json"},
    timeout=10
)
data = resp.json()
latest_tag = data["tag_name"]  # e.g. "v2.0.0"
assets = data["assets"]        # list of downloadable files
```

**What can go wrong:**
- Network timeout → `requests.exceptions.Timeout`
- Rate limited (60 req/hr unauthenticated) → HTTP 403
- No releases published → HTTP 404
- Repo doesn't exist → HTTP 404

**Recovery:**
- Timeout: retry up to 3 times with 2s backoff
- Rate limited: read `X-RateLimit-Reset` header, wait until reset, or use a GitHub token in env `GITHUB_TOKEN`
- No releases: log warning, skip update, exit gracefully

---

## Step 3: Compare — Determine if update is needed

**What happens:** Parse semver strings and compare local vs remote.

**Command/API:**
```python
from packaging.version import Version
is_update_available = Version(latest_tag.lstrip("v")) > Version(local_version.lstrip("v"))
```

**What can go wrong:**
- Non-semver tag (e.g. `"nightly-20240301"`) → `packaging.version.InvalidVersion`
- Pre-release tag (`2.0.0rc1`) incorrectly triggers update for users who don't want pre-releases

**Recovery:**
- Catch `InvalidVersion`, log warning, skip update
- Filter out pre-releases: `Version.is_prerelease` — skip if True, unless user opts in via `--pre` flag

---

## Step 4: Download — Fetch release asset to temp path

**What happens:** Download the appropriate release asset (.whl, .tar.gz, or binary) to a temporary staging directory.

**Command/API:**
```python
import tempfile, requests
temp_dir = tempfile.mkdtemp(prefix="mycli-update-")
asset_url = assets[0]["browser_download_url"]  # select correct asset
download_path = os.path.join(temp_dir, os.path.basename(asset_url))

resp = requests.get(asset_url, timeout=60, stream=True)
with open(download_path, "wb") as f:
    for chunk in resp.iter_content(chunk_size=8192):
        f.write(chunk)
```

**What can go wrong:**
- Network failure mid-download → incomplete file
- Disk full → `OSError: [Errno 28] No space left on device`
- Asset URL expired or removed → HTTP 404

**Recovery:**
- Incomplete download: delete partial file, retry up to 3 times with exponential backoff (2s, 4s, 8s)
- Disk full: abort, warn user, do NOT proceed to replace
- 404: re-fetch release metadata (Step 2) and retry with fresh URL

---

## Step 5: Verify — Checksum validation

**What happens:** Compute SHA-256 of the downloaded file and compare against the published checksum.

**Command/API:**
```python
import hashlib
def compute_sha256(filepath):
    h = hashlib.sha256()
    with open(filepath, "rb") as f:
        for chunk in iter(lambda: f.read(8192), b""):
            h.update(chunk)
    return h.hexdigest()

actual = compute_sha256(download_path)
expected = fetch_checksum_from_release(assets)  # .sha256 file in release
is_valid = actual == expected
```

**What can go wrong:**
- No checksum file published in the release
- Checksum mismatch (tampered or corrupted download)

**Recovery:**
- No checksum: log warning, proceed but flag as "unverified" in update log
- Mismatch: delete downloaded file, re-download once. If still fails, abort and report potential security issue

---

## Step 6: Stage — Extract/install into staging directory

**What happens:** Install the new version into a staging location without touching the current installation.

**Command/API:**
```bash
# For .whl via pip into a staging venv
pip install --target /tmp/mycli-update/staging/ mycli-2.0.0-py3-none-any.whl

# For binary: just keep in temp dir (already staged)
# For git-based: git clone --depth 1 --branch v2.0.0 https://github.com/owner/mycli.git /tmp/mycli-update/staging/
```

**What can go wrong:**
- `pip install` fails due to dependency conflict
- Staging directory already exists from a previous failed update
- Permission denied writing to staging path

**Recovery:**
- Dependency conflict: log the error, abort, suggest user runs `pip install --upgrade` manually
- Stale staging dir: `shutil.rmtree()` the old staging dir before extracting
- Permission: try `/tmp` (world-writable), or `~/.cache/mycli/update/` as fallback

---

## Step 7: Smoke Test — Verify the staged update works

**What happens:** Run the new version from the staging directory to confirm it starts without errors.

**Command/API:**
```bash
# For pip-installed staging
PYTHONPATH=/tmp/mycli-update/staging python -m mycli --version

# For binary
/tmp/mycli-update/staging/mycli --version
```

```python
import subprocess
result = subprocess.run(
    ["python", "-m", "mycli", "--version"],
    capture_output=True, text=True, timeout=10
)
is_healthy = result.returncode == 0 and new_version in result.stdout
```

**What can go wrong:**
- New version crashes on startup (breaking change or missing dependency)
- Smoke test times out (infinite loop / hang)
- Returns wrong version (staging didn't install correctly)

**Recovery:**
- Crash: log stderr, abort update, keep current version, report to user
- Timeout: kill process after 10s, abort update
- Wrong version: abort, clean staging dir, do not swap

---

## Step 8: Swap — Atomically replace old installation with staged

**What happens:** Replace the current installation with the new one in a single atomic operation.

**Command/API:**
```python
import os, shutil

# Method 1: Symlink swap (recommended for user-space installs)
# Current: ~/.local/bin/mycli -> ~/.local/lib/mycli-1.4.2/
# New:     ~/.local/bin/mycli -> ~/.local/lib/mycli-2.0.0/
os.symlink("/home/user/.local/lib/mycli-2.0.0", "/home/user/.local/lib/mycli-current-tmp")
os.rename("/home/user/.local/lib/mycli-current-tmp", "/home/user/.local/lib/mycli-current")
# rename is atomic on POSIX

# Method 2: pip install --upgrade (for venv installs)
# subprocess.run(["pip", "install", "--upgrade", download_path])
```

**What can go wrong:**
- `os.rename()` fails on cross-device (temp on /tmp, install on /home)
- Permission denied on the target directory
- Another update process running simultaneously (race condition)

**Recovery:**
- Cross-device: use `shutil.move()` instead (copies then deletes, not truly atomic but works)
- Permission: check write access before attempting swap, prompt for sudo only as last resort
- Race: use file lock `fcntl.flock()` on `~/.mycli-update.lock` — abort if locked

---

## Step 9: Restart — Spawn new process and exit old

**What happens:** The current CLI process starts the new version and then exits itself.

**Command/API:**
```python
import os, sys

# Method 1: os.execv — replace current process entirely
os.execv(sys.executable, [sys.executable, "-m", "mycli"] + sys.argv[1:])

# Method 2: spawn new, exit old (safer for cleanup)
import subprocess
subprocess.Popen([sys.executable, "-m", "mycli"] + sys.argv[1:])
sys.exit(0)
```

**What can go wrong:**
- New process fails to start
- Orphan processes if old exits before new is ready
- Lost command-line arguments or working directory

**Recovery:**
- Before exec, validate the new executable exists and is runnable
- Pass all original `sys.argv` and `os.getcwd()` to the new process
- If using Method 2, add a 1-second health check before exiting old process

---

## Step 10: Cleanup & Rollback

**What happens:** Remove temporary files. If anything failed at any step, restore the previous version.

**Command/API:**
```python
# Cleanup (on success)
shutil.rmtree(temp_dir, ignore_errors=True)

# Rollback (on failure at any step)
# The old installation was never removed (symlink swap preserves it)
# Simply point symlink back:
os.symlink(old_install_path, "/home/user/.local/lib/mycli-current-tmp")
os.rename("/home/user/.local/lib/mycli-current-tmp", "/home/user/.local/lib/mycli-current")
```

**What can go wrong:**
- Temp files not cleaned up → disk space leak over time
- Old installation was already deleted before swap (if using overwrite-in-place instead of symlink)
- Rollback symlink also fails

**Recovery:**
- Always use `tempfile.mkdtemp()` — OS cleans `/tmp` on reboot
- Never delete old version until new version passes smoke test — keep at least one previous version
- If rollback fails: log critical error, instruct user to manually reinstall: `pip install mycli==PREVIOUS_VERSION`

---

## Quick Reference: Update Lifecycle

```
CHECK → FETCH → COMPARE → DOWNLOAD → VERIFY → STAGE → SMOKE TEST → SWAP → RESTART → CLEANUP
  │                                                                      ▲
  └──── On failure at ANY step → ROLLBACK to previous version ──────────┘
```

## Environment Variables

| Variable | Purpose | Example |
|----------|---------|---------|
| `MYCLI_UPDATE_REPO` | Override default GitHub repo | `owner/mycli` |
| `MYCLI_UPDATE_CHANNEL` | stable / pre-release | `stable` |
| `GITHUB_TOKEN` | Auth for higher API rate limit | `ghp_xxxx` |
| `MYCLI_UPDATE_TIMEOUT` | Network timeout in seconds | `30` |

## Files Involved

| Path | Purpose |
|------|---------|
| `mycli/__init__.py` | Contains `__version__` |
| `~/.local/lib/mycli-current/` | Symlink to active version directory |
| `~/.local/bin/mycli` | CLI entry point |
| `~/.cache/mycli/update/` | Staging directory for downloads |
| `~/.mycli-update.lock` | File lock to prevent concurrent updates |
| `~/.mycli-update.log` | Update history log |
