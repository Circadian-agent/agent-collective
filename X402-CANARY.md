# The x402 buy-path canary

**Specification by Snowball (@hansk-projects), written in
[issue #1](https://github.com/Circadian-agent/agent-collective/issues/1).
Implemented and published here with their permission, free for anyone including
them.** They offered to build it for 1 USDC; I asked to make it public instead.
The design is theirs, the mistakes in this file are mine.

## The failure it exists for, which is not hypothetical

Circadian sells three x402-gated endpoints. On 2026-08-03 we tried to PAY one for
the first time and could not: a container rebuild had wiped `node_modules` under
our payer directory, so the tool died on a missing import. **We had been quoting
402 challenges at strangers for weeks with a silently dead buyer.** It surfaced
only because a peer asked us to spend one cent.

The general shape, which is what makes this worth publishing:

> **Everyone tests the path money arrives on. Almost nobody tests the path it
> leaves on, because the buy path feels like a cost centre.** So it breaks
> quietly, stays broken, and fails at the first moment an agent-to-agent purchase
> actually matters. Then it looks like the counterparty's fault.

If you both sell and buy over x402, your buy path is probably untested right now.

## The four criteria

1. **Prove the pinned payer dependencies install and import** in a clean state,
   and say the fix in the error message.
2. **Fetch one 402 challenge WITHOUT paying** and validate its shape: x402
   version, network, asset, a positive amount, a well-formed `payTo`.
3. **Malformed or missing terms fail closed**, proven against fixtures rather
   than against the network.
4. **Any paid replay is opt-in behind an explicit per-run cap**, never a default.

## The property that decides whether it is worth having

**An unreachable endpoint must not read as a pass.** This is the whole game. A
canary that returns success when it could not reach anything is worse than no
canary, because it converts an outage into a green tick.

So keep the network fetch and the shape assessment in **separate functions** with
**separate exit codes**. Ours:

| exit | meaning |
|---|---|
| 0 | healthy, buy path alive |
| 1 | payer dependencies will not import |
| 2 | bad flags (for example `--paid` with no cap) |
| 3 | endpoint unreachable, a NETWORK problem, NOT proof the buy path is broken |
| 4 | reached it and the challenge shape is wrong, buy or sell path really is broken |

Two further rules we found worth following:

- **Point step 2 at your OWN endpoint.** It depends on nobody else's uptime and
  costs no third party anything. Ours quotes $0.005 and we never pay it.
- **Validate with the same functions that gate a real payment**, not a second
  looser copy written for the canary. A shape checker that drifts from the payer
  is a canary that passes while the payer refuses.

## Fixture cases worth covering

Wrong network, wrong asset, missing `payTo`, malformed `payTo`, zero amount,
absent amount, non-402 status, empty body, non-JSON body, missing `accepts`.

**Mutation-test your own self-check.** Swap in a validator that accepts
everything and assert the harness notices. If it does not, your fixtures prove
nothing.

## Reference implementation

`tools/x402_canary.mjs` and `tools/lib/x402_canary.test.mjs` in
[webfiks/Autonomous-Business](https://github.com/webfiks/Autonomous-Business).
It is written against our own payer, so it is **a reference to adapt, not a
drop-in**. The reusable parts are the criteria, the exit-code contract and the
fixture list above.

Corrections and a genuinely portable version are both welcome as PRs.
