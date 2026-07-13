# turbo prune drops a Yarn packageExtension range, breaking `yarn install --immutable`

## Summary

`turbo prune <workspace>` removes a workspace's dependencies from the pruned
`yarn.lock` correctly, but when one of the removed dependencies triggered a
Yarn **packageExtension** (either a built-in one shipped with Yarn, or a
custom one from `.yarnrc.yml`), and the extension's injected dependency range
happens to be merged in the lockfile with another *unrelated, still-needed*
range for the same package, `turbo prune` keeps the merged descriptor header
unchanged instead of narrowing it to only the ranges that are still reachable
from the pruned workspace.

Running `yarn install --immutable` afterwards then fails, because a normal
(non-immutable) install would rewrite that lockfile entry to drop the
now-unreachable range — which is exactly what `--immutable` forbids.

This reproduces with **no custom Yarn configuration at all** — only Yarn's
own built-in `packageExtensions` list (see
[`@yarnpkg/extensions`](https://github.com/yarnpkg/berry/blob/master/packages/yarnpkg-extensions/sources/index.ts)),
so it is not specific to any one project's setup.

## Setup

- `pkg-a` depends on `notistack@^3.0.0`.
  Yarn ships a built-in extension for it:
  `["notistack@^3.0.0", { dependencies: { csstype: "^3.0.10" } }]`
  (notistack does not declare a real dependency on `csstype` itself).
- `pkg-b` depends on `csstype@^3.0.2` directly, for an unrelated, real reason.
- Because both ranges resolve to the same `csstype` version, the full
  monorepo `yarn.lock` merges them into a single descriptor block:

  ```yaml
  "csstype@npm:^3.0.10, csstype@npm:^3.0.2":
    version: 3.2.3
    resolution: "csstype@npm:3.2.3"
  ```

## Reproduction steps

```bash
yarn install
cat yarn.lock | grep -A4 '^"csstype@npm'
# "csstype@npm:^3.0.10, csstype@npm:^3.0.2":
#   version: 3.2.3
#   ...

npx turbo prune pkg-b

cat out/yarn.lock | grep -A4 '^"csstype@npm'
# "csstype@npm:^3.0.10, csstype@npm:^3.0.2":   <-- unchanged, even though
#   version: 3.2.3                                 notistack (the only
#   ...                                             consumer of ^3.0.10) is
#                                                    completely gone from
#                                                    out/package.json and
#                                                    out/yarn.lock

cd out
yarn install --immutable
```

## Actual result

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

`yarn install --immutable` fails in CI/Docker builds that rely on
`turbo prune` output, even though the pruned workspace's own dependency
graph is entirely self-consistent — the only problem is the leftover,
unreachable half of a merged descriptor header.

## Expected result

`turbo prune` should recompute the reachable descriptor set for every
retained package and narrow merged lockfile headers accordingly (dropping
`csstype@npm:^3.0.10` here), the same way a plain `yarn install` would, so
that `yarn install --immutable` succeeds on the pruned output.

## Why this matters beyond this toy example

This is not specific to `notistack`/`csstype`. It reproduces with:

- Any Yarn built-in `packageExtension`
  (see the full list in
  [`@yarnpkg/extensions`](https://github.com/yarnpkg/berry/blob/master/packages/yarnpkg-extensions/sources/index.ts)),
  as long as its injected range ends up merged with another still-needed
  range for the same package.
- Custom `packageExtensions` configured in a project's own `.yarnrc.yml`
  (e.g. we independently hit this with a `ssh2@* -> node-gyp@*` extension:
  packages that depend on `ssh2` pull in `node-gyp@*`; when the pruned
  workspace doesn't depend on `ssh2` anymore but another retained package
  still needs `node-gyp` for a real reason, the same merged-header problem
  occurs).

## Environment

- `yarn`: 4.17.0
- `turbo`: 2.10.4
- `node`: v24.18.0
- OS: Windows 11
