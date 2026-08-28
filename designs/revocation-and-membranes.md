# Revocable stubs and membranes: a cross-team API proposal

**Status:** draft for review. **Audience:** the workerd and Cap'n Web teams.

Both workerd and Cap'n Web now have working implementations of revocable RPC stubs behind the same
public API, `RpcStub.revocable()`. workerd's is gated on the `experimental` compat flag pending
API review. Before promoting, two things need deciding:

1. Is this the API we want in **both** runtimes long-term? The two implementations already agree
   on most semantics but diverge in a few observable ways (§3), and each divergence is a decision.
2. Should we instead (or eventually also) expose a **general membrane API**, of which revocation
   is one built-in policy? The current API is designed specifically for revocation.

This doc is about API shape and observable semantics only, not how either side is implemented.

Reference material:

- workerd: branch
  [`revocable-rpc-stubs`](https://github.com/Maximo-Guk/workerd/tree/revocable-rpc-stubs)
- Cap'n Web: branch
  [`revocable-stubs`](https://github.com/Maximo-Guk/capnweb/tree/revocable-stubs)
- Prior art:
  [`capnp/membrane.h`](https://github.com/capnproto/capnproto/blob/v2/c++/src/capnp/membrane.h),
  whose design preamble (lines 23–48) is the philosophy this doc leans on, and
  [Mark Miller on membranes](http://www.eros-os.org/pipermail/e-lang/2003-January/008434.html).

## 1. The proposed API

Both branches expose the same shape:

```ts
RpcStub.revocable(target)  // target: plain object, function, RpcTarget, or an existing RpcStub
// => { stub: RpcStub, revoker: RpcRevoker }

interface RpcRevoker extends Disposable {
  revoke(reason?: unknown): void;  // idempotent; `reason` becomes the rejection error
  readonly revoked: boolean;
  // [Symbol.dispose]() revokes with the default reason
}
```

Calling `revoke()` breaks the stub and, transitively, every capability derived from it — `dup()`s,
stubs obtained from call results (awaited or pipelined), and copies passed on to RPC peers — even
across process boundaries, without disturbing the rest of the session.

## 2. Where the implementations already agree

These are effectively spec already, since both sides independently landed on them:

- The return shape and revoker interface above; default reason `new Error("RPC stub was
  revoked.")`.
- `revoke()` is idempotent; the first reason wins.
- `dup()` of a revocable stub shares its revocation; there is no way to launder a revocable stub
  into a non-revocable one via `dup()` or via passing it onward.
- Wrapping an existing stub acts like `dup()`: the revocable stub is a new stub sharing the same
  target, and the original stub is unaffected by revocation.
- The revoker is not serializable — revoke authority stays wherever `revocable()` was called.
- Revoking and disposing are distinct: disposing the stub does not revoke, and revoking does not
  require the holder's cooperation.
- GC never revokes; only an explicit `revoke()` (or disposing the revoker) does.

## 3. Where they diverge

Each row is an observable behavior difference and therefore a decision to make for parity:

| Decision                | workerd                                                                                                                                                                                                                                | Cap'n Web                                                                                           |
| ----------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| Directionality          | Bidirectional: capabilities the *caller passes in* as arguments are also revoked                                                                                                                                                       | Outbound only: results flowing out are revocable; capabilities passed in as arguments are untouched |
| Severing timing         | Callers see rejections from the moment `revoke()` returns, but severing is eventual: a last-moment call may still run against the target ("revoke() removes authorization; it does not guarantee that the last-moment call never ran") | Synchronous: everything is severed before `revoke()` returns                                        |
| Target disposer         | Unspecified                                                                                                                                                                                                                            | The target's disposer runs exactly once at revoke time                                              |
| Reason fidelity         | `Error` reasons tunneled as name + message over the RPC boundary                                                                                                                                                                       | Any JS value, thrown verbatim (in-process; over the wire, subject to error serialization)           |
| Open streams            | Unspecified                                                                                                                                                                                                                            | Documented caveat: streams already open through the stub are not severed by revocation              |
| Revoker context binding | Usable only within the request that created it                                                                                                                                                                                         | Usable from anywhere in the process                                                                 |
| TypeScript types        | Not yet published                                                                                                                                                                                                                      | Published (`RpcRevoker`)                                                                            |

The directionality row is the deepest one: it is really the question of whether `revocable()` *is*
a membrane. workerd's semantics today are the membrane answer; Cap'n Web's are "the outbound half,
which is the direction that matters for revocation."

## 4. Option A: standardize `revocable()` as-is

Keep the narrow, revocation-specific API in both runtimes. The spec is §2 plus the §3 table with
each divergence resolved one way or the other.

Trade-offs:

- Both implementations already exist; the remaining cost is only reconciling §3, not new code.
- It covers the dominant use case directly — per `membrane.h`, "the most common use case for a
  membrane is revocation" — with a small, teachable surface: one static method, one handle type.
- Each *future* interposition need (attenuation, auditing/logging, taming a stub before handing it
  to less-trusted code) becomes another one-off static method designed from scratch.
- The §3 decisions must be made without a framework that answers them; e.g. directionality has no
  obviously principled answer unless you first decide whether this is a membrane.

## 5. Option B: a general membrane API

Expose the mechanism rather than the one policy — sketch, not a worked design:

```ts
RpcStub.membrane(target, policy)  // => { stub: RpcStub, handle: MembraneHandle }

interface MembranePolicy {
  // Each hook may pass the call through, redirect it to a different target, or throw.
  onOutbound?(call: MembraneCall): void;
  onInbound?(call: MembraneCall): void;
  // Plus some revocation trigger (an AbortSignal, or a method on the returned handle).
}
```

`revocable()` then becomes the trivial built-in policy (no hooks, just the trigger).

Trade-offs:

- One mechanism, many policies: attenuation, logging, and taming come for free later instead of
  each becoming a new API.
- The cost is asymmetric: the runtime already has a general membrane mechanism internally, while
  pure-JS Cap'n Web would need to build one before the API could ship with parity.
- Cap'n Proto's own `MembranePolicy` — the closest prior art for what `policy` would look like —
  carries an unresolved `TODO(cleanup)` (`membrane.h:120`) conceding that it conflates reversible
  and one-way membranes. Freezing a JS mirror of it now bakes in known regret.

## 6. Does shipping `revocable()` lock us in?

The API *shape* does not: `RpcStub.revocable(target)` can later be respecified as sugar for
`RpcStub.membrane(target, trivialPolicy)`, and the two can coexist. Cap'n Proto itself is
precedent that narrow and general tools live side by side: `capnp::RevocableServer<T>`
(non-transitive, single object, a lifetime-safety tool) coexists with `capnp::membrane()`
(authority containment), and neither obsoletes the other.

What *would* lock us in are the semantics we pin when resolving §3, because applications will
depend on them:

- **Directionality.** Only the bidirectional answer is compatible with "revocable() is a membrane
  with the trivial policy." Shipping outbound-only semantics forecloses that respecification.
- **Severing timing.** Promising synchronous severing forecloses membrane-based semantics, which
  can only promise "rejection is guaranteed once `revoke()` returns; severing is eventual." The
  weaker promise stays compatible with both.
- **Round-trip identity.** Whether a capability that passes in through a revocable stub and back
  out is recognized as the original (and escapes revocation) is observable, and either answer
  will be depended on. capnp's default is deliberately yes — "APIs commonly rely on the fact that
  a capability obtained and then passed back can be recognized as the original capability"
  (`membrane.h:40-46`) — at the cost that "Bob can create long-term irrevocable connections."

So a viable middle path is to ship Option A now but specify it *as* a membrane with the trivial
policy — bidirectional, eventual severing, membrane identity semantics — so that Option B remains
purely additive later.

## 7. Open questions for review

- Each divergence row in §3: directionality, severing timing, target disposer, reason fidelity,
  open streams, revoker context binding.
- Round-trip identity (§6): should a capability that round-trips through a revocable stub stay
  revocable?
- How does a *holder* of a revocable stub detect revocation? Cap'n Web has `onRpcBroken`; what is
  the workerd equivalent?
- Non-`Error` revocation reasons: what should survive the wire, and should both sides match?
- Path to stability: workerd's gate (experimental → compat flag, or ungated?) and adding the
  TypeScript types workerd currently lacks.
