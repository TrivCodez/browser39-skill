---
name: browser39-websearch
description: Use this skill whenever you need to search the web using browser39's MCP tools. Handles CAPTCHA avoidance via automatic search engine rotation and optimises browsing speed using browser39's content preselection system. Triggers when using browser39_search, browser39_fetch, or any browser39 tool to retrieve web content. Replaces naive single-engine searches with a resilient, token-efficient strategy.
license: MIT
version: 1.0.0
---

# browser39 Web Search Skill

This skill defines how to use browser39's MCP tools effectively: avoiding CAPTCHA walls, minimising token usage, and maximising fetch speed through targeted content retrieval.

## Core Principle

Never blindly dump a full page. browser39's content preselection exists for a reason — always use it. Always treat a CAPTCHA or bot-detection response as a signal to rotate engines, not to retry the same one.

## Search Engine Priority Order

Use engines in this order. Move to the next if you detect a CAPTCHA or bot wall.

| Priority | Engine | Config URL | Notes |
|----------|--------|------------|-------|
| 1 | Brave | `https://search.brave.com/search?q={}&source=web` | Independent index, low bot friction |
| 2 | Mojeek | `https://www.mojeek.com/search?q={}` | Independent crawler, extremely bot-friendly |
| 3 | Bing | `https://www.bing.com/search?q={}` | High tolerance, broad coverage |
| 4 | SearXNG | `https://searx.be/search?q={}&format=html` | Meta-search, no tracking, rarely CAPTCHAs |
| 5 | Startpage | `https://www.startpage.com/search?q={}` | Proxies Google, privacy-safe |
| 6 | Ecosia | `https://www.ecosia.org/search?q={}` | Bing-backed, low bot detection |
| 7 | Google | `https://www.google.com/search?q={}` | Last resort — most aggressive bot detection |

DuckDuckGo is intentionally excluded from the rotation as the primary engine due to aggressive bot detection that consistently triggers unsolvable CAPTCHA challenges for headless agents. It may be tried as an 8th-tier fallback only.

## Switching the Active Engine

Switch engines in-session without restarting browser39. Changes take effect immediately.

```
browser39_config_set key="search.engine" value="https://search.brave.com/search?q={}&source=web"
```

## CAPTCHA Detection

After calling browser39_search or browser39_fetch on a search results page, check the returned Markdown for any of these signals before processing results:

**Hard CAPTCHA signals (rotate engine immediately):**
- Text contains: "verify you are human", "complete the captcha", "unusual traffic", "robot", "I'm not a robot", "are you a bot", "prove you're human", "security check"
- Response is very short (<200 tokens) with no search result links
- Page title is "Just a moment", "Access Denied", "Robot Check", "Please verify", "Attention Required"
- Content contains Cloudflare Turnstile or hCaptcha markers

**Soft signals (try once with a header tweak, then rotate):**
- Zero results returned but query is not obscure
- HTML contains a redirect or meta refresh with no content
- Response is an error page or login wall

## CAPTCHA Rotation Protocol

When a hard CAPTCHA signal is detected:

1. **Do NOT retry the same engine.**
2. Move to the next engine in the priority table above.
3. Set the new engine via browser39_config_set.
4. Re-issue the search with browser39_search.
5. If the first 3 engines all CAPTCHA, apply the User-Agent Header Fix (below) before continuing down the list.
6. Log which engines were tried so you don't cycle back to a blocked one within the same session.

## User-Agent Header Fix

Apply this once if multiple engines block you. Sets a realistic desktop browser UA:

```
browser39_config_header_set domain="*" header="User-Agent" value="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36"
```

You can also set an Accept-Language header to appear more human:

```
browser39_config_header_set domain="*" header="Accept-Language" value="en-AU,en;q=0.9"
```

## Speed Optimisation: Content Preselection

browser39 returns a section map on first fetch instead of the full page. Always use this two-step pattern for any informational page fetch:

### Step 1 — Fetch sections list (cheap)

```
browser39_fetch url="https://example.com/article"
```

This returns available content sections with token estimates. Read section titles only — do NOT process the full content yet.

### Step 2 — Fetch only the target section (precise)

```
browser39_fetch url="https://example.com/article" selector="#section-heading"
```

Or use a CSS selector for the content container:

```
browser39_fetch url="https://example.com/article" selector="article.main-content"
```

**Token savings: up to 75× vs fetching the full page.**

## For Search Results Pages

After browser39_search, results are already Markdown-formatted. Extract the relevant result URLs from the list, then fetch only those URLs with targeted selectors rather than fetching all results wholesale.

## Speed Optimisation: DOM Queries

When you need a specific data point from a page (a price, a date, a stat), use browser39_dom_query instead of fetching and scanning the whole page:

```
browser39_dom_query script="document.querySelector('.price').textContent"
browser39_dom_query script="document.querySelectorAll('h2').length"
```

This avoids a full fetch entirely and returns a single value in minimal tokens.

## Session Reuse

browser39 persists cookies and session state to disk (AES-256-GCM encrypted). Do not re-authenticate or re-configure between searches in the same session. Reuse existing session state by default; only call browser39_config_set when you need to change engines or fix headers.

## Full Search Workflow (Reference)

1. `browser39_config_show section="search"` → Confirm current engine. If it's DuckDuckGo, switch before starting.
2. `browser39_search query="your query here"` → Scan result Markdown for CAPTCHA signals (see above).
3. If CAPTCHA → rotate engine → repeat step 2.
4. From results, pick 1–3 most relevant URLs.
5. `browser39_fetch url="<result URL>"` → Read section map only, identify target section.
6. `browser39_fetch url="<result URL>" selector="<target selector>"` → Read only the needed content.
7. If a specific value is needed → use browser39_dom_query instead of step 5–6.

## Quick-Reference: Engine URLs

Copy-paste ready for `browser39_config_set key="search.engine" value="...":`

```
# Brave (default recommended)
https://search.brave.com/search?q={}&source=web

# Mojeek (most bot-tolerant)
https://www.mojeek.com/search?q={}

# Bing
https://www.bing.com/search?q={}

# SearXNG (public instance)
https://searx.be/search?q={}&format=html

# Startpage (Google proxy)
https://www.startpage.com/search?q={}

# Ecosia
https://www.ecosia.org/search?q={}

# Google (last resort)
https://www.google.com/search?q={}
```

## Do NOT

- Do NOT use DuckDuckGo as the primary or early fallback engine.
- Do NOT retry the same engine after a CAPTCHA signal.
- Do NOT fetch full pages when section preselection is available.
- Do NOT loop through all results — pick 1–3 targeted URLs maximum.
- Do NOT re-configure headers or engines on every search; set once per session.
