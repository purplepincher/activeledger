# activeledger

Landing page for **activeledger.ai** — a Cloudflare Worker that serves a single
static HTML page introducing *activeledger-agent*, a Python package for logging
financial activity to a PLATO memory layer as "tiles."

This repo is part of the PurplePincher / SuperInstance domain family. Its
siblings — [`activelog`](https://github.com/purplepincher/activelog) and
[`luciddreamer`](https://github.com/purplepincher/luciddreamer) — share the
exact same Worker + design-system skeleton, differing only in the page they
serve and the Worker name.

> **Read this first:** this repository is a **landing page**, not the package.
> The package is `activeledger-agent` on PyPI. Its real, verified API is
> documented below and in [docs/product-status.md](docs/product-status.md) — and
> it does **not** match what older copies of the landing page showed. The page's
> quick-start has been corrected; the details are in the
> [Honesty / status](#honesty--status) section.

---

## What is actually in this repo

A Cloudflare Worker whose entire job is to serve a static asset directory. There
is no application logic, no server-side processing, and no build step beyond
what Wrangler does natively.

```
activeledger/
├── src/index.ts        # 13-line request handler: env.ASSETS.fetch(request) + 404/500
├── public/index.html   # the page that gets served (activeledger-agent explainer)
├── family/             # shared PurplePincher design-system skeleton (see below)
│   ├── README.md          # operator's manual for the design system
│   ├── tokens.css         # :root palette + type scale (inlined into the page)
│   ├── base.css           # reset + component classes (.eyebrow, .chain, .ledger, …)
│   ├── provenance-panel.css
│   └── provenance-panel.html
└── wrangler.jsonc      # name="activeledger", assets dir ./public, binding ASSETS
```

`src/index.ts` is byte-for-byte identical to the handler in `activelog` and
`luciddreamer`:

```ts
export default {
  async fetch(request: Request, env: any): Promise<Response> {
    try {
      const response = await env.ASSETS.fetch(request);
      if (!response) return new Response("Not found", { status: 404 });
      return response;
    } catch (e) {
      return new Response(`Error: ${e instanceof Error ? e.message : 'Unknown error'}`, { status: 500 });
    }
  },
};
```

Every request is handed to the Workers [static assets](https://developers.cloudflare.com/workers/static-assets/)
binding (`env.ASSETS`). If no file matches, the Worker returns `404`; on an
exception it returns `500`. That is the whole runtime behavior.

### The `family/` design system

`family/` is the shared PurplePincher design skeleton, documented in
[`family/README.md`](family/README.md). Its architecture is a deliberate
**inline-at-build-time, never-fetch-at-runtime** rule: `tokens.css` and
`base.css` are copied into the page's `<style>` block (which is why
`public/index.html` is self-contained and makes no runtime CSS requests).

Two notes a reader should be aware of:

- The page does **not** perform the per-site accent swap described in
  `family/README.md`. It leaves `--claw` at the default aubergine, so this site
  currently renders in the reference palette rather than a dedicated
  activeledger accent.
- The `provenance-panel.*` honesty component ships in `family/` but is **not**
  used by this page. The page has its own inline "Honesty note" instead.

---

## Run it

No `package.json`, no dependencies to install — Wrangler talks to the TypeScript
entry directly. Wrangler 4.x is what this was authored against.

```bash
# local dev server (serves public/index.html)
wrangler dev

# validate config + bundle without deploying
wrangler deploy --dry-run

# deploy to Cloudflare (requires you to be authenticated to the activeledger account)
wrangler deploy
```

`wrangler deploy --dry-run` was verified against the sibling repos (identical
config shape): it reads the `./public` assets directory, bundles the Worker, and
reports the `env.ASSETS` binding.

---

## The real `activeledger-agent` API (verified from the published wheel)

`activeledger-agent` (PyPI, published by `superinstance`; project
`github.com/SuperInstance/activeledger-agent`) is at version `0.2.1`. Its
summary line reads:

> activeledger domain agent for PLATO fleet

Inspecting the published wheel (`activeledger_agent/__init__.py`) shows it
exports a single class, `ActiveLedgerAgent`, whose methods are:

- `log_activity(activity, duration_minutes, energy_level="medium")`
- `log_investment(asset, amount, purchase_price)`
- `log_trade(asset, action, amount, price)` — `action` is `"buy"` or `"sell"`
- `ask(question)` — reads recent tiles from a PLATO server and returns a string
- `detect_emergence(events)`, `check_consensus(tile_ids)` — thin wrappers over
  `fleet_agent.fleet_math`

Each `log_*` method POSTs a "tile" to a PLATO room at
`http://localhost:8847/room/activeledger-ai`. The intended usage is:

```python
from activeledger_agent import ActiveLedgerAgent
agent = ActiveLedgerAgent()
agent.log_investment(asset="ESPP", amount=10, purchase_price=42.50)
agent.ask("recent investments")
```

---

## Honesty / status

Using the family's honesty-marker convention:

- ✅ **real today** — the static landing page renders; the Worker serves it via
  `env.ASSETS`; the `family/` design-system assets are present and inlined.
  `activeledger-agent` `0.2.1` installs and imports cleanly — a `0.2.0` release
  shipped with a malformed `__init__.py` (a module-level `def __init__` plus
  mis-indented methods) that raised `SyntaxError` on import; fixed in `0.2.1`.
- ⚠️ **real but conditional** — `activeledger-agent` is a **real, published PyPI
  package** and its intended API is the `ActiveLedgerAgent` class above. It only
  does anything useful with a **PLATO server** reachable at
  `http://localhost:8847` (it `import`s `fleet_agent` and `requests`).
- 🔮 **later phase / not done** — no tests, no CI, no `package.json` in this
  repo; the page does not apply the per-site accent swap or the provenance
  panel.

### What changed on the landing page

The landing page's quick-start previously showed an API that **does not exist**
in the package:

```python
# OLD — these symbols do not exist in activeledger-agent
from activeledger_agent import write_tile, query_ledger
write_tile({"type": "credit", "amount": 12500.00, "source": "Acme Corp", "category": "revenue"})
```

There is no `write_tile` and no `query_ledger` in `activeledger_agent`. The page
now shows the real `ActiveLedgerAgent` API instead. The surrounding narrative
("writes every transaction as a PLATO tile") remains accurate — the real
`_write` does post a tile to PLATO.

The page's honesty note previously read "*you can install today … the code and
the tile format are real*," which omits that the published 0.2.0 wheel raises
`SyntaxError` on import (see ❌ above). The page now states the import failure
explicitly.

---

## Related

- `docs/product-status.md` — the verified reality of `activeledger-agent` on
  PyPI, including the import-breaking parse bug.
- [`family/README.md`](family/README.md) — the design-system operator's manual.
- Sibling landing repos: [`activelog`](https://github.com/purplepincher/activelog),
  [`luciddreamer`](https://github.com/purplepincher/luciddreamer).
