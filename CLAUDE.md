# savage — Development Documentation

## Overview

savage serves a BDO record's `svg` property as a live webpage with Open Graph
preview tags. It is the thing that turns a published BDO into a shareable
link: BizBuz and Linkitylink both render a card to SVG, publish it to BDO, and
hand out a savage URL.

**Location**: `/savage/`
**Stack**: Node 18+, Express 4, ESM. No database, no build step.
**Extracted from allyabase** (was `allyabase/deployment/savage/`) so it can be
deployed on its own — see "History" below.

## Architecture

One file: `src/server/node/savage.js` (~170 lines). Three routes, all reading
through to BDO:

- `GET /user/:uuid/bdo` → HTML wrapper (`wrapHTML`)
- `GET /user/:uuid/bdo/svg` → the sanitized SVG, `image/svg+xml`
- `GET /user/:uuid/bdo/vcard` → the record's `vcard`, `text/vcard`, download

`fetchBdo` forwards the incoming query string verbatim to BDO's
`GET /user/:uuid/bdo`. savage holds no keys and performs no auth of its own —
the query string *is* the credential, pre-signed by whoever published the
record. That's what lets a savage URL be permanent: the signature doesn't
expire, and savage never needs to interpret it.

### Stateless by design

No persistence layer, no identity, no writes. Every request is a pass-through
plus a transform. This is why the service can be scaled or redeployed freely
and why its extraction from allyabase was straightforward — it had no
allyabase-internal dependencies to sever.

### Sanitization is load-bearing

`sanitizeSvg` runs untrusted SVG through `teleportation-js`'s
`removeJavaScript` before it's served as markup. Note the import is the
internal helper path (`teleportation-js/src/node-remove-javascript.js`), not
the package main — the main entry is `safeTeleportationParser`, a different
feature.

JSDOM parses even a bare `<svg>` fragment as a full document, so `sanitizeSvg`
pulls the `<svg>` element back out of the wrapper rather than returning the
whole document.

**Consequence for callers**: a published SVG cannot contain behaviour.
BizBuz's referral card wants to auto-redirect to the App Store and can't — it
uses a plain `<a href>` button instead. Any app publishing here should assume
scripts, `javascript:`/`data:` URLs, and `on*` attributes will be stripped.

### Absolute URLs and reverse proxies

The HTML page references itself (`og:image` → `/svg`, "Save Contact" →
`/vcard`), so savage must know its external scheme, host, and path prefix.
Behind path-based nginx routing, `proxy_pass` with a trailing slash strips the
`/savage` prefix before savage sees it.

`absoluteUrl` resolves this from `x-forwarded-host` / `x-forwarded-proto` /
`x-forwarded-prefix`, falling back to `req.get('host')` / `req.protocol` /
`PUBLIC_PREFIX`. Forwarded headers comma-append across proxy hops
(`"https,https"`), so `firstForwarded` keeps the first value — the original
client-facing one.

**The stock allyabase nginx config sets Host and X-Forwarded-Proto but not
X-Forwarded-Prefix.** Either add `proxy_set_header X-Forwarded-Prefix /savage;`
to the location block or set `PUBLIC_PREFIX=/savage` on the service. Without
one of them the page renders but its preview image and vCard links 404.

## Configuration

See README.md for the full table. In short: set `BDO_URL` and `PUBLIC_PREFIX`;
`LOCALHOST`/`SUBDOMAIN` are legacy fallbacks kept for older single-box setups.

Default port is **3012**, not savage's historical 3009 — 3009 is minnie's in
the shared allyabase port map, and the two collided.

## History

savage lived at `allyabase/deployment/savage/` and was deployed as part of the
Netlify gateway bundle (a `serverless-http` wrapper in `netlify/functions/`).
That bundle is retired. On extraction the Netlify artifacts (`netlify.toml`,
`netlify/functions/savage.js`, the `serverless-http` dependency) were dropped
and `BDO_URL`/`PUBLIC_PREFIX` were added so the service can be pointed at any
BDO and mounted at any prefix.

Unlike the other allyabase services, savage was never its own submodule — it
was committed directly into the allyabase repo on the `netlify-packaging`
branch.

## Related

- **BDO** — the service savage reads through to
- **BizBuz / Linkitylink** — publish SVG cards and hand out savage URLs
- **eumachia** — the sibling extracted at the same time; renders an
  *interactive* pay page, which savage deliberately cannot do since it strips
  all JavaScript
