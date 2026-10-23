[![tests](https://github.com/qlrd/krux-integration-tests/actions/workflows/tests.yml/badge.svg)](https://github.com/qlrd/krux-integration-tests/actions/workflows/tests.yml)
![Python Version from PEP 621 TOML](https://img.shields.io/python/required-version-toml?tomlFilePath=https%3A%2F%2Fraw.githubusercontent.com%2Fqlrd%2Fkrux-integration-tests%2Fmain%2Fpyproject.toml)

# krux-integration-tests

Integration tests for the [Krux](https://github.com/selfcustody/krux)
ecosystem.

## Summary

Integration tests are a important part of any project and maybe sits between
basic tests and fuzz tests. This one is to check if krux correctly (we expect so!)
with `bitcoin-core` (with or without `electrs` -- testing a basic setup of a
ideal `sparrow`), `florestad`, `utreexod` and `lianad` (TODO: maybe SP on regtest too?).

## Structure

Tests lives under `tests/integration/` drive real Bitcoin daemons
(`bitcoind`, `electrs` -- TODO, ...) on **regtest**.

Implementations are built from source, cached and handed to the tests as
fixtures by the [`bornal`](https://github.com/qlrd/bornal) pytest plugin **unless**
you provide your own bitcoin implementations.

## Setup

- Build deps for bitcoin-core and electrs
- python 3.12 (`.python-version`)
- [`uv`](https://docs.astral.sh/uv/)

### Installing

```sh
# Krux uses a lot of submodules, accelerate it with `--jobs`
git clone --recurse-submodules --jobs 8 <this-repo>
# or, in an existing clone:
git submodule update --init --recursive --jobs 8
uv sync --frozen
```

Optional, for the C `secp256k1` backend embit's tests expect
(pure-Python fallback otherwise):

```sh
cd vendor/krux
uv run poe secp256k1-build
uv run poe secp256k1-check
```

## Running tests

If you wanna to use your own daemon, add it to `BORNAL_TEMP_DIR`. First run
builds the backends into `BORNAL_TEMP_DIR` (default `$HOME/.cache/bornal/krux-integration-tests`,
see `bornal/paths.py`).

The compilation was made to separate from your own bitcoin-core binaries, logs
and data folders.

```sh
uv run poe tests                          # build (or reuse) bitcoind with wallet
uv run poe tests -B 30.2 -j 8             # pin release and run compile parallelism
uv run poe test-base -t p2pkh             # run only tests/integration/test_p2pkh.py
uv run poe test-base -t p2wpkh -B 31.1    # native SegWit review/sign/export round trip
uv run poe test-base -t p2pkh -- -k psbt  # anything after `--` goes to pytest
```

The plain `pytest` form still works and is what the tasks expand to:

```sh
uv run pytest --build-bitcoin latest --wallet tests     # it build bitcoind with wallet then run
uv run pytest tests                                     # reuse the cached build
uv run pytest --build-bitcoin 30.2 --nproc 8 tests      # pin release and compile parallelism
```

Flags such as `--force-build` / `--preserve-data` are documented in
`bornal`'s README.

### Native SegWit round trip

`tests/integration/test_p2wpkh.py` extends the lab described in
[Krux #968](https://github.com/selfcustody/krux/issues/968). It funds a BIP84
watch-only wallet, checks Krux's recipient/change/fee review against Core's
transaction data using integer satoshis, signs and exports the PSBT through
Krux, and asks Core to finalize, accept and confirm it. An unsigned control and
an output-tampering control must fail Core's script checks.

Run the whole module: its ordered tests share state, just like the legacy
module. `uv run poe tests -B 31.1` runs both modules. The existing CI's
`test-base` invocation also discovers both. bornal manages per-module node
directories, startup/shutdown timeouts and diagnostic logs.

The Krux revision is pinned by the `vendor/krux` submodule; inspect it with
`git submodule status vendor/krux`. bornal is pinned in `uv.lock`. Specify a Core
release with `-B` to reproduce a run, and retain the Core version and Krux SHA
when sharing results. These tests exercise host-side review text and signing
with the real secp256k1/uUR modules, not camera transport or physical displays.

### Debug tests

```sh
uv run poe test-base -t p2pkh -d                          # live output for the whole module
uv run poe test-base -t p2pkh -d -- -k test_004           # one test
uv run poe test-base -t p2pkh -d -- -v                    # -v also turns on bornal's RPC log
uv run poe test-base -t p2pkh -d -- --log-cli-level=DEBUG # everything the loggers emit
```

## Format and lint

```sh
uv run poe format        # black over tests/
uv run poe format -c     # check only (what CI / reviewers run)
uv run poe lint          # pylint over tests/ (krux.* resolves via [tool.poe.env])
```
