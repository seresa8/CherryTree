# Cherry Tree by Seresa

> AI-optimized content system for WordPress. AI drafts article ideas, humans approve, AI writes, humans publish — driven over a REST API consumable by Claude skills.

**Latest release: v4.0.1** · Requires PHP 8.3+ · WordPress 6.7+ (tested to 7.0.2) · License: BSL 1.1

---

## Download & install

1. Go to the [**Releases**](https://github.com/seresa8/CherryTree/releases/latest) page and download the latest `cherry-tree-by-seresa-<version>.zip`.
2. In WordPress admin: **Plugins → Add New → Upload Plugin**, choose the zip, **Install Now**, then **Activate**.
3. Once active, open **Cherry Tree → Settings → API Key** to view your auto-generated key.

Installed sites **self-update** from this repository — new releases appear as standard one-click plugin updates.

## Videos

📺 [Cherry Tree workflow tutorials on YouTube](https://www.youtube.com/playlist?list=PLWQqIKRrk2xs)

## How it works

```
idea → queued → review → published
  ↓        ↓        ↓
rejected rejected rejected
```

AI proposes article ideas via the REST API, a human approves, AI writes the draft, and a human publishes — with rejection possible at each stage.

## Requirements

- PHP 8.3+
- WordPress 6.7+ (tested up to 7.0.2)

## Support

Found a bug or have a feature request? Please open an [**Issue**](https://github.com/seresa8/CherryTree/issues/new/choose).

---

© 2026 SERESA PTE. LTD. (Singapore). Licensed under the Business Source License 1.1 (BSL 1.1).
