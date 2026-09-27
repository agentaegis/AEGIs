# AEGIs

An autonomous trading agent for the **Robinhood Chain** (EVM L2, `chainId 4663`). It
finds tokens that are not scams, sizes a position it can afford to be wrong about,
executes the swap, manages the exit, and repeats — indefinitely, without a human in
the loop.

```
python main.py --config config.yaml
```

It ships in **paper mode**. Paper mode is not a flag on the live engine; it is a
different broker object, and the wallet on that path holds no key at all, so the
simulator cannot reach a signing code path by accident. Between paper and live there
is **dry run**, which is the live path in full — real balances, real quotes, real
`eth_estimateGas`, real nonce — stopping at the signature and reporting the exact
transaction it would have broadcast. Going live is an edit to a config file that
persists, which is why there are `--paper` and `--dry-run` flags and deliberately no
`--live` one: the two flags that exist can only ever remove the ability to spend.

- 10,600 lines of package plus 4,700 of tests, no framework, five pinned dependencies
- 561 unit tests, ~9 s, no network and no chain required
- Peak RSS 55 MB for a full startup and connectivity check, measured

---

## Quick start

```bash
cd ~/AEGIs/aegis
python3 -m venv ../.venv
../.venv/bin/pip install -r requirements.txt

cp config.example.yaml config.yaml     # every value in it is already the default
../.venv/bin/python main.py --check    # connect, verify chainId, print the report
../.venv/bin/python main.py --config config.yaml
```

`--check` is the honest first command: it validates the config, dials the RPC,
confirms the chain is 4663, probes which RPC methods the endpoint actually
implements, prints the wallet and balances, lists the enabled strategies, and shows
whether the kill switch is engaged. It never sends a transaction. If `--check` is
happy, the agent will start.

Verify the build before trusting it:

```bash
../.venv/bin/python -m unittest discover -s tests      # 561 tests
```

---

## What it does, in the order it does it

One **tick** is one complete pass. The default cadence is 5 s. A tick never raises;
a failed step is recorded in the tick report and the pass continues.

1. **Discovery.** New pools since the last tick become screening *candidates*, never
   positions.
2. **Pricing.** Reserves for the watchlist and everything held, as JSON-RPC batches
   rather than one call per pool — 25 tokens cost one or two HTTP posts, which on a
   2 req/s endpoint is the difference between a 5 s tick and a 30 s one.
3. **Snapshot.** One consistent view of holdings, equity, and the day's PnL. Every
   later decision reads this snapshot, not the chain, so the tick cannot act on two
   different versions of reality.
4. **Exits — before entries, and they run even when entries are halted.** A daily
   loss limit that also blocked selling would trap the agent in the position that
   breached it.
5. **Entry gates.** The kill switch and the daily loss limit. If either is engaged the
   tick ends here.
6. **Screening**, budgeted to `safety_scan_per_tick: 3` so a long queue cannot stall
   the loop — a round trip is several RPC calls and the queue after a busy discovery
   pass can be hundreds of tokens deep.
7. **Strategies → sizing → execution.**

`--once` runs exactly one tick and exits (useful from cron); `--ticks N` stops after
N. Both are the same code path the daemon runs.

---

## Pillar 1 — finding tokens that are not scams

Screening is the part that decides whether the other three pillars matter. Nothing
reaches a strategy until it has passed, and the verdict is cached with an expiry
rather than kept forever.

### The launchpad gate — where a token came from, checked before anything else

The first question is not whether a token is safe but whether it is *eligible*. With
`screening.launchpads` set, a token that did not come out of one of the named
launchpads is refused before its symbol is read — no score, no liquidity check, no
simulation, and nothing later in the analyser can overrule it. It is the only rule in
the agent that will refuse a token no measurement objects to.

Provenance is one `eth_call`. Both Pons factories keep an on-chain registry of what
they launched and answer `getLaunchedToken(address)`; a token they launched comes back
with its own address in the struct's first word, and a token they never touched comes
back as an all-zero struct. So there is no log scan over 54 million blocks, no archive
node (this endpoint has none), and no external API:

| Launchpad | Registry | What it launches |
| --- | --- | --- |
| `pons-v1` | `0xA5aAb3F0c6EeadF30Ef1D3Eb997108E976351feB` | Uniswap-V3/WETH pools in the 1% tier — directly tradeable — but nothing in ~18 days |
| `pons-v2` | `0x7eD598BcEf8bd9Edd8C97A195C6d13f40801EC7e` | bonding-curve launches; native-ETH curves are directly tradeable by AEGIs before graduation, ERC-20-quoted curves are discovery-only |
| `letscash` | *not published* | unverifiable; see below |

The decode reads word 0 and treats the reply's length only as a hint. V1 returns 13
words with `exists` at word 11, V2 returns 15 with `exists` at word 14, and pinning
either layout would mean a silent wrong answer the day a third factory ships with one
more field — where "silent wrong answer" means buying a token the operator excluded.

**Three verdicts, not two.** `LAUNCHED` and `NOT_LAUNCHED` are answers. `UNKNOWN`
means the chain did not answer — rate limit, transport failure, a reverting call — and
it is treated differently everywhere it matters: the token is refused for this tick,
the verdict is never cached, and it is never written to the blocklist, because a busy
endpoint is not evidence about a token and a blocklist entry is forever. The log line
is `RETRY`, not `UNSAFE`, and the status line carries `lp_unknown=` so an outage does
not read as a chain full of ineligible tokens.

**Pons V2 market handling.** Pons V2 is a real bonding-curve venue, not a
Uniswap-V2 pair. Each launch exposes its own curve with `buy()` / `sell()` and
`getReserves()`, and the curve uses the same quote asset that the eventual V4 pool
will use. AEGIs now recognises those curves directly. Native-ETH curves can be
screened, simulated and executed through the curve; ERC-20-quoted curves are
identified but remain non-executable from the ETH/WETH base path until a verified
quote-conversion route is added. Graduated V4 pools remain discovery-only for now.

```yaml
screening:
  launchpads: ["pons", "letscash"]   # default: provenance required
  # launchpads: ["pons"]             # drop the unverifiable name, same coverage
  # launchpads: ["all"]              # no provenance gate at all
```

**LetsCash is declared and unresolved, on purpose.** It is real — its own site names
chain 4663 — but it publishes no contract addresses, ships no source, and none of the
fingerprints in its documentation (`FIRST_CONFIG_ID()`, `getLaunchConfig(uint256)`, a
`pending(poolId)` hook, the "every token address ends in `cc`" claim) match any
contract on this chain. A v4 factory/hook pair fits several of those hints, and a full
selector scan of its runtime found no per-token registry to call, so nothing could be
verified even if the address were right. Guessing an address into a hard safety gate
either rejects everything or waves through tokens the operator excluded, so the name
stays listed with no address: it logs one warning at startup, contributes no verdict,
and accepts an operator-supplied address the moment one is published —
`launchpads: ["pons", "0x…"]` takes any factory that answers `getLaunchedToken`.

### The safety measurements

**The central measurement is a simulated round trip.** For each candidate the
analyser simulates buying with a small probe and immediately selling the proceeds
back in one `eth_simulateV1` request — state carried between the calls, against real
chain state, spending nothing and touching no funds. (This chain's public RPC serves
`eth_simulateV1` and does *not* serve `debug_traceCall`, both verified; with
`require_simulation: true` a token whose simulation cannot be completed fails
screening rather than passing unmeasured.)

It then compares the *measured* loss against the loss the pool's own maths says an
honest token must produce:

```
buy:   out_t = γ·p·T / (B + γ·p)
sell:  out_b = γ²·p·(B + p) / (B + γ²·p)            γ = 1 − fee
```

The interesting consequence, and the reason this works as a test: because
`(B+p)/(B+γ²p) > 1`, the price impact of the buy is handed back by the sell, so an
honest round trip loses **slightly less** than the two swap fees — converging up to
`1 − γ² = 0.5991%` as the pool deepens, and losing *less* in a shallow pool, not more.
The honest baseline is therefore depth-independent to within rounding. Anything left
over is not slippage; it is a tax, a blocked sell, or a transfer hook.

- `excess_loss ≈ 0.00%` → clean
- `excess_loss ≈ 5.87%` → a 3%/3% taxed token: measured, priced, rejected against
  `max_excess_loss: 0.05` — and deliberately **not** blocklisted, because a tax is a
  cost that can change, not a scam
- `excess_loss ≈ 99%` or a reverting sell → **honeypot**, score 0, permanent blocklist

Around that sit the cheaper checks, each contributing to a score and each reported in
full even when it only halves the score rather than failing outright:

| Check | What it rejects |
| --- | --- |
| Launchpad provenance | anything not launched by a configured launchpad — first, hard, and not overridable |
| Code presence and ERC-20 conformance | EOAs, proxies to nothing, non-tokens |
| Bytecode scan | mint-after-deploy, blacklist/whitelist gates, `onlyOwner` transfer hooks, pausable transfers, fee setters with no ceiling |
| Liquidity depth | pools too thin for the intended size — and it runs *before* the round trip, so a thin token costs one reserves read rather than a full simulation |
| LP concentration and lock | a single holder who can withdraw the float |
| Holder distribution | supply concentrated where one wallet can exit into you |
| Token age and volume history | dead projects and one-block launches |
| Fee-on-transfer detection | routers that must be called through the FOT variant |
| Blocklist | anything already judged, not re-litigated |

**Third-party reputation APIs are not used.** The brief allowed either them or local
chain-state checks; this is the local half, and only the local half. Token Sniffer,
GoPlus and Honeypot.is have no coverage of chainId 4663 to query, so an integration
would be an unreachable code path plus a network dependency in the screening hot loop —
and a "no data" answer that read as reassurance. The round trip above measures the
same property they would report, on the actual token, at the current block. If those
services add this chain, they belong in the score as one more weighted signal that
never blocks; that is listed under future work, not claimed here.

## Pillar 2 — executing trades

The execution engine is the only module that spends money, so it is written around
the ways that goes wrong on a fast chain.

- **Simulate before signing.** `eth_estimateGas` executes the call against current
  state, so a swap that would revert fails for free instead of on-chain for gas. When
  an approval is still missing, the (approve, swap) pair is simulated *together* via
  `eth_simulateV1`, so a honeypot's sell is discovered before paying for the approval
  that would enable it.
- **A revert is never retried.** The same call against the same state reverts again;
  paying gas twice to learn that is how an agent burns a balance overnight. Only
  transport failures — dropped connections, rate limits, `replacement transaction
  underpriced` — are retried, with the fee bumped 15% (`fee_bump_bps: 1500`).
- **Received, not returned.** The amount out is measured from the `Transfer` logs in
  the receipt, not from the router's return value: the fee-on-transfer router variants
  return `void`, and for a taxed token the received amount is smaller than the swap's
  own output. That difference is the tax actually paid, and it belongs in the audit
  trail.
- **A timeout is not a retry.** If a transaction has not confirmed in 20 s — about
  200 blocks, at this chain's measured 0.101 s block time — it is left pending and
  reported. Two copies of the same swap is the one
  outcome worse than none; the deadline in the calldata is what stops the first from
  landing later at a price nobody quoted.
- **Approvals are exact by default.** `approve_unlimited: false` costs one approval per
  position and leaves no standing claim on the balance.
- **The trade row is written before the broadcast.** If the process dies between
  sending and the receipt, that row is the only record the transaction exists.

**The base asset is native ETH, and the operator never touches WETH.** The wallet
holds ETH; that is the only balance an operator has to fund, and it is the balance
every number in the logs, the sizing and the trade rows is denominated in
(`base_asset: native`). Pools, however, are WETH-paired, so the *path* of every swap
is still `[WETH, token]` — which is why quoting, reserves and the arbitrage maths are
untouched by any of this. Only two things change:

- **The calldata variant.** A buy is `swapExactETHForTokens` with the size as
  `msg.value` and **no approval at all**; a sell is `swapExactTokensForETH`, which
  unwraps inside the swap. The router wraps and unwraps in the same transaction, so
  there is never a separate wrap to pay for. The four selectors this needs
  (`0x7ff36ab5`, `0x18cbafe5`, and the supporting-FOT pair `0xb6f9de95` /
  `0x791ac947`) were read off the live V2 router at
  `0x89e5DB8B5aA49aA85AC63f691524311AEB649eba` rather than assumed.
- **How the fill is measured.** A native sell pays the wallet ETH, and ETH movement
  emits no `Transfer` log, so "received, not returned" needs a different source: the
  WETH `Transfer` to the router, or the `Withdrawal` event, or — last — the native
  balance delta with the gas added back. Getting this wrong would not lose money, it
  would mis-report every sell, which is worse.

V3 is the one asymmetry: SwapRouter02 takes ETH in but can only pay ETH out through an
`unwrapWETH9` multicall built on the `ADDRESS_THIS` recipient sentinel, which is not a
convention to take on trust with funds. So a V3 sell delivers WETH, the plan says so in
its notes, and the runner unwraps it on the next tick (`auto_unwrap: true`, above
`min_unwrap_eth: 0.0002` — below that the withdrawal costs more gas than it recovers).
A failed unwrap is a warning, not a halt: wrapped proceeds still count as equity, so
the cost is a little flexibility and the next tick tries again. Setting
`base_asset: weth` restores the token-to-token variants for an operator who would
rather hold WETH.

**Venues.** Uniswap V2 forks are the primary one, and their quotes are computed
**locally** from `getReserves` — which is exact, not an approximation, because the
fork's swap fee was solved against `router.getAmountsOut` on a live pair and is exactly
997/1000. `self_check()` re-verifies that numerator and the CREATE2 derivation at
startup and refuses to trust local maths until both agree, so a fork that changed its
fee cannot silently skew every quote. At ~0.10 s blocks, one saved round trip per
candidate is a meaningful share of the decision window, and it lets thousands of pairs
be priced from one batched sweep. Uniswap V3 forks are the second venue and go through
QuoterV2 (`0x33e885eD…`, confirmed live), because a V3 quote cannot be derived from
`slot0` alone once a tick boundary is in the way.

The registry quotes every venue that has the pair and ranks by the amount actually
returned, not by pool depth — with one guard: a V3 quote that came from the local tick
approximation rather than the on-chain quoter cannot win, because signing an
`amountOutMinimum` derived from an optimistic estimate produces a revert, not a
bargain. Depth is used where depth is the right question: the spot price, and the
liquidity cap on size.

**On the `0x39ecce49` selector from the brief.** The two example transactions trade
through the aggregator at `0xdf040D9DF9fd77cd3318F54A8CDC44112a354c30`, and AEGIs
deliberately does **not** execute there. Decoding that calldata shows its last
parameter is a 13-word authorisation struct ending in a 65-byte ECDSA signature over an
*off-chain* quote, with fee recipients and a unix deadline baked in; the measured router
fee is 1.00%, which is 3.3× the V2 pool fee. An autonomous agent cannot mint that
signature — there is no public quote API to authenticate against — so a "buy through
`0x39ecce49`" path would be a code path that cannot run unattended. The brief's own
wording, *"or equivalent router functions"*, is the clause this build takes: it executes
through the permissionless Uniswap routers instead. The aggregator addresses and that
selector are kept in `constants.py` for the other half of the job — recognising a token
whose **only** exit is a signed-quote curve, and refusing it for having no
permissionless exit at all.

## Pillar 3 — generating profits

Three strategies ship enabled, all of them EV-positive only *after* costs, which is
why every signal carries its own cost estimate and is rejected if the edge does not
clear it:

- **Arbitrage** — the same token quoted on two venues. Both legs are quoted on-chain
  and the spread must clear `min_profit_bps: 25` *after* both swap fees, both slippage
  allowances and gas at the current price, with a `max_impact: 0.02` ceiling so the
  trade cannot move the price it is exploiting. If the second leg fails the position is
  *stranded* — recorded as such and handed to the exit logic, never retried blindly
  into a market that has already moved.
- **Momentum** — an SMA crossover, with the guards doing the real work. A cross is an
  *event*, compared against the previous sample, so it fires once rather than every
  tick while the averages stay crossed. A realised-volatility floor rejects crosses in
  a series that is flat — mid price comes from integer reserves, so an untraded pool
  crosses constantly and those crosses are the majority on a new chain. And the cross
  must agree with both the lookback return (`min_momentum: 0.01`) and RSI
  (`max_rsi: 80`), because an up-cross with RSI already at 90 is the top, not the
  start. It exits on thesis invalidation — the down-cross, or RSI at 95 — and leaves
  the stop and the profit ladder to the risk manager.
- **Mean reversion** — entry at `z ≤ −2`, but the number that has to clear the fee
  floor is `|z| × volatility`, not the z-score: a two-sigma dip in a token that barely
  moves is not worth two swap fees, and `min_edge: 0.012` says so. Guarded by the
  lookback return (a dip inside a `max_drawdown: 0.30` collapse is not a dip) and by an
  RSI band, so it buys weakness, not free-fall.

Paper mode prices fills through the same quoter the live path uses and then applies a
haircut (`paper.haircut_bps`) for the latency and adverse selection a simulator cannot
feel, so paper PnL is a lower bound rather than a forecast.

## Pillar 4 — repeating, safely

Risk is enforced in one place, `RiskManager.check_entry`, which can shrink or refuse a
size but **never enlarge** one. Every cap is evaluated and the *smallest* wins; the
winning cap's name is logged with the fill, so "why was this trade only 0.004 ETH"
always has an answer.

| Gate | Default | Notes |
| --- | --- | --- |
| Kill switch | `state/KILL` | `touch` it and entries stop within a tick |
| Daily loss limit | 15% of equity | Halts **entries only** — exits keep running. Anchored to equity at the first tick of each UTC day, and to the *day* rather than to the value, so an unfunded start logs one boundary and not one per tick; a wallet funded later re-anchors exactly once, because 15% of nothing is not a limit |
| Gas reserve | 0.0005 ETH | An unfundable exit is a total loss of that position, so a low native balance blocks new entries. ~8 swaps at the gas measured on this chain, which closes a full book of 5 |
| Absolute position | 0.02 ETH | Hard ceiling regardless of equity |
| Minimum position | 0.0005 ETH | A floor on *permission*, not on viability — the gas guard below is what decides whether a position this small is worth opening |
| Gas burden | 10% of the position | Entry + exit gas at the current price. The check that makes a small position honest: at 0.36 gwei a 250k+200k round trip is 0.000163 ETH, which is 21% of a 0.00077 ETH position, and a trade needing a 21% move to break even is a loss with extra steps |
| Per-token share | 10% of equity | Add-ons share the token's budget with what is already held, or the cap falls one buy at a time |
| Total exposure | 60% of equity | |
| Pool share | 1% of the base reserve | Unknown depth means *no depth information*, so the cap is skipped and the log says so — never treated as infinite depth |
| Max positions | 5 | |
| Stop loss | −18% | Moves to break-even once the first take-profit fills |
| Take profit | +35% (half), +100% (rest) | |
| Trailing stop | 22% off peak | Arms only once the peak has been +15% above entry |
| Max hold | 6 h | No exit signal in six hours frees the slot |
| Unpriceable position | — | Every pool gone means nothing to sell into: flagged and marked at zero, not sold into a void |

The gas burden gate is checked **last, against the size that survived every other
cap** — a trade cleared at its requested size and then shrunk by the exposure cap into
a position that cannot carry its own fees is exactly the case the gate exists for. It
prices a round trip as `gas_units_entry: 250000` plus `gas_units_exit: 200000` at the
live `eth_gasPrice`, and refuses with the arithmetic spelled out — *"gas cost 0.000163
ETH is 21.1% of position 0.000771 ETH … it needs 0.001629 ETH to clear"* — so the log
says what to change and not merely that something was wrong. A failed gas read means
*unknown*, not *free*: the gate is skipped, on the same principle as unknown pool
depth. `max_gas_pct_of_position: 0` disables it; zero gas units is a config error
instead, because a guard that is off while the config says it is on is worse than one
that was never there. Because the gate binds long before the 0.0005 ETH floor does, a
wallet too small to trade now says so at startup and in `--check`, naming the three
ways out rather than going quiet and never entering.

**Every rejection is attributable to a gate and a number.** A log line says *what*
happened; it does not survive log rotation and cannot be queried. So each gate also
writes a row — token, stage, gate, the value measured, the threshold it was measured
against, verdict, block, tick — and `store.explain_token(address)` reads them back as
one answer to "why did nothing happen with this token". The verdict vocabulary is the
point: `PASS`, `REJECT`, `UNKNOWN`, `STALE`, `UNAVAILABLE`, `ERROR` are six different
answers, and the four that are not `PASS`/`REJECT` all mean *we could not measure*,
which is a statement about the agent rather than about the token. A token whose most
recent non-pass row is an `UNKNOWN` is one to come back to; a `REJECT` is not. That
distinction is invisible in any log format, because both used to print one line and
then be forgotten. Traces are on by default (`trace_enabled: true`) and kept for
`trace_keep_days: 3` — the table grows with the number of tokens *rejected*, which on a
launchpad chain is nearly all of them, so retention is days and the housekeeping pass
that enforces it runs hourly off the tick loop. It also finally calls
`prune_prices_days: 7`, a documented knob whose method existed and was never invoked.

**Notifications are off by default and never on the critical path.** Telegram and
Discord are both `enabled: false` until an operator sets the env var, and when they are
on, a message is put on a bounded queue and delivered by a daemon thread — because an
HTTP POST to Telegram that takes 5 s is 5 s in which a stop-loss is not evaluated. When
the queue is full the *oldest* alert is dropped: during a burst the newest state is the
one worth knowing. A notifier that cannot deliver logs and moves on; it never raises
into the tick.

**The kill switch is a file, not a signal.** `touch state/KILL` stops entries within
one tick, survives a crash, a restart and a reboot, and needs no working RPC, no
process ID and no terminal. `SIGTERM`/`SIGINT` shut down gracefully, leaving positions open —
an exit is a trade, not cleanup. A watchdog thread watches tick completion, not
liveness: it alerts after 180 s without a completed tick and **trips the kill switch**
after three times that, on the reasoning that an agent which has not completed a tick
does not know what it holds, and should not act on a stale view the moment the RPC
comes back. It does not restart the loop. An unhandled exception alerts and exits
non-zero, so a supervisor decides whether a restart is wanted.

**A tick either happened or it did not.** SQLite runs in autocommit mode, so left
alone every write commits by itself and a process killed mid-tick keeps whichever
half landed first: a discovery cursor advanced past launches whose rows never
arrived, a price for a block whose pool state did not. Neither failure is loud —
nothing in the database afterwards says anything is missing, and the tokens in that
range are simply never screened again. So one `BEGIN IMMEDIATE` now wraps the whole
pass, with two kinds of seam cut into it on purpose. Each non-executing stage runs in
a savepoint, so a stage that raises undoes *its own* half-written state and the tick
continues without it, instead of committing the half. And anything that has just
changed the *chain* commits immediately: the trade row written as `sent` before the
broadcast, and the position, gas and realised-P&L bookkeeping that follows a fill. A
broadcast cannot be rolled back, so the record of it must not be waiting on the rest
of the tick to succeed — `store.flush()` commits and reopens, and it *refuses* below
the top level, because a savepoint cannot be made durable on its own. That turns
"nothing that can sign runs inside a block that may be rolled back" from a rule to
remember into one the code enforces. No new dependency: SQLite already had all of
this. The tests kill a real process mid-transaction with `fork` and `_exit` rather
than simulating a crash, and restart against the same file to show the cursor is not
rescanned from zero, an open position comes back with its levels intact, and a trade
left `sent` is still there to be reconciled.

---

## Configuration

`config.example.yaml` is the annotated inventory of all ~132 knobs across 18 sections.
Every value in it **is already the default**, which is a property the test suite
enforces in both directions — so any line may be deleted without changing behaviour,
and a knob added to the code without being documented there fails a test.

Three conventions, all enforced at load time with an error that names the key:

- `_eth` and `_gwei` are human units, converted to wei exactly once, at load.
- `_pct` is a **fraction**: `0.10` is ten percent. `max_position_pct: 25` is refused
  with *"percentages are fractions here"* rather than silently sizing the first trade
  at 25× equity. The rule follows the word, not the suffix: `max_gas_pct_of_position`
  is checked too, because the one knob spelling `_pct` mid-name would otherwise be the
  one where `10` means 1000% and the guard it controls silently never fires.
- `_bps` is basis points.

One section is a gate rather than a threshold. `screening.launchpads` decides what the
agent is *allowed* to buy, so an unknown name there fails the load with a `ConfigError`
naming the real ones rather than quietly widening or narrowing the universe, and an
empty list is refused outright because no token could ever pass it. A config file
written before this feature existed gains the gate rather than trading unfiltered: the
default lives in the code, not in the YAML. `launchpad_cache_size: 50000` bounds memory
and nothing else — a factory cannot retroactively launch a token, so a verdict cannot
go stale.

The loader also refuses configs that are internally impossible — a minimum position
above the maximum, take-profits out of order, `slippage_bps: 0` (reverts every swap),
`slippage_bps` above 3000 (accepts a 30% worse fill), an unknown `mode`. A config loader
that accepts a bad file is worse than one that crashes: the agent starts, looks
healthy, and trades on numbers nobody meant.

### The three modes

`mode` takes one of three values, and each one gets a *different object* rather than a
flag on a shared one — so a mode cannot be defeated by a stale boolean:

| `mode` | Balances | Chain reads | Signs | What it is for |
| --- | --- | --- | --- | --- |
| `paper` | virtual | none | never — no key, no client | rehearsing the position lifecycle |
| `dry_run` | **real** | **real** | never — no key at all | rehearsing the *live path* |
| `live` | real | real | yes | trading |

Anything other than these three fails the load, naming the ones that exist. The
distinction the code cares about is not "paper or not" but two separate questions —
*are balances simulated?* and *may this process put a transaction on the chain?* —
because a single flag answering both is how a third mode silently becomes a fourth.
Only `live` answers yes to the second, checked by equality, so every way of being
wrong comes out as "may not spend".

### Keys and secrets

**No key ever goes in the config file, and the loader enforces it.** `private_key`,
`mnemonic` and `keystore_password` are refused by name with the fix in the message,
and *any* bare 32-byte hex value anywhere in the file — under any innocent key name,
including inside a list — is refused as *"looks like a 32-byte private key"* with its
path. A normal 20-byte address is not mistaken for one.

Two supported paths, in order of preference:

```bash
# 1. An encrypted V3 keystore. The password lives in the environment, the key on disk.
#    wallet.keystore: state/keystore.json
export AEGIS_KEYSTORE_PASSWORD='…'

# 2. A raw key in the environment, for a throwaway hot wallet.
export AEGIS_PRIVATE_KEY='0x…'
```

Live mode without either is refused at startup: *"live mode needs a signing key"*. Dry
run is refused without an *address* and asked for no key at all — set `wallet.address`
if there is no keystore to read one from.
Five environment variables are read, and no others:
`AEGIS_PRIVATE_KEY`, `AEGIS_KEYSTORE_PASSWORD`, `AEGIS_TELEGRAM_TOKEN`,
`AEGIS_TELEGRAM_CHAT_ID`, `AEGIS_DISCORD_WEBHOOK`. The agent never logs a key, and
`--check` prints the address only.

---

## Going live

Paper mode is not a rehearsal to be skipped. Run it long enough to see the agent
screen, enter, and *exit* — the exit path is the one that matters and the one a short
run never reaches.

```bash
# 1. Paper, until the numbers are boring.
../.venv/bin/python main.py --config config.yaml
../.venv/bin/python main.py --status            # positions and the last 10 trades

# 2. Screen a token by hand and read the full verdict.
../.venv/bin/python main.py --screen 0x…        # exit code 1 if it fails

# 3. Fund a wallet with only what you accept losing, then rehearse the live path
#    against it without spending anything.
../.venv/bin/python main.py --dry-run --config config.yaml

# 4. Then, and only then:
#    mode: live   in config.yaml
export AEGIS_PRIVATE_KEY='0x…'
../.venv/bin/python main.py --check             # confirm the balances are real
../.venv/bin/python main.py --config config.yaml
```

Step 3 is the one paper mode cannot do. Paper mode proves the *strategy* works; it
never touches the code that builds, prices and signs a transaction, so a wrong router
address, a gas ceiling below the real basefee, an allowance that reverts or a quote
that is stale by the time it is estimated all survive any amount of paper trading and
surface on the first real fill. A dry run reads the funded balance, quotes against live
reserves, simulates `(approve, swap)` as a pair with `eth_simulateV1`, runs
`eth_estimateGas` against the real state, takes the real nonce, builds the EIP-1559
transaction — and then stops, logging what it would have sent:

```
DRY RUN buy 0x1234abcd: would send 10000000000000000 to 0x…, nonce 41,
        gas 187432 -> 234290 (not signed)
```

Three independent layers make that stop unconditional, each tested with the other two
removed: the engine returns before the send, `DryRunWallet` has no signing account at
all and raises `SigningDisabled` if asked, and the RPC client is constructed with
`allow_broadcast=False` so `eth_sendRawTransaction` refuses before it reaches the
network. The approval is *not* sent either — an approval is itself a transaction, and
it is the one step that would otherwise have to be paid for before the swap could even
be estimated.

A dry run needs an address but no key, which is the point: it runs on a machine with
nothing to leak. It takes the address from `wallet.address`, or from a keystore's
plaintext `address` field without decrypting it, or by deriving it from
`$AEGIS_PRIVATE_KEY` — in that order, and it keeps the address, never the key. Its
fills carry `status: dry_run` and are written to the simulated half of the trade log,
so a rehearsal can never move the daily-loss figure that gates tomorrow's sizing. It
also opens no position — nothing was bought, and a position the wallet does not own
would make every later exit a fiction — so the same entry is re-evaluated on later
ticks rather than tracked. That is the deliberate difference from paper mode, which
exists to rehearse the lifecycle.

Live mode logs one unmissable `LIVE MODE: real funds` warning with the kill-file path,
because a config that went live by accident should be obvious in the first screenful
and not on the first fill. A dry run logs its own, for the opposite reason: it looks
exactly like a live run right up to the send, and an operator who thinks those orders
are real will wonder where their fills went. `--paper` and `--dry-run` override the
file in the safe direction at any time.

Under a supervisor:

```ini
[Service]
ExecStart=/root/AEGIs/.venv/bin/python /root/AEGIs/aegis/main.py -c config.yaml
WorkingDirectory=/root/AEGIs/aegis
Environment=AEGIS_KEYSTORE_PASSWORD=…
Restart=always
RestartSec=10
```

### Flags

| Flag | Effect |
| --- | --- |
| `--config`, `-c PATH` | config file (default `config.yaml`) |
| `--paper` | force paper mode regardless of the file |
| `--dry-run` | force dry-run mode: the live path, never signed (exclusive with `--paper`) |
| `--once` | one tick, then exit |
| `--ticks N` | stop after N ticks |
| `--check` | validate, connect, report, exit — sends nothing |
| `--status` | portfolio and recent trades, then exit |
| `--screen TOKEN` | full safety verdict for one token |
| `--verbose`, `-v` | DEBUG on the console (the file is always DEBUG) |
| `--no-config-ok` | run on built-in defaults if the file is missing |

Exit codes: `0` clean, `1` startup or fatal error, `2` config error or wrong chainId,
`130` interrupted. `--screen` returns `1` when the token fails.

---

## Layout

```
main.py                  entry point, flags, --check/--status/--screen
config.example.yaml      every knob, annotated, all at their defaults
requirements.txt         five exact pins, and why the obvious ones are absent
aegis/
  config.py              load, validate, convert units, refuse secrets
  constants.py           chainId, router and factory addresses, selectors, topics
  runner.py              the tick loop, watchdog, signals, shutdown
  decision.py            the stage/outcome/state vocabulary, and the tracer
  logging_setup.py       rotating DEBUG file + readable console
  chain/     client.py   JSON-RPC: batching, retries, rate limit, capability probe
             codec.py    ABI encode/decode and every calldata builder
             wallet.py   keystore/env key, nonce management, signing
             erc20.py    batched metadata, balances, allowances
             base_asset.py  ETH or WETH, and what that implies for calldata
  dex/       registry.py pool discovery, per-token venue ranking, self-check
             univ2.py    V2 forks: exact local maths, CREATE2, LP analysis
             univ3.py    V3 forks: QuoterV2, slot0, tick liquidity
             base.py     the venue interface
  safety/    analyzer.py the verdict, the score, the cache, the blocklist
             simulate.py eth_simulateV1 round trip, tax and honeypot measurement
             bytecode.py opcode and selector scan for owner-controlled hazards
             checks.py   liquidity, holders, age, volume, LP lock
  strategy/  arbitrage.py momentum.py mean_reversion.py base.py
  execution/ engine.py   approvals, gas, send, confirm, what was received
             router.py   OrderPlan, Fill, slippage maths
             paper.py    PaperBroker — same interface, no key, haircut applied
             sweeper.py  unwraps WETH a V3 sell paid out, back to one ETH balance
  risk/      manager.py  every cap, the kill switch, the daily limit, exits
             portfolio.py positions, marks, equity, peak tracking
  data/      store.py    SQLite: trades, positions, safety verdicts, blocklist, stats
             prices.py   price history and the indicator inputs
             discovery.py new-pool scanning
  notify/    alerts.py   Telegram and Discord, queued off the tick thread
tests/                   561 tests: unittest, no network, no chain
docs/CHAIN_NOTES.md      what was measured on this chain, and what surprised us
```

State lives under `state/` by default: `aegis.db` (twelve tables: `trades`,
`positions`, `safety_reports`, `blocklist`, `allowlist`, `tokens`, `pools`,
`price_history`, `daily_stats`, `kv`, `decisions`, `token_state`), `aegis.log` (rotating, always DEBUG), and `KILL` if you create
it. Relative paths in the config resolve against the config file, not the working
directory, so cron and systemd behave the same as a shell.

---

## Testing

```bash
../.venv/bin/python -m unittest discover -s tests            # all 561
../.venv/bin/python -m unittest discover -s tests -p "test_safety.py" -v
```

Two rules make the suite worth running. **The fakes assert**: the fake RPC client
raises `AssertionError` from `eth_call`, `estimate_gas` and `send_raw_transaction`, so
a test that reaches the chain fails loudly instead of passing quietly — and the paper
tests prove paper mode signs nothing by the fact that those never fire. The dry-run
tests invert that: the chain reads are *expected* to fire and are counted, and what is
asserted absent is only the send. **The real
maths runs**: only `RpcClient` and `Registry.pools_for` are ever faked. Constant-product
quotes, slippage, tax arithmetic, position marks, PnL and the risk caps are the
production functions, exercised against integer reserves.

The negative tests are the point. A config loader is tested mostly on files it must
*refuse*; the safety analyser mostly on tokens it must *reject*.

Two of the modules test the deliverables rather than the code, because both are claims
that rot silently:

- `test_dependencies.py` asserts the package imports with `web3`, `pandas`, `numpy` and
  `pytest` **blocked at the meta path** — in a subprocess, so a lazy import inside a
  function is caught too — and that every third-party module the package imports is
  pinned in `requirements.txt` at the version actually installed.
- `test_readme.py` checks every `knob: value` pair quoted in this file against
  `DEFAULTS`, and the flag table against the real `argparse` parser. Writing it found
  eight stale numbers in the first draft of this README and one documented flag that
  does not exist. Prose does not fail a build unless something makes it.

Three production bugs were found by writing these tests, and each has a regression
test named after the symptom:

- A cache hit in the safety analyser raised `AttributeError` on every call
  (`SafetyReport` is a slots dataclass, so the obvious copy has no `__dict__`). The
  runner swallows analysis errors, so every already-screened token quietly went
  *unscreened* for the rest of the cache window — the exact opposite of what the cache
  is for. Fixed with `dataclasses.replace`, plus a deep copy of the two mutable fields
  so a caller that annotates its report cannot corrupt the entry.
- `logging_setup.setup()` set only the root logger to DEBUG. A logger's own level is
  checked before any handler runs, so any level left on the `aegis` logger dropped
  debug records before the file's DEBUG handler saw them — and the file, not the
  console, is what a post-mortem reads.
- `DEFAULTS` never met the validators, so `execution.fee_bump_pct: 15` shipped as a
  default that would have been *refused* if an operator had written it down (15 as a
  `_pct` is 1500%). It is `fee_bump_bps: 1500` now, and a test dumps `DEFAULTS` to YAML
  and loads it back so no default can ever again mean something the loader rejects.

---

## Reused work, and what was written instead

No trading bot, sniper or strategy framework was forked. The searchable ones are
tuned for Ethereum mainnet or BSC and encode assumptions this chain breaks — a
constant router set, a working `eth_getLogs` range, a subgraph, an aggregator API,
`pandas` in the hot loop. Adapting one is not obviously less work than writing 9,500
auditable lines, and the result is code nobody in this repo can defend line by line.

What *was* reused is the well-audited primitive layer, pinned exactly:

| Dependency | Pin | Why |
| --- | --- | --- |
| `eth-abi` | 6.0.0 | ABI encode/decode. Hand-rolling this is how you learn about dynamic-offset bugs on mainnet. |
| `eth-account` | 0.14.0 | Key handling, EIP-155/1559 signing, V3 keystore. Never reimplement a signer. |
| `eth-utils` | 6.0.0 | Keccak, checksum addresses, hex conversion. |
| `requests` | 2.34.2 | HTTP with connection pooling for the JSON-RPC transport. |
| `PyYAML` | 6.0.3 | `safe_load` only. |

Pinned exactly rather than loosely: an agent that signs transactions should not have
its ABI encoder or its signer change underneath it on a `pip install` six months from
now.

**Absent on purpose**, and the brief did suggest them:

- **`web3.py`.** The agent needs eight RPC methods, batched, behind its own rate
  limiter and retry policy, with selectors pinned as constants. `web3` supplies a
  contract abstraction on top of those eight and a middleware stack that hides which
  call went out and when — the opposite of what an unattended signer wants. It stays
  in `requirements.txt`'s rationale section rather than in the install, and a test
  asserts the package imports with `web3` *blocked at the meta path*, so the claim
  cannot rot.
- **`pandas` / `numpy`.** The indicators run over at most 600 floats per token.
  `mean()` and `std()` over a `deque` is a dozen lines; ~50 MB of wheels and a
  multi-second import on a small VPS is not a trade worth making. Same test blocks
  both.
- **`pytest`.** The suite is stdlib `unittest`, so a fresh box runs it with nothing
  installed but the five pins above.

---

## Limitations

Read these before funding anything.

- **The strategies are not an edge.** SMA crossover and z-score reversion are textbook
  and public. What is defensible here is the *cost accounting* — every signal must
  clear fees, slippage and gas before it is taken — and the risk engine that bounds
  the damage when a signal is wrong. On a low-liquidity chain, fees and adverse
  selection are the dominant term, and this agent can lose money while behaving
  exactly as designed.
- **Paper PnL is a lower bound, not a forecast.** The haircut approximates latency and
  adverse selection; it cannot reproduce being front-run, and paper mode never
  competes for block space.
- **Screening bounds a known set of scams.** The round trip catches honeypots and
  taxes; the bytecode scan catches the common owner-controlled hazards. A contract
  that behaves for the probe and turns hostile on the third sell, an upgradeable proxy
  whose implementation changes after screening, or a social rug where the code is
  clean and the team leaves — none of these are detectable this way. The cache expiry
  narrows the window; it does not close it.
- **The launchpad gate is a provenance rule, not a safety guarantee — and it is
  expensive.** A token being launched by Pons says nothing about whether its owner can
  mint or blacklist, which is why every other check still runs behind it. In the other
  direction it excludes almost everything: with the default list, one of 89 recent
  WETH-quoted tokens was eligible, because the active Pons factory launches onto curves
  this agent cannot route and the dormant one launches nothing. An operator who wants
  the agent to trade at all should read that paragraph in Pillar 1 before funding it,
  and decide deliberately between `launchpads: ["pons"]` and `launchpads: ["all"]`.
- **There is no external corroboration.** Every verdict is computed by this process from
  chain state (Pillar 1 says why), so nothing here benefits from another party having
  already flagged a scam that these heuristics happen to miss, and no human sees a
  token before it is bought. A hazard that is obvious to a human reading the project's
  chat is invisible to this agent.
- **Discovery depends on the endpoint.** Where `eth_getLogs` ranges are limited,
  discovery is narrower than it looks; `--check` reports which methods the endpoint
  actually implements, and that report is worth reading rather than assuming.
- **One process, one wallet, no MEV protection.** No private mempool, no bundle
  submission. On a chain with active searchers, sandwiching is a real cost that the
  slippage bound limits but does not prevent.
- **Timeouts are reported, not resolved.** A transaction left pending is exactly that;
  the calldata deadline stops it landing at a stale price, and the next tick re-reads
  the chain, but reconciliation is not automatic.
- **Not audited, not advice.** This is software that spends money in a hostile
  environment. Fund it with what you accept losing entirely.

## Future work

Roughly in order of expected value per hour spent:

1. **Reconciliation of pending transactions** — walk `sent` rows with no receipt on
   startup and resolve them from the chain before trading again. The store already has
   everything needed; only the walk is missing.
2. **The launch sniper.** `data/discovery.py` already produces new pools and the
   analyser already scores them; what is missing is the tighter risk profile a
   first-block entry needs (smaller size, harder liquidity floor, faster expiry).
3. **Copy trading.** Watch a set of addresses, mirror entries that pass screening at
   your own size. Cheap to add on the existing discovery and safety layers; the hard
   part is choosing whom to follow.
4. **LP auto-compounding** — a real yield source on a chain where trading edge is
   thin, and the LP-analysis code needed to judge a pool already exists.
5. **MEV-aware submission** if this chain grows a private relay.
6. **Walk-forward backtesting** over the stored price history, so a strategy change is
   measured before it is deployed rather than after.
7. **A status dashboard.** `--status` and the tick report are the data; a read-only
   HTTP view of them is a small job. It must stay read-only and bound to localhost —
   an endpoint that can move funds is a second, unaudited way to lose them.
