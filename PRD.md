## Watson Smart Redirects — Product Requirements Document (PRD)

### Product name
- Watson Smart Redirects

### Problem statement
- Migrate legacy traffic from old WordPress slugs and structures to the new Webflow site without SEO loss. No user or bot should reach a 404 from any legacy URL; all final hops must return 200 on the new site.

### Objectives and KPIs
- 100% of legacy URLs either:
  - 301 to a final 200 on `www.watsoncreative.com`, or
  - appear in a documented ignore list with reason.
- Max redirect chain length: 2.
- All redirects are 301 (no 302s).
- No locale misrouting; rules duplicated per locale when localization is enabled.
- Preserve full query strings on all redirects.

### Users
- External visitors from organic search
- Crawlers/bots
- Internal link traffic from legacy content

### In scope
- Canonical folder-level remaps that preserve the leaf slug
- Exact-match exceptions for renamed slugs via Worker KV
- Non-www to www normalization
- Locale-prefix duplication of rules when localization is enabled
- Automated outputs for Webflow import and Worker KV
- Automated test matrix and runner

### Out of scope
- Predictive/fuzzy redirects at runtime in the Worker
- AI matching in production traffic paths
- Changes to Webflow CMS structure

### Constraints and assumptions
- Webflow 301 engine supports literal paths, `(.*)` capture, and `%1` backrefs. Certain characters in Old path require `%` escaping; rules do not auto-apply across locale prefixes; conflicting slugs/pages can block redirects. Webflow applies rules top-to-bottom. After each import, verify with spot checks before publish. See: [Set 301 redirects](https://help.webflow.com/hc/en-us/articles/33961294898835-Set-301-redirects-to-maintain-SEO-ranking?utm_source=chatgpt.com), [Special character redirects](https://webflow.com/updates/special-character-redirects-support?utm_source=chatgpt.com).
- Site is fronted by a Cloudflare Worker for flexible logic and explicit exceptions.
- Old site URLs are provided as XML sitemaps in `_data/oldsite/*.xml`.
- Live URLs are sourced from `https://www.watsoncreative.com/sitemap.xml` (follow nested sitemap indexes).
- Preserve query strings on all redirects.
- Primary risks: slug conflicts in Webflow, locale duplication gaps, chains >2, and redirect loops.

### Functional requirements
1) Canonical folder remaps (preserve leaf slug)
   - Example rules:
     - `^/portfolio-items/(.+?)/?$` → `/portfolio/$1`
     - `^/services-home/(.+)$` → `/services/$1`
   - Normalization: treat `/path` and `/path/` equivalently (trailing slash normalization), collapse duplicate slashes, lowercase paths.
   - Webflow-compatible expression uses `%` escaping in Old path only where required. Example proven: `/portfolio%-items/(.*)` → `/portfolio/%1`.

2) Exact-match exceptions for renamed slugs (Worker KV)
   - Detect old → new slug changes (e.g., `slice-of-a-city` → `slice-of-a-city-portland-oregon-gift`) and store an explicit path-to-path mapping in KV.
   - KV entries must preserve query strings at runtime.
   - KV JSON schema:
```json
{
  "old_path": "/portfolio-items/slice-of-a-city",
  "new_path": "/portfolio/slice-of-a-city-portland-oregon-gift",
  "section": "portfolio",
  "confidence": 0.94,
  "needs_review": false
}
```

3) Content decommission policy (410 Gone)
   - Maintain a first-class `410_GONE` list for intentionally retired content. The Worker returns `410` for these exact paths.
   - Document policy: the 410 list is for deliberately removed content; the ignore list is for legacy garbage or spam URLs that should remain 404.

4) Localization handling and safety rails
   - Duplicate folder rules and KV exceptions across locale prefixes (e.g., `/en/...`). Reattach locale prefix post-map.
   - Do not cross-map slugs between locales.
   - Parse hreflang from the live sitemap when localization is enabled and ensure compare logic does not propose cross-locale mappings. See: [Create a sitemap in Webflow](https://help.webflow.com/hc/en-us/articles/33961355371667-Create-a-sitemap-in-Webflow?utm_source=chatgpt.com).

5) Host normalization
   - Redirect apex and non-www to `www.watsoncreative.com` without dropping path or query.

6) Normalization of tricky URL variants (Worker)
   - Apply percent-decoding where safe, normalize diacritics in slugs, handle spaces and encoded characters, lowercase paths, collapse duplicate slashes, and normalize trailing slashes (except root). Add tests for `é`, spaces, and encoded characters. Reference: [Webflow forum note on diacritics](https://discourse.webflow.com/t/301-redirects-with-diacritical-marks/144791?utm_source=chatgpt.com).

7) Rule synthesis pipeline
   - Parse `_data/oldsite/*.xml` and the live sitemap, following nested sitemap indexes.
   - Normalize to lowercase, hostless paths with leading slash; strip trailing slash except root; collapse duplicate slashes; optionally percent-decode.
   - Apply folder rules to generate 1:1 candidates.
   - Mark candidates whose targets exist in live set as “Webflow-eligible”.
   - For unmatched old paths, propose best targets by section-limited token similarity (e.g., hyphen-token Jaccard). Use proposals for human review exceptions (in KV), not for autonomous runtime matching.
   - Require a sitemap count parity test (old vs. ingested; live vs. ingested including nested indexes and hreflang sets) and report any gaps.

8) Safety checks against loops and long chains
   - Pre-deploy loop detector script: simulate each redirect until terminal; reject any A→B→A or chains > 2.
   - In Worker, guard against self-redirect by comparing `request.pathname` to target before issuing a 301.

9) Worker performance and caching
   - Use Worker Cache API to cache successful 301 responses at edge; continue to use KV for exceptions. KV is the source of truth for exceptions and can be updated without redeploy. See: [Workers Redirect example](https://developers.cloudflare.com/workers/examples/redirect/?utm_source=chatgpt.com), [Cloudflare SEO blog](https://blog.cloudflare.com/diving-into-technical-seo-cloudflare-workers/?utm_source=chatgpt.com).

10) Observability and monitoring
   - Log: old path, decision type (Webflow vs KV vs fallthrough), target, HTTP status, and timing.
   - Rollout monitoring: use Cloudflare Analytics, `wrangler tail` during deployment, and export a weekly CSV of top misses to seed new KV entries. Reference: [Cloudflare Community example](https://community.cloudflare.com/t/example-of-using-cloudflare-workers-to-cache-301-responses/657839?utm_source=chatgpt.com).

11) Webflow import discipline
   - Group generic folder rules at the top and explicit page rules below; verify with spot checks; then publish (changes apply only after publish). Re-verify post-publish.

12) Search Console hygiene
   - Submit the new sitemap, annotate migration date, and monitor Not Found metrics to validate KPIs and catch regressions quickly.

### Outputs for implementation
- `out/redirects_webflow.csv` → two columns `old_path,new_path`, only 1:1 folder-like rules Webflow can own (use `%` escapes in Old path when needed).
- `out/redirects_kv.json` → exact `old_path:new_path` pairs for renamed slugs; include `confidence` and `needs_review` flags.
- `out/redirects_testmatrix.txt` → list of old URLs to test with curl.
- `out/redirects_gone_410.txt` → one path per line for intentionally retired content to be served as HTTP 410 by Worker.
- `out/ignore_404.txt` → one path per line for legacy garbage/spam to remain 404 (documented with reasons).

### Non-functional requirements
- Redirect latency target < 50 ms at edge (Worker) with caching.
- No loops; guard against self-redirects and cycles.
- Observability: log old path, decision (Webflow rule vs KV), target, status, and timing; weekly top-misses review.

### Deliverables
- Final PRD (this document).
- `out/redirects_webflow.csv` ready for import to Webflow UI.
- `out/redirects_kv.json` for Worker KV loader.
- `out/redirects_testmatrix.txt` for batch testing.
- `out/redirects_gone_410.txt` and `out/ignore_404.txt` with policy notes.
- Test plan (below).
- Rollback plan (below).
- Brief runbook for Webflow escaping rules, rule order, and locale duplication steps; include “how to verify” checklist for Webflow imports and publish.

### Success criteria (acceptance)
- Every old URL from `_data/oldsite` either:
  - terminally resolves to 200 within ≤2 hops via 301s, or
  - is in ignore list with rationale, or
  - is in `410_GONE` and returns 410.
- All hops are 301.
- Query strings preserved end-to-end.
- Locale-prefixed requests resolve into the correct locale path when localization is on; no cross-locale mappings.
- No redirect loops.

### Test plan
- Unit tests:
  - Path normalization (case folding, duplicate slashes, trailing slash, percent-decoding, diacritics, query preservation).
  - Folder-rule application and Webflow eligibility detection.
  - Similarity scoring and thresholding with `needs_review`.
  - Loop detection utility and self-redirect guard.
- Integration tests:
  - Generate all outputs under `out/` and assert sitemap count parity (including nested indexes and hreflang when present).
  - Dry-run Worker in preview: confirm host normalization, KV hits, regex rule hits, 410 handling, and no loops.
- End-to-end curl suite:
  - Single URL spot checks:
```bash
curl -IL "https://www.watsoncreative.com/portfolio-items/slice-of-a-city?utm_source=test"
```
    - Assert final 200, hops ≤ 2, all 301, and query preserved (verify `Location` and final URL).
  - Batch:
```bash
xargs -n1 -P8 curl -sIL < out/redirects_testmatrix.txt | tee out/testlog.txt
```
- Negative tests:
  - Paths in `out/redirects_gone_410.txt` return 410.
  - Paths in `out/ignore_404.txt` remain 404.

### Rollback plan
- Webflow:
  - Export current redirect rules before import; re-import to restore on failure.
- Worker / Cloudflare:
  - Maintain versioned KV namespace; revert to prior KV dataset on failure.
  - Roll back Worker to previous version.
  - Temporarily disable Worker routes (one-click remove routes) to let Webflow handle traffic while KV or code is rolled back.
- Monitoring:
  - If elevated 404s or loops detected, disable Worker routes, remove new Webflow rules if needed, adjust thresholds, and re-run exception list.

### References
- Webflow 301 redirects and escaping: [Set 301 redirects to maintain SEO ranking](https://help.webflow.com/hc/en-us/articles/33961294898835-Set-301-redirects-to-maintain-SEO-ranking?utm_source=chatgpt.com)
- Webflow special characters support: [Special character redirects support](https://webflow.com/updates/special-character-redirects-support?utm_source=chatgpt.com)
- Webflow forum (diacritics): [301 redirects with diacritical marks](https://discourse.webflow.com/t/301-redirects-with-diacritical-marks/144791?utm_source=chatgpt.com)
- Webflow sitemap and hreflang: [Create a sitemap in Webflow](https://help.webflow.com/hc/en-us/articles/33961355371667-Create-a-sitemap-in-Webflow?utm_source=chatgpt.com)
- Cloudflare Workers redirect example: [Redirect - Workers](https://developers.cloudflare.com/workers/examples/redirect/?utm_source=chatgpt.com)
- Cloudflare SEO blog: [Diving into Technical SEO using Cloudflare Workers](https://blog.cloudflare.com/diving-into-technical-seo-cloudflare-workers/?utm_source=chatgpt.com)
- Cloudflare Community: [Cache 301 responses example](https://community.cloudflare.com/t/example-of-using-cloudflare-workers-to-cache-301-responses/657839?utm_source=chatgpt.com)
- BMAD Method (planning flow): [GitHub - BMAD-METHOD](https://github.com/bmad-code-org/BMAD-METHOD)
- MCP explainer for stakeholders: [Why product people should know about MCP](https://maa1.medium.com/why-product-people-should-know-about-mcp-and-what-they-should-know-about-it-df8749ea57cf?utm_source=chatgpt.com)

### Notes and examples
- Proven Webflow rule pattern: `/portfolio%-items/(.*)` → `/portfolio/%1`
- Example renamed slug handled via KV: `/portfolio-items/slice-of-a-city` → `/portfolio/slice-of-a-city-portland-oregon-gift`
