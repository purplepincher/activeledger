# Product status: activeledger-agent on PyPI

> **Update:** the `0.2.0` parse bug documented below is fixed as of `0.2.1`,
> the current release — `pip install activeledger-agent` installs a working
> package today. This document is kept as the audit record of the original
> finding; see the main [README.md](../README.md#honesty--status) for
> current status.

This document records what the installable `activeledger-agent` package
**was** at the time of this audit, verified directly from its published
wheel, including a parse bug that prevented that release from importing.

## TL;DR

| claim / artifact | status |
|---|---|
| `activeledger.ai` landing page renders and is served by a Worker | ✅ real |
| `activeledger-agent` is a real, published PyPI package | ✅ real (v0.2.0, publisher `superinstance`) |
| The package's API is what the landing page claimed (`write_tile` / `query_ledger`) | ❌ **no** — those symbols do not exist |
| The package's real API | `ActiveLedgerAgent` class with `log_activity` / `log_investment` / `log_trade` / `ask` |
| `activeledger-agent` 0.2.0 can be imported today | ❌ **no** — `SyntaxError` on import |
| The package does useful work standalone | ⚠️ conditional — needs a PLATO server at `localhost:8847` |

## What the landing page used to say (now corrected)

The quick-start on `public/index.html` previously showed:

```python
from activeledger_agent import write_tile, query_ledger
write_tile({"type": "credit", "amount": 12500.00, "source": "Acme Corp", "category": "revenue"})
```

Neither `write_tile` nor `query_ledger` exists in the package. This was an
oversold doc — exactly the failure mode this org's doc discipline targets — so
the page has been corrected to the real API.

## What `activeledger-agent` actually exports

From the published wheel `activeledger_agent-0.2.0-py3-none-any.whl`, the module
`activeledger_agent/__init__.py` defines a single public class,
`ActiveLedgerAgent`, with these methods:

| method | what it does |
|---|---|
| `log_activity(activity, duration_minutes, energy_level="medium")` | POSTs an `activity` tile to PLATO |
| `log_investment(asset, amount, purchase_price)` | POSTs an `investment` tile |
| `log_trade(asset, action, amount, price)` | POSTs a `trade` tile (`action` = `"buy"` / `"sell"`) |
| `ask(question)` | GETs recent tiles from PLATO and returns a short string |
| `detect_emergence(events)` | wrapper over `fleet_agent.fleet_math.EmergenceDetector` |
| `check_consensus(tile_ids)` | wrapper over `fleet_agent.fleet_math.HolonomyConsensus` |

Each write constructs a tile of the form:

```python
{ "question": f"ledger:{entry_type}",
  "answer": str(data),
  "confidence": 0.9,
  "metadata": { "ledger_id": self.ledger_id, "entry_type": entry_type,
                "timestamp": time.time(), **data } }
```

and POSTs it to `http://localhost:8847/room/activeledger-ai`. So the landing
page's narrative ("writes every transaction as a PLATO tile") is accurate; only
the concrete symbol names it showed were wrong.

## The current 0.2.0 release is not importable

The published `0.2.0` wheel has a malformed `__init__.py`. Inside the
`ActiveLedgerAgent` class only `detect_emergence` and `check_consensus` are
defined; the intended `__init__`, `_write`, and the `log_*` methods were written
at the wrong indentation, so a module-level `def __init__(self, …)` appears
followed by methods whose indentation matches no outer level. Python rejects the
file before any code runs:

```
activeledger-agent: SYNTAX ERROR -> line 55:
  unindent does not match any outer indentation level
  offending: '    def _write(self, entry_type: str, data: Dict[str, Any]) -> bool:\n'
```

Observed behavior:

```
$ pip install activeledger-agent==0.2.0      # succeeds (it's a valid wheel)
$ python -c "import activeledger_agent"
SyntaxError: unindent does not match any outer indentation level (line 55)
```

So although the package is "published and installable," it is not usable as
shipped. It also depends on `fleet-agent>=0.2.0` and `requests>=2.28`, and even
with the parse bug fixed it would need a PLATO server running at
`http://localhost:8847` to do anything.

## What it would take to make the landing page's story fully true

1. Fix the `__init__.py` indentation so `import activeledger_agent` works.
2. Confirm `self.ledger_id`, `self.plato_url`, and `self.room` are actually set
   in `__init__` (they are referenced by `_write` but not defined in the parsed
   portion of the current release).
3. Provide a running PLATO server (or document that requirement on the page).

Until at least (1) and (2) land, the honest framing is: *the package exists and
its intended design is the `ActiveLedgerAgent` API, but the current release
cannot be imported.*
