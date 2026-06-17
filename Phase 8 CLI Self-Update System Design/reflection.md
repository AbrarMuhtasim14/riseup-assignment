# Reflection — CLI Self-Update System Design

I verified the GitHub Releases API by running `curl https://api.github.com/repos/python/cpython/releases/latest` and inspecting the JSON response — confirming `tag_name`, `assets[].browser_download_url`, and the 60 req/hr rate limit firsthand. I also tested `packaging.version.Version("2.0.0") > Version("1.9.3")` in a Python REPL to confirm semver comparison works as documented.

I initially assumed `os.rename()` would always work atomically, but after researching, I discovered it fails across filesystems (e.g., `/tmp` and `/home` on different mounts). I changed the runbook to recommend `shutil.move()` as a fallback, which I verified by testing a cross-device rename attempt that raised `OSError`.

The step I am least confident about is **Step 8: Swap on Windows**. Windows locks running `.exe` files, preventing in-place replacement. The shadow-copy-and-rename-on-restart pattern works but requires careful testing across Windows versions. I would need to verify this on an actual Windows machine before trusting it in production.

---

*AI was used to research the GitHub API structure and Windows file-locking behavior. I verified the API responses and Python code locally. The design decisions and failure-mode analysis are my own.*
