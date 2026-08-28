# Revocable stubs and membranes: a cross-team API proposal

**Status:** draft for review. **Audience:** the workerd and Cap'n Web teams.

Both workerd and Cap'n Web now have working implementations of revocable RPC stubs behind the same
public API, `RpcStub.revocable()`. workerd's is gated on the `experimental` compat flag with a
`TODO(soon)` to promote it after API review. Before promoting, two things need deciding:

1. Is this the API we want in **both** runtimes long-term? The two implementations already agree
   on most semantics but diverge in a few observable ways (§3), and each divergence is a decision.
2. Should we instead (or eventually also) expose a **general membrane API**, of which revocation
   is one built-in policy, the way Cap'n Proto C++ layers `capnp::membrane()` under everything?

Reference material:

- workerd implementation: branch
  [`revocable-rpc-stubs`](https://github.com/Maximo-Guk/workerd/tree/revocable-rpc-stubs)
  (`src/workerd/api/worker-rpc.{h,c++}`)
- Cap'n Web implementation: branch
  [`revocable-stubs`](https://github.com/Maximo-Guk/capnweb/tree/revocable-stubs)
  (`src/index.ts`, `src/core.ts`)
- Prior art:
  [`capnp/membrane.h`](https://github.com/capnproto/capnproto/blob/v2/c++/src/capnp/membrane.h),
  whose design preamble (lines 23–48) is the philosophy this doc leans on, and
  [Mark Miller on membranes](http://www.eros-os.org/pipermail/e-lang/2003-January/008434.html).

## 1. What is implemented today

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

Mechanically the two differ. workerd wraps the target capability in `capnp::membrane()` with a
revocation-only policy (the pre-existing `RevokerMembrane` in `util/completion-membrane.h`, whose
call hooks are no-ops; its only job is `onRevoked()`), so it inherits full Cap'n Proto membrane
semantics. Cap'n Web wraps the stub's `StubHook` in a `RevokerStubHook` decorator
(`src/core.ts`) on the **outbound** side only, plus a new `RpcPayload.rewriteStubs()` primitive to
extend the wrap into pulled resolutions; every hook derived from one `revocable()` call shares one
kill switch, which is what makes revocation transitive.

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

| Decision                 | workerd (`capnp::membrane`)                                                                                                                                | Cap'n Web (`RevokerStubHook`)                                                                    |
| ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| Directionality           | Bidirectional membrane: capabilities the *caller passes in* as arguments are wrapped and revoked too                                                        | Outbound only: results flowing out are wrapped; capabilities passed in as arguments are untouched |
| Severing timing          | Callers see rejections from the moment `revoke()` returns, but the membrane severs on the next event-loop turn; a last-moment call may still run against the target ("revoke() removes authorization; it does not guarantee that the last-moment call never ran") | Synchronous: every wrapped hook is broken before `revoke()` returns                               |
| Target disposer          | Emergent from the capability being dropped; not explicitly specified or tested                                                                               | The target's disposer runs exactly once at revoke time; tested                                    |
| Reason fidelity          | `Error` reasons tunneled as name + message over the RPC boundary                                                                                             | Any JS value, thrown verbatim (in-process; over the wire, subject to error serialization)         |
| Open streams             | Not separately specified                                                                                                                                     | Documented caveat: streams already open through the stub are not severed by revocation            |
| Revoker context binding  | Bound to the creating request's `IoContext`; usable only within it                                                                                           | No equivalent concept; usable from anywhere in the process                                        |
| TypeScript types         | Not yet added (`types/defines/rpc.d.ts` still shadows `RpcStub`)                                                                                             | Shipped (`RpcRevoker` in `src/index.ts`), with a known gap around `ElideStub` in nested stubs     |

The directionality row is the deepest one: it is really the question of whether `revocable()` *is*
a membrane. workerd's answer today is yes (it literally is one); Cap'n Web's answer is "the
outbound half, which is the direction that matters for revocation."

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

`revocable()` then becomes the trivial built-in policy (no hooks, just the trigger), exactly as
workerd's `RevokerMembrane` already is.

Trade-offs:

- One mechanism, many policies: attenuation, logging, and taming come for free later instead of
  each becoming a new API.
- workerd is nearly free — its `revocable()` is already `capnp::membrane()` with a policy object,
  so this mostly exposes what exists.
- Pure-JS Cap'n Web must build what `membrane.c++` does: both directions, recursive wrapping of
  payload capability tables (`rewriteStubs()` is a seed for this, outbound-only today),
  the identity map for unwrap-on-return, `map()` captures, stream handling, and every wire point
  where stubs are minted. Kenton's "the pure-JS implementation may need to do more work" is
  concretely confirmed by the branch.
- Cap'n Proto's own `MembranePolicy` carries an unresolved `TODO(cleanup)`
  (`shouldResolveBeforeRedirecting`, `membrane.h:120`) conceding the policy API conflates
  reversible and one-way membranes — freezing a JS mirror of it now bakes in known regret.

A middle path the teams may converge on: ship Option A, but *specify* `revocable()` as a membrane
with the empty policy (bidirectional, membrane identity semantics), so that Option B remains
purely additive later. Cap'n Proto itself is precedent that narrow and general tools coexist:
`capnp::RevocableServer<T>` (`capability.h` — non-transitive, single local object, synchronous,
a lifetime-safety tool) lives alongside `capnp::membrane()` (authority containment), and neither
obsoletes the other.

## 6. Open questions for review

- Each divergence row in §3: directionality, severing timing, target disposer, reason fidelity,
  open streams, revoker context binding.
- The unwrap-on-return identity hole: should a capability that round-trips through a revocable
  stub stay revocable? capnp's default is deliberately no ("Bob can create long-term irrevocable
  connections"), because "APIs commonly rely on the fact that a capability obtained and then
  passed back can be recognized as the original capability" (`membrane.h:40-46`).
- How does a *holder* of a revocable stub detect revocation? Cap'n Web has `onRpcBroken`; what is
  the workerd equivalent?
- Non-`Error` revocation reasons: what should survive the wire, and should both sides match?
- Path to stability: workerd's `TODO(soon)` (experimental → compat flag, or ungated?) and adding
  the TypeScript types workerd currently lacks.
