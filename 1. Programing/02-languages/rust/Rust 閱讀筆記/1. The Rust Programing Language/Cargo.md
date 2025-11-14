# Workspace
---
*  library crate continues to get bigger and you want to **split your package into multiple library crates**.
* _workspace_ is a set of packages that share the same _Cargo.lock_ and _target_ output directory.

#### Create Workspace
* the _Cargo.toml_ won't have `[package]` section, instead with `[workspace]` section.
```toml
[workspace]
members = [
  "crates/globset",
  "crates/grep",
  "crates/cli",
  "crates/matcher",
  "crates/pcre2",
  "crates/printer",
  "crates/regex",
  "crates/searcher",
  "crates/ignore",
]
```

