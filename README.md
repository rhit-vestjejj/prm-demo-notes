# prm-demo-notes

A tiny notes API whose pull requests are gated by **[PRediction Market](https://github.com/aadev151/PRediction-market)**:
AI agents trade on whether each PR is correct, and the closing price decides what merges.

> This repo is a deliberately small **demo target** built for Indy Init 2026. The service is real and tested, but its whole
> purpose is to show what the factory does to a PR.

## What happens when you open a PR

1. A webhook tells the factory about the PR. It posts a `pr-market` status: **pending, "market open…"**.
2. It reads [`.prm.toml`](.prm.toml) from the **base** branch (so a PR can't change its own gate) and fetches the diff.
3. **Four AI traders** each see *different* evidence and give a probability that the PR holds up after merge:

   | Trader | Sees |
   |---|---|
   | `diff_reader` | the diff only |
   | `coverage` | the diff, the test changes, and which changed files have no test change |
   | `security` | the diff, hits on sensitive paths (`**/auth*`, …) and risk signals (a removed auth check, `eval`, SQL built from strings…) |
   | `skeptic` | the diff and the PR description, prompted to argue why it fails |

4. They trade through an LMSR market maker, starting at 0.50. Each trader can spend at most 30 credits per market,
   which is not enough for any single trader to push the price to 0.90 alone.
5. The closing price becomes the `pr-market` status:

   | Price | Status |
   |---|---|
   | ≥ 0.90 | ✅ `success`: "auto-merge OK (p=0.94)" |
   | < 0.60 | ❌ `failure`: "blocked (p=0.21)" |
   | in between | the traders see each other's reasoning and trade again (up to 2 escalation rounds); still undecided → ❌ "no consensus" |

6. **Settlement is real, not claimed.** A merged PR settles **NO** if, within 14 days, it is reverted or a later fix commit changes
   the lines it added; otherwise **YES**. Traders who were right gain bankroll, so their future trades move the price more.
   Rejected PRs are refunded. Any error fails closed: the market is voided and the PR is blocked, never merged.

The agents only produce numbers. Deterministic code owns trading, gating, settlement and every state change.

## Try it: the two demo branches

| Branch | Change | What the market should do |
|---|---|---|
| `demo/remove-admin-check` | drops the admin dependency from `DELETE /notes/{id}`, so anyone can delete notes | the security trader sees a removed auth check, the price crashes, `pr-market` goes **red** |
| `demo/docstrings` | adds two docstrings, no behavior change | the price climbs past 0.90, `pr-market` goes **green** |

Open a PR from either branch into `main` and watch the `pr-market` check. Its link opens the market page: price chart, every
trade, the state timeline and (after close) each trader's rationale.

Note that the author's own tests in `tests/` **pass on both branches**: they don't cover unauthenticated deletes. That's the
point: the gate has to catch what the author's tests miss.

## The service

`notes_app.create_app(admin_token, clock=None)` builds a FastAPI app with an in-memory store:

- `GET /health`
- `POST /notes`: `title` 1–100 chars, `body` ≤ 2000, unknown fields rejected → `201`; 10 POSTs/min per client → `429`
- `GET /notes/{id}` → `404` if missing; `GET /notes?limit=` (1–100)
- `DELETE /notes/{id}` and `GET /admin/stats`: require `Authorization: Bearer <admin token>` → `401` otherwise

The full contract is in [`docs/SPEC.md`](docs/SPEC.md).

```bash
python -m venv .venv && . .venv/bin/activate
pip install -r requirements.txt
python -m pytest
```

## Plugging PRediction Market into your own repo

1. Add a `.prm.toml` like [this one](.prm.toml) to your default branch.
2. Register the repo with the factory (`prm repo add owner/name`) and add a `pull_request` webhook pointing at the factory.
3. Optionally require the `pr-market` status in branch protection.
