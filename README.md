# turbo prune loses track of Yarn packageExtension edges, breaking `yarn install --immutable`

## Summary

`turbo prune <workspace>` computes which parts of the monorepo `yarn.lock`
are still needed by walking the dependency graph. That walk appears to be
based purely on the `dependencies:` fields that are **textually present**
in each package's own lockfile resolution block.

Yarn **packageExtensions** (built-in ones shipped with Yarn itself, or
custom ones from a project's `.yarnrc.yml`) inject an extra dependency into
a package's manifest *before* resolution — but the injected edge is **not**
persisted as text under the origin package's own lockfile entry. It only
shows up as an entirely separate, independently-merged descriptor block for
the target package.

Because of that, `turbo prune`'s graph walk can never "see" a
packageExtension-injected edge, in either direction:

- **Variant A** — pruning to a workspace whose *only* reason to need the
  extension's target package is the (invisible) extension edge: the whole
  target package is dropped from the pruned lockfile, even though it is
  still genuinely required at install time.
- **Variant B** — pruning to a workspace that needs the target package for
  an unrelated, real, textually-visible reason, while another (now pruned)
  workspace was the one whose extension had injected an extra range for
  the same package: the merged lockfile descriptor header is kept
  byte-for-byte unchanged, including the now-orphaned range from the
  removed workspace.

In both variants, a plain `yarn install` afterwards would rewrite the
lockfile entry (to add the missing package, or to narrow the stale merged
header) — and that rewrite is exactly what `yarn install --immutable`
forbids, so CI/Docker builds that rely on `turbo prune` output fail.

This reproduces with **no custom Yarn configuration at all** — only Yarn's
own built-in `packageExtensions` list (see
[`@yarnpkg/extensions`](https://github.com/yarnpkg/berry/blob/master/packages/yarnpkg-extensions/sources/index.ts)),
so it is not specific to any one project's setup.

## Setup

- `pkg-a` depends on `notistack@^3.0.0`.
  Yarn ships a built-in extension for it:
  `["notistack@^3.0.0", { dependencies: { csstype: "^3.0.10" } }]`
  (notistack does not declare a real dependency on `csstype` itself — its
  own lockfile entry only lists `clsx` and `goober`).
- `pkg-b` depends on `csstype@^3.0.2` directly, for an unrelated, real reason.
- Because both ranges resolve to the same `csstype` version, the full
  monorepo `yarn.lock` merges them into a single descriptor block:

  ```yaml
  "csstype@npm:^3.0.10, csstype@npm:^3.0.2":
    version: 3.2.3
    resolution: "csstype@npm:3.2.3"
  ```

  ...while `notistack`'s own entry shows no trace of that `^3.0.10` edge:

  ```yaml
  "notistack@npm:^3.0.0":
    version: 3.0.2
    resolution: "notistack@npm:3.0.2"
    dependencies:
      clsx: "npm:^1.1.0"
      goober: "npm:^2.0.33"
    # csstype is applied live via the packageExtension, but never written here
  ```

## Variant A: pruning to the workspace that only needs the package via the extension

```bash
yarn install
npx turbo prune pkg-a
cd out
yarn install --immutable
```

`out/yarn.lock` no longer contains **any** `csstype` resolution block at
all (`pkg-b`, the only textually-visible consumer, was pruned away, and
`notistack`'s extension-injected need for `csstype` is invisible to the
walk). Result:

```
➤ YN0000: ┌ Post-resolution validation
➤ YN0002: │ pkg-a@workspace:packages/pkg-a doesn't provide react (pe8a6ee), requested by notistack.
➤ YN0002: │ pkg-a@workspace:packages/pkg-a doesn't provide react-dom (p456600), requested by notistack.
➤ YN0000: │ @@ -53,8 +53,14 @@
➤ YN0028: │ +"csstype@npm:^3.0.10":
➤ YN0028: │ +  version: 3.2.3
➤ YN0028: │ +  resolution: "csstype@npm:3.2.3"
➤ YN0028: │ +  languageName: node
➤ YN0028: │ +  linkType: hard
➤ YN0028: │ +
➤ YN0028: │ The lockfile would have been modified by this install, which is explicitly forbidden.
➤ YN0000: └ Completed
➤ YN0000: · Failed with errors in 0s 138ms
```

(The `react`/`react-dom` peer warnings are pre-existing and unrelated to
the bug — `notistack` peer-depends on React and neither the toy repo nor
the pruned output provides it.)

## Variant B: pruning to the workspace that needs the package for a real, unrelated reason

```bash
yarn install
npx turbo prune pkg-b
cd out
yarn install --immutable
```

`out/yarn.lock` still contains the full merged header from the original
lockfile, unnarrowed, even though `notistack` (and with it, the only
consumer of the `^3.0.10` half) is completely gone from `out/package.json`
and `out/yarn.lock`:

```yaml
"csstype@npm:^3.0.10, csstype@npm:^3.0.2":   # ^3.0.10 is now orphaned
  version: 3.2.3
  resolution: "csstype@npm:3.2.3"
```

Result:

```
➤ YN0000: ┌ Post-resolution validation
➤ YN0000: │ @@ -46,9 +46,9 @@
➤ YN0028: │ -"csstype@npm:^3.0.10, csstype@npm:^3.0.2":
➤ YN0028: │ +"csstype@npm:^3.0.2":
➤ YN0000: │    version: 3.2.3
➤ YN0000: │    resolution: "csstype@npm:3.2.3"
➤ YN0028: │ The lockfile would have been modified by this install, which is explicitly forbidden.
➤ YN0000: └ Completed
➤ YN0000: · Failed with errors in 0s 35ms
```

## Expected result

`turbo prune` should recompute the reachable descriptor set for every
retained package from the *actual* resolved dependency graph (the same one
`yarn why` / `yarn install` see, packageExtensions included), not just from
the literal `dependencies:` text of each lockfile entry. That would mean:

- Variant A: `csstype` stays in the pruned lockfile, because it is still
  really needed.
- Variant B: the merged header narrows to `csstype@npm:^3.0.2` only,
  because `^3.0.10` is no longer requested by anything reachable.

Either way, `yarn install --immutable` should then succeed on the pruned
output, matching what a plain `yarn install` would produce from scratch.

## Why this matters beyond this toy example

This is not specific to `notistack`/`csstype`. It reproduces with:

- Any Yarn built-in `packageExtension`
  (see the full list in
  [`@yarnpkg/extensions`](https://github.com/yarnpkg/berry/blob/master/packages/yarnpkg-extensions/sources/index.ts)).
- Custom `packageExtensions` configured in a project's own `.yarnrc.yml`
  (e.g. we independently hit this with a `ssh2@* -> node-gyp@*` extension:
  packages that depend on `ssh2` pull in `node-gyp@*`; pruning to a
  workspace that no longer depends on `ssh2` can hit either variant above,
  depending on whether another retained package also needs `node-gyp` for
  a real, unrelated reason).

## Environment

- `yarn`: 4.17.0
- `turbo`: 2.10.4
- `node`: v24.18.0
- OS: Windows 11
