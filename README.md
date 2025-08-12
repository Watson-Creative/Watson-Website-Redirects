<a href="https://dash.cloudflare.com/?to=/:account/workers-and-pages/create/deploy-to-workers&amp;repository=https://github.com/Watson-Creative/Watson-Website-Redirects" target="_blank" rel="noopener"><img src="https://deploy.workers.cloudflare.com/button" alt="Deploy to Workers"></a>

## Watson Smart Redirects

A migration helper to move legacy WordPress URLs to the new Webflow site without SEO loss. It synthesizes Webflow-ready 301 rules, generates Cloudflare Worker KV exceptions for renamed slugs, and provides a test matrix and guardrails against loops and long chains.

See the full PRD: [PRD.md](./PRD.md)

### What this does (goals)
- **Zero 404s from old URLs**: All legacy URLs either 301 to a final 200 on `www.watsoncreative.com` or are documented (ignore/410).
- **Preserve SEO signals**: All hops are 301; query strings are preserved end‑to‑end; max chain length is 2; no loops.
- **Locale safety**: When localization is enabled, rules duplicate per locale prefix without cross‑locale mapping.

### How it works (high‑level)
- **Webflow 301 rules**: Owns canonical folder‑level remaps that preserve leaf slugs (e.g., `/portfolio%-items/(.*)` → `/portfolio/%1`).
- **Cloudflare Worker + KV**: Exact path‑to‑path exceptions for renamed slugs; handles normalization (case, slashes, trailing slash, encoding/diacritics), host normalization, and a first‑class 410 list.
- **Rule synthesis pipeline**: Parses old and live sitemaps, normalizes paths, applies folder rules, marks Webflow‑eligible targets, and proposes KV exceptions for human review.

### Inputs
- **Old site**: XML sitemaps in `_data/oldsite/*.xml`.
- **Live site**: `https://www.watsoncreative.com/sitemap.xml` (follows nested indexes; parses hreflang when enabled).

### Outputs (under `out/`)
- `redirects_webflow.csv`: Two columns `old_path,new_path` for import into Webflow 301 UI.
- `redirects_kv.json`: Exact `old_path:new_path` pairs with `confidence` and `needs_review` flags for Worker KV.
- `redirects_testmatrix.txt`: Old URLs to test via curl.
- `redirects_gone_410.txt`: Paths that should return HTTP 410 from the Worker.
- `ignore_404.txt`: Paths intentionally left to 404 (documented with reasons).

### Acceptance criteria
- Every legacy URL either resolves to a final 200 within ≤2 hops via 301s, is in the ignore list (with rationale), or is served 410 (in `410_GONE`).
- All redirects are 301; query strings preserved; no loops; locale‑prefixed requests resolve correctly.

### Test plan (brief)
- **Unit**: Path normalization, folder‑rule application, similarity thresholding, loop detection/self‑redirect guard.
- **Integration**: Generate all artifacts under `out/`; assert sitemap count parity (including nested indexes and hreflang where present); dry‑run Worker.
- **End‑to‑end curl**: Spot check and batch test. Example:
```bash
curl -IL "https://www.watsoncreative.com/portfolio-items/slice-of-a-city?utm_source=test"
```

### Out of scope
- Predictive/fuzzy runtime redirects in production; AI matching in live traffic; Webflow CMS structure changes.

### Rollback (high‑level)
- **Webflow**: Export current redirect rules before import; re‑import to restore if needed.
- **Cloudflare**: Versioned KV dataset and Worker code; revert KV and/or roll back Worker; temporarily disable Worker routes to fall back to Webflow.

### References
- Webflow 301 redirects and escaping: Set 301 redirects to maintain SEO ranking
- Webflow special characters support: Special character redirects support
- Diacritics discussion: 301 redirects with diacritical marks
- Cloudflare Workers example: Redirect – Workers
- Cloudflare SEO blog: Diving into Technical SEO using Cloudflare Workers
