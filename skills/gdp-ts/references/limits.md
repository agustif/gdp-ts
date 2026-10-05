# What this does not guarantee

TypeScript is not Haskell, and this is not a proof assistant. Be clear-eyed
about the boundary:

| Concern | Haskell | TypeScript, with gdp-ts |
|---|---|---|
| Forging a proof | Impossible outside the trusted module | `{} as UserIsProjectAdmin<U, P>` compiles. The honest path never needs `as`, so the [lint preset](recipe.md#6-turn-on-the-lint-preset) flags assertions to proof types (and, in strict mode, every `as`/`any` outside `proofs/`). Review that directory with care. |
| Minting from the wrong place | Constructor not exported | `defineProof` is importable anywhere; the lint preset confines it to `proofs/`. |
| Phantom type parameters | Nominal | Must be used structurally or TypeScript ignores them. `Named` and `Proof` keep their names in invariant function slots. `Proof` also keeps its kind invariant when TypeScript falls back to structural comparison. `Named` has a private nominal marker so object spread loses its identity. |
| Names mixing via subtyping | No subtyping | `in out` variance annotations (TS 4.7+) make `N` invariant; `never` and `unknown` are rejected. |
| Names escaping their scope | Rank-2 type error | Compile error at the callback (TS 5.6+): the result would have to be `Named<unknown, _>`, which invariance forbids, and the escaped value is typed `unknown`. TS 5.4/5.5 catch a returned `Named` or proof too, but let a name wrapped in an object leak as an unbound type parameter; it stays distinct, so a mixup is still an error, just a later and less readable one. gdp-ts requires TS 5.4 or newer. |
| Existential types | `forall` in a data constructor | A callback (`withOrg(k)`). |
| Runtime | Erased | A frozen `{ kind }` per proof kind, a frozen `{ value }` per named value. |
| Stale proofs | Same | A proof says the fact held when it was checked. Keep proofs request-scoped and use transactions or constraints where a race matters. The callback rejects direct name/proof escapes; it does not enforce lifetime for a closure capturing them. |
| The checks themselves | Still code you wrote | Still code you wrote. Test the trusted modules; they are small. |

## Preserve named wrappers

Pass the original `Named` wrapper to checkers and protected functions. A spread
copy loses the private nominal marker, so `{ ...project, value: otherId }` cannot
reuse evidence about `project`. Spreading a proof with additional evidence is
supported, but replacing its `kind` cannot promote it to another policy.

This is a guard against ordinary object spread, not complete protection from
TypeScript's structural typing. `Object.assign` returns an intersection of its
input types, which can retain the original nominal identity even when the
runtime value is replaced:

```ts
name("allowed", "denied", (allowed, denied) => {
  // Compiles: the helper's return type still includes typeof allowed.
  const substituted = Object.assign({}, allowed, { value: denied.value });
  // substituted.value is "denied", despite retaining the name of allowed.
});
```

Do not use `Object.assign` or other intersection-returning helpers to copy or
rewrite named wrappers. Assertions, inferred `any`, and incorrectly typed
helpers can also bypass these contracts. The lint preset is a syntactic aid,
not a complete soundness checker.

The runtime wrapper is frozen only at the top level. If its `.value` is mutable,
later changes to that payload are not prevented. Prefer immutable identifiers
and keep checked state fresh through the appropriate transaction or constraint.

It also does not decide policy for you. Whether Viewers should see that a
project is password protected is a product question. With preserved named
wrappers, these signatures connect your checked facts to the values used by
protected functions; they do not validate the policy or its runtime freshness.
