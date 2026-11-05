# aibank — the idea, and what is left to build

A read-only brief. **No code lives here**, by design: this repo exists so the
idea and the remaining work can be read from any device without opening the
implementation. The code is at
[Knox-mgk/aibank](https://github.com/Knox-mgk/aibank) — **private**, and it will
404 for you unless you are the author. The design is public so it can be read
and discussed; the implementation is not.

**Nothing described here holds money, and the one deployment that exists
cannot move any.** A single instance runs on the author's own VPS, bound to
loopback and reachable only through an SSH forward — no public route, no
customers, no provider credential. Its `/health` reports
`"payouts": "none"`, which is the activation gate refusing to load a real
provider (§6). The status board in §6 lists unfinished and known-broken work
openly, because that is what makes a status board useful. §2 explains why no
payment licence applies to this shape in the first place.

**Last synced with the code:** 2026-11-05, commit `82cc2e2`.
**Suite at that commit:** 475 unit tests, 0 failures, re-run on 2026-11-05 on
the author's phone (116s) — under `-W always::ResourceWarning`, 0 warnings.
Plus 469 live assertions against real servers, all re-run the same day: 74
HTTP, 100 MCP, 139 dashboard, 65 MFA, 91 DOM, and 33 packaging assertions.

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
| **Agent** | request payouts, read own balance, add payees, create and fund a subordinate strictly within its own grant |
| **User** (human session) | set policy, fund wallets, grant and revoke authority, approve/reject/refund, rotate keys |
| **Admin** | pricing, settlement deposits, sweeper, cross-tenant reads |

An agent key presented to a user-domain route is rejected before any handler
runs, so **no sequence of agent API calls can raise an agent's own ceiling.**

One hard lesson is baked in here. Scoping that lived only in the HTTP router
was not enough: the library underneath it was one call site away from being
reachable, and a bug of exactly that class let any account holder rewrite
another's agent limits (§6). The checks now sit on the **method**, not the
route.

### Delegation: an agent that hires an agent

An agent holding `financial_delegation` may create a subordinate, fund it out
of its own allocation, and delegate authority to it — **strictly within its own
grant**, with depth capped. This is what makes the model an org chart rather
than a flat list of keys, and it is the part most likely to be got wrong, so
the rules are the narrow ones:

- A delegation requesting more than the delegator itself holds is refused.
  Authority can only ever narrow on the way down.
- An authority may not be granted an expiry that **outlives its parent's**, and
  expiry is checked for the agent *and every ancestor* at evaluation time, with
  `authority_expired` reported distinguishably from `no_authority`.
- Revoking a grant **cascades to money already allocated under it**. A
  subordinate's next spend is refused, not merely its next grant.
- Authority is evaluated **inside the same write transaction that places the
  hold**, against amount + fee, on every payout and again when a human
  approves one. It was previously recorded correctly and consulted nowhere on
  the payout path — a delegated subtree cap was invisible and revocation
  stopped nothing (§6).
- Policies and authorities are mirrored in both directions, so an owner setting
  a ceiling through the dashboard **is** an owner granting authority, rather
  than leaving the agent silently with none.

The agent tool set is defined by what it is willing to hand out, not by what it
remembers to withhold: owner-only powers (granting itself authority, reading
another owner's agents) are absent from the MCP registry entirely, so no code
path can expose them later by omission.

## 4. The ledger

A general ledger with its invariants enforced **in the database**, not merely in
the module that writes it:

1. **Postings within an entry sum to exactly zero.** An unbalanced entry is
   rejected, not corrected.
2. **Postings are immutable.** Corrections are new reversing entries, so
   history is append-only and auditable. Reversing twice is refused.
3. **`balance = SUM(postings)`** and **`available = balance − active holds`**.
   Balances are never stored, only derived — so they cannot drift.
4. **An idempotency key produces at most one entry, ever**, enforced by a
   UNIQUE constraint rather than an application check.

Money is integer minor units throughout; no float touches a balance.

### Why those invariants are triggers and not assertions

All three of immutability, the no-zero-leg rule and double entry itself used to
live inside `ledger.py`. That module kept them perfectly — and **nothing else
had to.** Any other statement on the same connection (a repair script, a future
endpoint, `sqlite3` on the live file) could rewrite an amount, drop a leg, or
leave a one-sided entry, and the books would simply read differently afterwards
with no trace of the change. An invariant enforced only by the code that
happens to write it is an invariant that ends at the next author, and for a
ledger that is the whole of the asset: **the books are evidence only if they
cannot be edited after the fact.**

Schema v6 moved all three below the application. UPDATE and DELETE on
`postings` abort unconditionally, DELETE on `entries` aborts, a zero-amount leg
aborts. Double entry needed more care, because SQLite has no deferred triggers
and an entry's legs necessarily arrive one at a time — a per-INSERT sum check
would reject every two-leg entry on its first leg. So an entry is **sealed**
when its legs are complete: sealing an entry whose legs do not net to zero
aborts, appending a leg to a sealed entry aborts, and unsealing aborts.

The subtlety worth keeping: the summary columns the seal check reads are
maintained by a trigger that **recomputes** them from postings rather than
incrementing, and a second trigger refuses any value that disagrees with the
postings it claims to summarise. Without that, setting `legs_sum = 0` by hand
would let a one-legged entry seal — money from nothing.

The migration backfills and seals balanced history and deliberately leaves a
pre-existing imbalance **unsealed and visible** rather than blessing it;
`backup()` counts those, so an archive carrying an entry the database never
verified fails verification now instead of at restore.

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

### Three defects the lifecycle work found

All three were found by writing the ten mandated lifecycle scenarios, and all
three are the same shape — an identity or a comparison that was *nearly* right:

- **Event dedupe keyed on `(event, data.id)` and ignored the provider status**,
  but Flutterwave reuses one transfer id across every callback for that
  transfer. A genuine `FAILED → SUCCESSFUL` transition was therefore answered
  "duplicate" and dropped before any state machine saw it. Status is now part
  of event identity: replays still dedupe, transitions reach the state machine,
  which refuses to re-debit and records the disagreement.
- **An idempotency key matched on reference alone**, so retrying the same key
  with a different amount or payee returned the earlier payout and reported
  success for a payment the caller never asked for. A reused key whose request
  differs is now refused, naming the differing field.
- **A stale success reaching a refunded or rejected payout** is recorded in
  `payout_state_conflicts` for an operator, not applied and not dropped:
  applying it invents a debit nobody authorised, dropping it hides money that
  may really have left.

Holds also stopped lying in the same pass: `capture_hold` raises instead of
silently matching no rows when the hold is not active, and `release_hold`
refuses to release a captured hold.

## 5a. A second payout rail: CoreBank, trusted no further than Flutterwave

The sibling project `corebank` holds a ledger that already has money in it, so
aibank can settle a payout through it instead of through a card/transfer
provider. The adapter satisfies the same surface `FlutterwaveClient` does and is
selected by the same three-variable activation gate, so a deployment that does
not set `AIBANK_PAYOUT_PROVIDER` is unchanged in every respect.

**The decision the whole thing turns on: a 5xx is not a refusal.**

`_submit` branches on one flag to decide whether the agent's hold is *released*
or *left held*. Get it wrong in one direction and the agent respends money that
has already left; wrong in the other and funds are stranded behind a hold for a
payment that was cleanly refused. Copying Flutterwave's mapping was the
expensive mistake on offer, because Flutterwave treats every HTTP reply as
definite — and for Flutterwave that is correct: it answers a malformed or
unaffordable transfer with a 4xx and queues nothing.

CoreBank is not shaped like that. Its write path is hold + post + capture inside
**one SQLite transaction, with the HTTP response written afterwards.** A 500 can
therefore come from the catch-all *after* that transaction committed: the
posting exists, the money has moved, and the body says `{"error": "internal"}`.
Treating that as a refusal would release aibank's hold against a debit CoreBank
had already made. So:

| provider says | classification | hold |
|---|---|---|
| 4xx | definite — CoreBank parsed it, decided, and rolled back (its own hold included) | released |
| 5xx | indeterminate — the posting may have committed before the reply failed | kept |
| timeout, connection error, unreadable body, 2xx with no entry id | indeterminate | kept |

Two different mechanisms do two different jobs here and neither substitutes for
the other: the **classification** protects aibank's hold, the **idempotency
key** protects CoreBank's ledger. An adapter with the right key and the wrong
classification double-spends; with the right classification and the wrong key it
pays twice on retry.

Four consequences worth stating:

- **The payout reference *is* the idempotency key** — not a value minted per
  attempt, which would make a retry carry a different key and have CoreBank pay
  twice. It is already derived from the payout row and the caller's own key, so
  it is stable across retries of one payout and unique across different ones.
- **A terminal submit reply settles immediately.** A synchronous provider that
  answers "settled" must not be left `submitted`, or the hold lives until the
  sweeper, the agent's funds are immobilised for the TTL, and an operator gets a
  queue of payouts that already worked. The branch keys on **what the provider
  said**, not on which adapter is loaded.
- **An agent key, not an admin key.** The funding wallet is fixed by the
  credential aibank holds and cannot be redirected by anything in the request.
  The admin key would have been easier and untenanted — the body would then have
  had to name the agent, and one leaked key would drain any agent in any
  organisation.
- **The adapter sends CoreBank nothing resembling permission** — no policy
  verdict, no ceilings, no agent id, no balance, asserted absent by a test.
  CoreBank re-derives authority, limits, fee, currency and available balance on
  its own side and refuses requests aibank has already allowed. The test funds
  the fake CoreBank with *less* than aibank believes it can spend, and then — in
  a second test — funds it generously and watches the same payout succeed, or
  the refusal could have come from aibank's own policy and the claim would be
  untested.

Reads refuse rather than invent: CoreBank has no by-reference entry lookup, no
bank directory and no name resolution, so the adapter **raises** on all four
instead of returning a plausible empty result — `reconcile` treats the
provider's answer as authoritative about real money, and an adapter reporting
"no such transfer" for a payout CoreBank had settled would drive a
reconciliation to the wrong conclusion.

**Adding the second provider exposed a latent bug in the activation gate.**
`activation_status` asked for `FLW_SECRET_KEY` unconditionally. With one
provider that reads as a spelling choice; with two it is a hole — a deployment
selecting `corebank` while holding a Flutterwave key would have been activated
**by that key**, a credential for one partner opening a money path to another,
and a deployment holding only a CoreBank key could never activate at all. The
credential is now looked up per provider.

## 6. Status board

Working and verified, re-run on 2026-11-05 at commit `9ead69d`:

| | state |
|---|---|
| Ledger, holds, idempotency, reversals | done |
| Ledger immutability enforced by **database triggers**, not by the writing module | done (schema v6) |
| Policy engine incl. count ceiling + WAT-correct resets | done |
| Three trust domains over HTTP | done |
| Authority, delegation, expiry and revoke-cascade | done — enforced inside the hold transaction |
| Delegation + reporting exposed over HTTP **and** MCP | done |
| Payout lifecycle: 28 scenarios, no second debit, no silent hold | done |
| Second payout rail (CoreBank adapter) with its own indeterminacy mapping | done against a fake CoreBank **and** a real `corebank serve` process |
| Payout activation gate: three independent assertions, per-provider credential | done |
| Throttles on payouts, key auth, agent requests, transfers, subordinate creation | done — written in their own transactions |
| Rotatable, database-backed admin keys with overlap and immediate revoke | done |
| Dashboard (single file, strict CSP with a per-response nonce, no build step) | done — **layout never judged by a human eye** |
| MCP server, typed tools over stdio | done |
| TOTP second factor on the human session | done, RFC 6238 vectors pinned |
| Brute-force throttle: per-account **and** per-source-address | done |
| Deploy: Dockerfile, compose, nginx TLS, systemd units, quickstart | written; **docker/compose never built** |
| Backups: written, **read back, and restore-rehearsed** weekly | done — see below |
| `connect()` closes its handle when the file turns out to be unreadable | done 2026-11-05 — the restore drill was the leaking caller |
| Reconciliation tooling that refuses to self-correct | done |
| pip-installable wheel, zero deps | builds and installs clean (33 assertions) |
| One deployment, loopback-only, no provider credential | running on the author's VPS at this brief's commit, schema 8, `payouts: none` |
| Real payment provider, **write** path | **not proven** — see below |
| Inbound collections / virtual accounts | not implemented (the fork built it; porting back is optional) |
| Email verification, password reset | not implemented |
| Postgres / multi-process | not implemented, and it is a correctness boundary |

### The backup half that was theatre

aibank wrote four verified copies a day and **never read one back.** The copies
were checked *as they were written*, which cannot see the failures that arrive
afterwards — a truncated write from a full disk, a bad block, silent
corruption — and there was no drill at all.

Worse, the verifier opened the archive **in place**, and `connect()` sets
`journal_mode = WAL`, which is a write: it modified the very files whose
integrity was in question, left `-wal`/`-shm` sidecars beside them, and would
fail outright on read-only media. It now copies the archive out, carries any
sidecars with it, and opens the copy — with a test asserting the archive's
sha256 and mtime are unchanged afterwards and that no sidecar appeared.

The second finding is the one that generalises. Both products required
`schema_version == SCHEMA_VERSION` to call an archive restorable. **The day a
migration lands, every retained copy is one version behind**, so 27 of 28 read
as RED and the real zero-decision recovery window collapses to the hours since
the schema bump — which trains an operator to ignore a failing drill. Verdicts
are now three: `ok` restores with nobody in the loop, `MIGR` restores after a
migration that was **rehearsed** on the scratch copy, `BAD` would not come back.
The rehearsal re-counts entries, postings and the trial balance afterwards,
because a migration that rebuilds a table to change a CHECK constraint can drop
rows on the DROP/RENAME, and "the migration ran" is not the claim worth making.

`restore-verify` exits 0 restorable, 1 unusable, **2 nothing found** — the 2
matters, because an empty directory used to be indistinguishable from a passing
drill, which is a timer that reports success forever on no backups.

**Note on the live instance:** it now runs at this brief's commit, migrated to
schema 8 on 2026-11-05 — backup taken and integrity-checked first, v7 → v8
applied, trial balance 0 after. Before that it ran an eight-commit-old copy
whose code and database agreed at schema 7: stale, not half-migrated. It
refused to start against the older database rather than guessing, which is
exactly what the version check exists for.

### The one blocker that no amount of coding removes

**The real provider API has been called for READS only.** Against a live
sandbox key, auth, JSON handling and response shapes are confirmed on bank
listing (598 Nigerian banks), balance and account resolution. But
`POST /transfers` is still refused with HTTP 400, and the refusal **moved** on
2026-10-23: it used to be *"Please enable IP Whitelisting to access this
service"*, which was a network problem, and once a fixed egress address was
whitelisted it became *"merchant is not enabled to make transfers"* — an
**account entitlement** the provider grants after compliance review. No code
and no infrastructure changes that. No real webhook has ever been delivered
either, so transfer field names, success-status strings and webhook payloads
are still documentation-derived.

Every test and smoke script uses a fake provider. **The layer that actually
moves money is the least proven layer in the project**, and the only thing that
closes it is the provider granting the Transfers product. The CoreBank rail
(§5a) is a genuine second path and was driven end to end against a real
CoreBank process — but both sides of it are the author's own ledgers, so it
proves the protocol, not the banking rail.

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

- **A concurrency assertion that measured the machine, not the product.** The
  duplicate-webhook test counted threads whose reply said `settled`. A
  duplicate echoes the claim row's stored result, which is NULL while the
  winner is still mid-settlement — so the count measured the thread *schedule*.
  Identical code passed on Python 3.14 and failed on 3.12. Nothing ever settled
  twice; the invariants (one event row, one claim, two ledger entries, trial
  balance zero, balance debited once) are what the test asserts now.
- **The README told strangers to install from a repo that 404s.** After the
  code repo went private the install line still read
  `pip install git+https://…`, so the first command anyone ran failed with an
  error that reads as a broken project rather than a closed repo. Documentation
  that contradicts the deployment is the same class of defect as code that
  contradicts the docs.
- **Authority was recorded correctly and consulted nowhere that mattered.**
  `request_payout` asked the policy engine only, so a delegated subtree cap was
  invisible and `revoke_authority` stopped nothing — the delegation model was
  bookkeeping, not enforcement. Authority is now evaluated inside the same write
  transaction that places the hold.
- **The activation gate could be opened by a credential alone.** The only thing
  between the deployment and real money leaving a real bank account was a blank
  `FLW_SECRET_KEY`, on a host that owns the provider-whitelisted egress IP: one
  env-var write plus a restart, no code change and no recorded decision. Depth
  of control was one, and its single element was the kind of value someone
  pastes in while fixing something unrelated. Activation now needs three
  independent assertions, one of which is a *sentence* no boolean-flipping
  config template can produce.
- **The dev-secrets escape hatch was one assertion** whose value is exactly what
  a template defaults to, or what an operator types at 02:00 chasing an
  unrelated "X is not set". Setting it on a production host silently swapped the
  pepper guarding every API-key hash and every sealed TOTP secret for a constant
  published in the repository, and **nothing in the running service looked
  different.** Now two assertions plus an outright refusal when the deployment
  looks like production at all — and the refusal is the half that matters: env
  vars are things an operator can type, deployment markers are things the
  deployment asserts about itself.
- **The brute-force counter could never be spent.** Extending the login counter
  to cover payouts would have been the obvious move and was wrong: it is written
  by the same transaction as the thing it counts, and a refused payout **rolls
  back**, so the attacker's budget rolls back with it and the audit trail shows
  nothing happened. Throttle events are now written in their own transactions at
  the request edge.
- **The admin key was one value in an env file.** The comparison was correct;
  the lifecycle was the finding. Changing it meant editing a root-owned file and
  restarting, so every client had to cut over in the same instant — which means
  it never changed. No revoke for a suspected leak, no expiry for a key issued
  for one afternoon, and exactly one key, so no admin action could be attributed
  to anyone.
- **The backup verifier wrote to the archives it was verifying**, and a schema
  bump made 27 of 28 retained copies report RED (above).
- **A merge silently deleted a security fix, and the suite stayed green.**
  Found 2026-11-05 on *pushed* `main`. A merge resolution in the fork reordered
  `require_agent_authority` so the staff bypass was checked *before* the caller's
  asserted identity — and that flag is true for the unauthenticated local
  handle, so a supplied `user_id` was discarded entirely and one account holder
  could again raise another's agent ceiling or freeze their agent through the
  library. Nothing re-ran the one test file that attacks through the bare handle
  after the merge, so `git log` showed a fix that the code no longer contained.
  The lesson is narrower than "run your tests": **a merge is a code change that
  no review diff shows you**, because the conflict resolution *is* the edit, and
  the file reads plausibly either way.
- **A failed database open leaked its handle, and the warning blamed the
  standard library.** Found 2026-11-05. `connect()` opens the file and then
  issues four PRAGMAs, which are the first *reads* — so a corrupt or truncated
  database raises after the handle exists and before the caller receives it.
  Nobody can close a connection they were never given, so the descriptor and
  its WAL sidecar survived until the garbage collector ran, at which point
  `-W always::ResourceWarning` reported "unclosed database" against whatever
  line happened to trigger collection. The restore drill is the caller that
  matters: opening deliberately-unreadable archives is its job. Fixed in both
  repos, with a negative control proving the leak detector can fail.
- **The guard against the *older* leak was scoped to the wrong half of the
  repo, and that is the more useful finding.** An AST sweep has pinned "never
  read an `HTTPError` body without closing it" since the original socket leak
  was fixed — but it scanned the package directory only. Eighteen handlers in
  `tests/` and `scripts/` were still leaking, and those leak *harder*: a smoke
  script's request helper runs on every refusal the script asserts, and the
  throttle tests exist to provoke 401s and 429s in volume. A full suite run
  emitted 30 resource warnings from one test helper alone; it is 0 now. Test
  and script helpers are also the code a human copies when writing the next
  adapter, so excluding them guarded the half that mattered least.
  The generalisable pair: **a resource warning names where collection
  happened, not where the leak is** — so "that warning is from a dependency"
  is the wrong conclusion to reach cheaply — and **a guard is only as good as
  its scope, which nothing re-checks once it is green.**

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
  so it cannot be reached around the proxy. The live instance is reached only
  over an SSH forward, which is why it needs no proxy today — and also why
  nothing has ever exercised the TLS path.
- **No metrics or structured logging.** A health endpoint and a reconciliation
  exit code are enough to alert on, and are not observability.
- **Never going to a package index, by decision.** The wheel builds and
  installs clean, and that is kept purely as a self-check that the install
  path works. This is private software, not a library: the only install is
  `pip install git+ssh://git@github.com/Knox-mgk/aibank` with author access.
- **The live instance now runs this brief's commit**, migrated v7 → v8 on
  2026-11-05. It had been eight commits behind — code and database agreeing at
  schema 7, so stale rather than half-migrated, and it refused to start against
  the old database instead of guessing. It is loopback-only with no provider
  credential, so the write path below is still unproven, but the recent work
  has now run somewhere other than a phone.
- Not load-tested.

## 8. What would come next, in order

1. **Get the provider's Transfers entitlement granted, then run one real
   transfer and one real webhook.** This is the single largest unknown in the
   project and the only item between it and real money. It is not an
   engineering task: the network half was solved on 2026-10-23 and what remains
   is a compliance request to the provider.
2. **Open the dashboard in a real browser and judge the layout.** Machine
   verification has gone as far as it can.
3. **Migrate the live instance from v7 to v8** and re-run the restore drill
   against it, so the deployment and the code agree.
4. **Build the container images once** on a machine that has a container
   runtime. They are written and reviewed but unexecuted.
5. **Account recovery** — email verification and password reset. Today an
   operator with CLI access is the only recovery path.
6. **Inbound money** — collections and virtual accounts, so funding is not
   admin-gated. (The fork, corebank, has now *built* this: a permanent
   per-customer account number requires identity verification first, because
   the account number lands on a licensed institution's core banking system and
   that institution owes the identity record. Porting it back here is optional,
   not required.)
7. **Postgres behind the data layer**, which is what multi-process actually
   requires.

---

Licence: MIT. Copyright Odusipe Kayode Oluwaseun.
