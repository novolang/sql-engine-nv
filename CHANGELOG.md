# Changelog

Newest first.  Below `1.0.0` a breaking change bumps the **minor**
number and a compatible one the **patch**; see [Version numbers in the
Orbit package registry](https://novo-lang.org/docs/registry/semver.html).

## 0.0.3 — 2026-09-10

- **Toolchain floor is 0.8.9**: the bodies and signatures use what 0.8.9 added (`todo()`, a bound effect parameter, the four layers), and the manifest says so instead of letting an older toolchain fail on an undefined function.  No signature changed.

## 0.0.2 — 2026-09-09

- **Dependencies are registry ranges**, not paths: the interface release 0.0.1 shipped a manifest whose dependencies pointed at sibling directories that exist only in the monorepo, so a consumer resolved the closure and then could not load the dependency.  No signature changed.

## 0.0.1 — 2026-09-09

- First interface release: every public signature and effect row, every body a `todo()`.  **NOT IMPLEMENTED — interface only.**
