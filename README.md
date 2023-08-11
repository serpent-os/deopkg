# deopkg

A [PackageKit](https://github.com/PackageKit/PackageKit) plugin for [Solus](https://getsol.us)
encapsulating the `eopkg` package manager. This effort exists to assist in decoupling Solus from
`python2` and unlocking the path towards Serpent based tooling.

eopkg carries a lot of legacy from its predecessor, PiSi, such as Python2, piksimel, etc. Basic
operations such as enumeration of the available candidates via `PackageDB` take approximately
5-10s to complete, leaking approx. 300MiB per iteration in our tests.

The `deopkg` plugin aims to bridge the gap by alleviating performance and stability issues, whilst
allowing Solus to put `eopkg` into a sealed unit.

### Technical notes

This plugin is implemented in D Lang, exposing a C ABI via [packagekit-d](https://github.com/packagekit-d).
All Pythonic calls are implemented directly in `libpython2.7` using the [pyd](https://github.com/ariovistus/pyd) embedded module.

Due to the extreme overhead of interacting with the PiSi internals, a simplified RPC model is employed:

 - A `socketpair` is constructed
 - Caller invokes `fork()` `wait()` via `runForked` helper API
 - Fork child initialises `libpython2.7`, runs Pythonic functions, and **yields** via Generator.
 - Fork child serializes all results over `socket` (currently using [asdf](http://asdf.libmir.org/))
 - Main process pulls artefacts from `socket` and handles, using `@nogc` strategy where needed.

Each "fat" operation is handled, automatically, via this forking architecture. Once the operation has completed,
the fork child is disposed of and memory is returned immediately to the kernel. Optimisations are made within `packagekit-d`
and this project to minimise any use of a garbage collector and ensure minimal footprint over time.

Note that caching is employed for package lists using an `sqlite3` database to minimise the lookup cost for `resolve` and `get-packages`.