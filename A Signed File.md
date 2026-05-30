
# Test Note for Obsidian Red Signer

### modified on purpose to test failures.

This is a demo Markdown file intended to verify the cryptographic signing functionality of **Project R.E.D. Signer**.

## Purpose

- Test the one‑click signing workflow (ribbon icon / command palette).
- Verify that the `manifest.json` is correctly updated with the file hash and signature.
- Check the real‑time status bar indicator (`✓ Signed` / `Unsigned`).

## Sample Content

Below is a code block (should not affect hashing in a problematic way – the engine hashes the raw file content):

```bash
echo "Hello from the Project R.E.D. test grid"
```

A simple bullet list:
- Ed25519 signing
- Automatic manifest injection
- Zero‑friction maintainer experience

> **Maintainer note:** After modifying this file, the status will revert to `Unsigned` until you sign it again.

---

**Ready for signing.** ✅
