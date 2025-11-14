### Packages and Crates

* Package > Crates > Moduels
###### example
```text
.
├── Cargo.toml
├── build.rs
└── src
    ├── bin
    │   └── alt.rs
    ├── lib.rs
    ├── main.rs
    └── util.rs
```
* There is Cargo.toml, so it is package. 
* src/main.rs is binary crate root.
* src/lib.rs is liberary crate root.
* src/bin/util.rs is another binary crate.
* build.rs is a building script.

### Modules to Control Scope and Privacy
