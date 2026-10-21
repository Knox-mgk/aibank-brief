# aibank — the idea, and what is left to build

A read-only brief. **No code lives here**, by design: this repo exists so the
idea and the remaining work can be read from any device without opening the
implementation. The code is at
[Knox-mgk/aibank](https://github.com/Knox-mgk/aibank) — **private**, and it will
404 for you unless you are the author. The design is public so it can be read
and discussed; the implementation is not.

**Nothing described here is deployed, and it holds no money.** The status board
in §6 lists unfinished and known-broken work openly, because that is what makes
a status board useful — none of it is reachable by anyone and there is no
running instance. §2 explains why no payment licence applies to this shape in
the first place.

**Last synced with the code:** 2026-10-21, commit `1c1c292`.
**Suite at that commit:** 250 unit tests + 451 live assertions, 0 failures,
0 resource warnings under `-W always::ResourceWarning`.

There is a sibling project: [corebank](https://github.com/Knox-mgk/corebank-brief)
is a fork of this code aimed at multi-tenant Banking-as-a-Service, where *we*
hold a pooled balance instead of the customer. That brief is the authority on
the fork; this one describes the original, self-hosted shape.

---

## 1. The idea in one paragraph

aibank is a **double-entry banking backend where humans hold the money and AI
agents spend it under limits they cannot reach.** An account holder signs in,
funds a wallet, allocates money to their agents, and sets each agent's ceilings
from a dashboard. The agent gets an API key that can do exactly one thing with
money: request a payout the server either allows, holds for human approval, or
denies. It ships both ends of that — a dashboard for the human, and an MCP
server so the agent spends through typed tools rather than hand-rolled HTTP.

Written in the Python standard library and nothing else. Zero third-party
dependencies, asserted by a test.

## 2. The shape, and why there is no licence question

**You self-host it against your own payment credentials.** aibank never holds
your money and never touches a balance you do not own, so there is no custody,
and none of Nigeria's payment licence classes apply to the software or its
author — the deployment is yours.

That was a deliberate choice between three options, decided 2026-10-09:

| | what it is | status |
|---|---|---|
| **A. Self-hosted software** | customer runs it against their own provider keys; no custody | **CHOSEN** |
| B. BaaS via a licensed partner | partner holds the float, we are the technology | became `corebank` |
| C. Own e-money licence | CBN MMO: ₦2bn capital + ₦2bn refundable escrow | out of reach, recorded to close it off |

The consequence that shaped the whole codebase: **the deploy artifact is the
product.** There is no hosted service to sign up for, so one command has to get
a stranger from clone to a running, TLS-terminated, backed-up service — and the
regulatory workload is zero.

## 3. The central invariant

> The human sets the limits. The agent operates under them and can never reach
> them.

Enforced structurally, not by convention. Three separate trust domains:

| domain | may do |
|---|---|
| **Agent** | request payouts, read own balance, add payees |
| **User** (human session) | set policy, fund wallets, approve/reject/refund, rotate keys |
| **Admin** | pricing, settlement deposits, sweeper, cross-tenant reads |

An agent key presented to a user-domain route is rejected before any handler
runs, so **no sequence of agent API calls can raise an agent's own ceiling.**

One hard lesson is baked in here. Scoping that lived only in the HTTP router
was not enough: the library underneath it was one call site away from being
reachable, and a bug of exactly that class let any account holder rewrite
another's agent limits (§6). The checks now sit on the **method**, not the
route.

## 4. The ledger

A general ledger with its invariants enforced in code:

1. **Postings within an entry sum to exactly zero.** An unbalanced entry is
   rejected, not corrected.
2. **Postings are immutable.** Corrections are new reversing entries, so
   history is append-only and auditable. Reversing twice is refused.
3. **`balance = SUM(postings)`** and **`available = balance − active holds`**.
   Balances are never stored, only derived — so they cannot drift.
4. **An idempotency key produces at most one entry, ever**, enforced by a
   UNIQUE constraint rather than an application check.

Money is integer minor units throughout; no float touches a balance.

**Money flow**

```
    admin deposit            user allocates          agent spends
settlement ──► user wallet ──────────► agent wallet ──────────► payee
                   ▲                        │
                   └──────── withdraw ──────┘
```

A user cannot credit their own wallet, and an agent cannot be funded from
nothing. Money committed to an in-flight payout is held, so a user cannot
withdraw funds their agent has already promised.

## 5. Spend policy, and the payout lifecycle

Evaluated server-side before funds are held, **deny by default**:
per-transaction ceiling; rolling daily and monthly caps; a **daily payout count
ceiling** (catches a retry loop making a thousand individually-legal payments,
which an amount cap cannot); timezone-correct daily resets (a UTC-midnight
"daily" cap resets at 01:00 in Lagos and hands the agent a second allowance);
payee allowlist with currency matching; an approval threshold; freeze; and
suspending the human immediately stops all of their agents.

**No limits means no spending.** An agent with nothing configured is refused,
not permitted — see §6, where that was a real inversion and not just a design
note.

```
request → policy check → HOLD → submit → webhook → capture
                           │                 │
                           │                 └─ FAILED → release hold
                           └─ over threshold → pending approval → human decides
                           └─ no callback ever → sweeper releases the hold
```

Funds are held before the provider is called, so a slow or failed provider can
never leave the ledger overdrawn. The distinction that matters most:

- The provider **replied and refused** → nothing was queued, fail the payout
  and release the hold immediately.
- The provider **did not reply** (timeout, connection error, unreadable 200) →
  the transfer may already be queued and the response simply lost. The payout
  stays non-terminal, the **hold is kept**, and it goes to reconciliation.
  Releasing it would let the agent respend money that is already on its way
  out, and the second spend would be backed by nothing.

Guessing is refused throughout: the sweeper flags an expired hold for
reconciliation rather than declaring it failed, because nobody knows whether
the provider moved the money.

## 6. Status board

Working and verified, re-run on 2026-10-21 at commit `1849362`:

| | state |
|---|---|
| Ledger, holds, idempotency, reversals | done |
| Policy engine incl. count ceiling + WAT-correct resets | done |
| Three trust domains over HTTP | done |
| Dashboard (single file, strict CSP, no build step) | done — **layout never judged by a human eye** |
| MCP server, 8 typed tools over stdio | done |
| TOTP second factor on the human session | done, RFC 6238 vectors pinned |
| Brute-force throttle: per-account **and** per-source-address | done |
| Deploy: Dockerfile, compose, nginx TLS, systemd units, quickstart | written; **docker/compose never built** |
| Verified ledger backups (read back, not just written) | done |
| Reconciliation tooling that refuses to self-correct | done |
| pip-installable wheel, zero deps | builds and installs clean (33 assertions) |
| Real payment provider, **write** path | **not proven** — see below |
| Inbound collections / virtual accounts | not implemented |
| Email verification, password reset | not implemented |
| Postgres / multi-process | not implemented, and it is a correctness boundary |

### The one blocker that no amount of coding removes

**The real provider API has been called for READS only.** Against a live
sandbox key, auth, JSON handling and response shapes are confirmed on bank
listing (598 Nigerian banks), balance and account resolution. But
`POST /transfers` is refused by the sandbox with *"Please enable IP Whitelisting
to access this service"* (HTTP 400) until the account whitelists the caller's
address, and no real webhook has ever been delivered. So transfer field names,
success-status strings and webhook payloads are still documentation-derived.

Every test and smoke script uses a fake provider. **The layer that actually
moves money is the least proven layer in the project**, and one whitelisted
address plus one real transfer closes it.

### Bugs this project found in itself

Kept here because the list is the strongest evidence about the codebase that
exists. Each was reproduced by a test before being fixed, and each test
remains, verified by a negative control — remove the guard and the test fails.

- **The least-configured agent had the MOST authority** — the exact inverse of
  the product's central claim, asserted in the README since the first commit.
  `create_agent` always wrote a policy row, so the "no policy → deny" branch
  was unreachable, the row held all-NULL ceilings, and every limit check skips
  NULL. An agent created with no limits could move its entire wallet in one
  unapproved payout. Proved with a 890,000 NGN approval before fixing.
- **Any account holder could rewrite another's agent limits.** New users
  defaulted to the org role `owner`, so one user could read another's agents,
  raise a ceiling and freeze the agent. Fixed at the method, not the route.
- **A refused provider call leaked the socket it arrived on.**
  `urllib.error.HTTPError` *is* the response object, so reading the error body
  without closing it leaks a file descriptor per failure — and the service
  makes the most provider calls exactly when the provider is refusing them.
  Pinned by a syntax-tree sweep, because the bug is a *missing* statement and
  the next adapter gets written by copying this one.
- **Lockout was dead code.** Failed logins were recorded inside the transaction
  that then raised, so the rollback erased every record: the brute-force
  throttle counted rows that never committed, and an attacker had unlimited
  guesses.
- **Per-account lockout did nothing against password spraying.** One common
  password tried once each against a thousand accounts tripped nothing, while
  every attempt still cost a full scrypt run — cheap for the attacker,
  expensive for the defender. Now counted per source address, checked *before*
  any key derivation.
- **The spender set our revenue.** The fee was whatever the caller put in the
  request body, and could be zero.
- **Secrets had working dev defaults**, so a deployment that forgot them had a
  publicly known admin credential and no error. Both now fail closed.
- **Migrations never ran in a transaction.** `executescript()` issues an
  implicit COMMIT that discarded the `BEGIN` before it, so a mid-migration
  failure left a half-built schema.
- **A test-isolation fault that made the suite lie.** Reloading a module
  rebuilt an exception class while other modules still referenced the dead one,
  so `assertRaises` stopped matching and the exception escaped as an error
  whose text was exactly what the test awaited — accusing the product of a bug
  it did not have.
- **A working authenticator reported as broken.** Confirming 2FA enrolment
  spends the code on screen, so signing in seconds later presented a
  correct-but-used code — reported as "invalid" *and* charged against the
  brute-force cap. Telling a user whose setup is fine that it is broken is how
  people conclude 2FA is flaky and switch it off.
- **A mistyped one-time code signed the user out**, because the page treated
  every 401 as "my session died".
- **The limits form would have erased the limits.** The API returned only
  integer minor-unit fields while the page read decimals, so every configured
  ceiling rendered as an empty input — which reads as "no limit" — and saving
  the form would have silently removed them. Found by asserting the inputs were
  pre-filled, not by asserting the endpoint returned 200.
- **`Database.close()` closed nothing** but the caller's connection: SQLite
  refuses every operation from a foreign thread, `close()` included, and a
  broad `except` swallowed it — so a dozen webhook threads leaked a dozen
  connections, each holding a descriptor for the process's life, under exactly
  the threaded server this ships.
- **Two backup tests leaked a full ledger copy each run** — 38 abandoned
  directories on a phone at 99% disk, with a permanently green suite.
- **Backup wrote a file where a directory was meant**, on exactly the path both
  shipped timers use on a first run; the next run then refused with "exists".
- **A missing request field answered with `{"error": "KeyError", "reason":
  "'user_id'"}`** — a Python class name and a quoted dict key. Under a
  self-hosted shape the HTTP surface *is* the documentation.

The pattern worth naming: almost none of these were found by reading the code.
They were found by *running* it — against a live socket, a built wheel, a real
browser engine, a real sandbox key — and several were invisible to a green test
suite.

## 7. Known limits, stated rather than discovered later

- **Single process.** The write path serialises on a process-global lock, so
  correctness assumes one process owns the database file. Two workers against
  one SQLite file would reintroduce the policy-check/hold race. This is a
  correctness boundary, not a tuning choice, and compose pins one replica.
- **Timezones are fixed offsets**, not tzdata zones, because the service must
  run on hosts that have no tzdata. Correct for zones without DST, which is all
  of WAT.
- **A refund is a book entry, not a clawback.** Nothing can pull money back out
  of a beneficiary's account.
- **The dashboard has never been opened in a real browser.** Its JS is executed
  under node against a DOM shim, which proves the code runs and renders — not
  that the layout is right. Treat the first real browser session as a test
  still owed.
- **Plain HTTP from the application itself.** It belongs behind a
  TLS-terminating proxy before it leaves localhost; a session token in clear is
  the whole account. The shipped compose service publishes no port of its own,
  so it cannot be reached around the proxy.
- **No metrics or structured logging.** A health endpoint and a reconciliation
  exit code are enough to alert on, and are not observability.
- **Not published to any package index, deliberately.** The wheel builds and
  installs clean, and that stays a local capability. `pip install aibank` does
  not work and is not going to.
- Not load-tested.

## 8. What would come next, in order

1. **Whitelist an egress address and run one real transfer plus one real
   webhook.** This is the single largest unknown in the project and the only
   item that stands between it and real money.
2. **Open the dashboard in a real browser and judge the layout.** Machine
   verification has gone as far as it can.
3. **Build the container images once** on a machine that has a container
   runtime. They are written and reviewed but unexecuted.
4. **Account recovery** — email verification and password reset. Today an
   operator with CLI access is the only recovery path.
5. **Inbound money** — collections and virtual accounts, so funding is not
   admin-gated. (The fork, corebank, has probed this: a permanent per-customer
   account number requires identity verification first, because the account
   number lands on a licensed institution's core banking system and that
   institution owes the identity record.)
6. **Postgres behind the data layer**, which is what multi-process actually
   requires.

---

Licence: MIT. Copyright Odusipe Kayode Oluwaseun.
