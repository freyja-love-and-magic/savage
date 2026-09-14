# savage

savage takes a BDO record that has an `svg` property and serves it as a real
webpage — the SVG rendered inline, with `og:image`/`twitter:image` pointing at
a standalone SVG endpoint so the link unfurls with a preview in iMessage,
Slack, and anywhere else that reads Open Graph tags.

It is a thin read-through in front of BDO. It stores nothing, mints no
identity of its own, and holds no keys: whatever credentials arrive on the
query string (`timestamp`/`hash`/`signature`/`pubKey`) are forwarded to BDO's
own `GET /user/:uuid/bdo` unchanged, and BDO decides whether the read is
allowed. That is what makes a savage URL safe to treat as a permanent share
link — it is a pre-signed read, computed by the client, that savage never has
to interpret.

## Routes

| Route | Returns |
|---|---|
| `GET /user/:uuid/bdo` | HTML page: the SVG inline, plus a "Save Contact" button when the record has a `vcard` |
| `GET /user/:uuid/bdo/svg` | The SVG alone, `image/svg+xml` — this is what `og:image` points at |
| `GET /user/:uuid/bdo/vcard` | The record's `vcard`, `text/vcard`, as a file download |

All three take the same query string and pass it straight through to BDO.

## Sanitization

A BDO's `svg` is untrusted: anyone holding a valid signature for that uuid
wrote it. Before being served as live markup it goes through
`teleportation-js`'s `removeJavaScript`, which strips `<script>`/`<noscript>`,
`javascript:` and `data:` URLs, CSS `expression()` injections, and `on*`
handler attributes.

This is why a published SVG cannot carry behaviour — no auto-redirects, no
click handlers. Links have to be plain `<a href>` elements.

The `vcard` route needs no such pass: it is served as `text/vcard` with a
`Content-Disposition: attachment`, so browsers download it rather than render
it.

## Configuration

| Env var | Default | Purpose |
|---|---|---|
| `PORT` | `3012` | Listen port. Not savage's historical 3009 — on a shared allyabase box that belongs to minnie. |
| `BDO_URL` | see below | Base URL of the BDO to read through to. A missing trailing slash is added for you. |
| `PUBLIC_PREFIX` | `""` | Path prefix savage is mounted under externally, e.g. `/savage`. Only needed when the proxy doesn't send `X-Forwarded-Prefix`. |
| `LOCALHOST` | unset | Legacy shorthand: when set, `BDO_URL` becomes `http://127.0.0.1:3003/`. |
| `SUBDOMAIN` | `dev` | Legacy: used only in the `https://<subdomain>.bdo.allyabase.com/` fallback. |

`BDO_URL` wins if set; otherwise `LOCALHOST` selects loopback; otherwise the
`allyabase.com` subdomain fallback applies. New deployments should just set
`BDO_URL`.

## Running behind nginx

The page embeds absolute URLs back to itself, so savage has to know the path
it is reached at. With path-based routing the prefix is stripped before savage
sees the request, so pass it explicitly:

```nginx
location /savage/ {
    proxy_pass http://localhost:3012/;
    proxy_set_header X-Forwarded-Prefix /savage;
}
```

If you'd rather not touch the proxy config, set `PUBLIC_PREFIX=/savage` on the
service instead — savage checks the header first and falls back to the env
var. Get neither right and the page still renders, but its `og:image` and
"Save Contact" links will 404.

## Running

```bash
cd src/server/node
npm install
BDO_URL=http://127.0.0.1:3003/ PUBLIC_PREFIX=/savage npm start
```

## Tests

```bash
npm run test:sanitize   # the removeJavaScript pass
npm run test:vcard      # vCard route behaviour
npm run smoke           # end-to-end against a running BDO
```
