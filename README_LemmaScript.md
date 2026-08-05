# MCP TypeScript SDK — Verified with LemmaScript

[![LemmaScript verified](https://img.shields.io/github/actions/workflow/status/midspiral/mcp-sdk-lemmascript/lemmascript.yml?branch=lemmascript&label=LemmaScript%20verified)](https://github.com/midspiral/mcp-sdk-lemmascript/actions/workflows/lemmascript.yml)

Fork of [modelcontextprotocol/typescript-sdk](https://github.com/modelcontextprotocol/typescript-sdk) with the connect-time probe classifier verified **in place** using [LemmaScript](https://github.com/midspiral/LemmaScript)'s Dafny backend. Five production functions in `packages/client/src/client/probeClassifier.ts` are checked with 8 Dafny verification obligations and 0 errors. The classifier remains in its production module; no clean-room verification copy or `autohavoc` is used.

The production decision logic is unchanged. The only executable edit names the ordinary `Error` created for an HTTP close in a local variable, so the adjacent havoc and assumption can refer to it; evaluation and the returned value are the same as in the original direct call.

The proof focuses on the classifier's safety boundary: modern or corrective negotiation must have corresponding evidence, while authentication and transport failures must not be silently reclassified as an era verdict. Calls outside LemmaScript's supported fragment are isolated with explicit, reviewable `//@ havoc`, `//@ assume`, and `//@ extern` annotations.

## What's verified

### End-to-end outcome routing

`classifyProbeOutcome` proves:

- A `modern` verdict can only come from a `result` probe outcome.
- A `corrective` verdict can only come from an RPC error or an HTTP error carrying a parsed RPC error.
- `auth-required` always becomes a typed `error`.
- A stdio timeout or close becomes `legacy`; the corresponding HTTP timeout or close becomes a typed `error`.

These are implications over every value in the modeled `ProbeOutcome` and `ProbeClassifierContext` datatypes, not a finite fixture table.

### Discover-result selection

For `classifyResult`:

- It never returns `corrective`.
- Every selected modern version belongs to both the client's preference list and the parsed server `supportedVersions` list.
- An `error` verdict implies that legacy fallback was unavailable.

The codec call itself is havoced, so this proves the selection and fallback logic for any projected codec result; it does not prove that the external codec accepts exactly the intended wire values.

### RPC error classification

For `classifyRpcError`:

- Any code other than `-32022` returns `legacy`.
- The function never returns `modern`.
- A `corrective` verdict implies code `-32022` and selects a version from `context.clientModernVersions`.

The `NOT_PROBE_RECOGNIZED.has(code)` branch is the sole `//@ skip`. Both that branch and its fallthrough return `legacy`, so omitting the membership test preserves the modeled return behavior while avoiding verification of JavaScript `Set` construction and lookup.

### HTTP and network policy

`classifyHttpError` proves that it never returns `modern`, and that HTTP 401, 403, and every 5xx response return a typed `error`. `classifyNetworkError` proves that it never returns `modern` or `corrective`; Node network failures and non-opaque browser failures return a typed `error`.

## Explicit abstraction boundary

There is no `autohavoc`. Each unsupported expression is named at its source location:

- The external discover codec's projected result is havoced; overlap selection after that boundary is verified.
- The structurally narrowed RPC/HTTP helper parameters are represented as their enclosing `ProbeOutcome` union variants in Dafny and guarded by variant preconditions. This is a nominal proof-model adaptation, not a runtime cast.
- `parseSupportedList`, `parseRequested`, and `modernProtocolVersions` are havoced. Consequently, the proof covers the control-flow consequences of their outputs, not those helpers' parsing/filtering correctness.
- SDK error construction is havoced. Error payload contents and concrete error subclasses are outside the proof; the surrounding verdict tags are verified.
- The HTTP JSON-RPC parser result is havoced and immediately assumed to be the RPC-error union variant when present. This represents the parser's untagged runtime record in the Dafny union model.
- `isOpaqueFetchTypeError` is an uninterpreted `//@ extern` predicate because its implementation uses `instanceof`. The ordinary `Error` synthesized for an HTTP close is havoced and assumed not to satisfy that predicate.
- Only the decision-neutral `NOT_PROBE_RECOGNIZED` membership branch is skipped.

The proof therefore establishes the classifier's branching and selection invariants **conditional on these explicit seams**. It does not verify JSON parsing, codec conformance, protocol-version filtering, JavaScript exception identity, or SDK error-object contents.

The boundary can be audited directly:

```sh
rg -n '//@ (havoc|skip|assume|extern)' packages/client/src/client/probeClassifier.ts
```

## Files

```text
packages/client/src/client/probeClassifier.ts       production source plus contracts
packages/client/src/client/probeClassifier.dfy.gen regenerated Dafny translation
packages/client/src/client/probeClassifier.dfy     verified Dafny target
LemmaScript-files.txt                              batch verification manifest
.github/workflows/lemmascript.yml                  reusable verification workflow
```

The `.dfy` target currently equals `.dfy.gen`; all contracts and abstraction annotations live in the TypeScript source, with no hand-written Dafny proof additions.

## Verify

Prerequisites: Node.js 18 or newer, Dafny 4.x, and a sibling LemmaScript checkout with its tool dependencies installed.

```sh
git clone https://github.com/midspiral/LemmaScript.git ../LemmaScript
cd ../LemmaScript && npm install && cd -
../LemmaScript/tools/check.sh dafny
```

Expected result:

```text
Dafny program verifier finished with 8 verified, 0 errors
```

For a single-file regeneration after changing annotations:

```sh
node ../LemmaScript/tools/dist/lsc.js regen --backend=dafny \
  packages/client/src/client/probeClassifier.ts
```

## LemmaScript improvements found by the case study

The in-place extraction exposed three backend name-generation bugs, each now covered by `examples/nameClash.ts` in LemmaScript:

- A payload-bearing implication such as `\result.kind === "modern" ==> ...` used the raw `\result` spelling in a match binder, producing an invalid Dafny identifier. Match-binder prefixes are now sanitized.
- Hyphenated discriminants such as `"rpc-error"` were escaped in datatype declarations but not discriminator predicates. Dafny produced a subtraction expression instead of `rpc_error?`, while Lean emitted `.rpc-error` instead of `.«rpc-error»`.
- A union constructor and local payload variable can share a source name (`error(error)`). Constructors are now datatype-qualified when their emitted name is already occupied.

## Next steps

The highest-value extension is to replace the parser/filter havocs with verified contracts. In particular, characterizing `modernProtocolVersions(supported)` as a subset of `supported` would let the corrective theorem state that the selected version appears in both the client's and server's lists. Verifying `parseSupportedList` would then connect the `-32022` theorem all the way back to `outcome.data`.
