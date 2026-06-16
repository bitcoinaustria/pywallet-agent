# AGENTS.md

Guidance for AI agents (and humans) working in this repository.

## What this is

`pywallet` is a single-file tool for inspecting, dumping, recovering and
manipulating Bitcoin (and related) `wallet.dat` files and keys. This is a
fork of jackjack's pywallet (itself forked from Joric's), being modernized
for current Python.

The entire program is one file: **`pywallet.py`** (~4200 lines). There is no
package, no build step, and no `setup.py`. `README` is the user-facing doc.

## Hard constraints — read before changing anything

1. **It must stay offline.** The core operations (dumping a wallet, key
   inspection, BIP32/BIP39 derivation, key recovery) run with no network
   access. Do **not** add network calls, telemetry, "phone home", or
   auto-update behavior to any default code path. The existing online
   features (`--balance`, `--dumpwithbalance`, the embedded block-explorer
   helpers, `update_pyw`/`retrieve_last_pywallet_md5`) are strictly opt-in
   and must remain that way. `update_pyw`/`retrieve_last_pywallet_md5` are
   defined but never called from `__main__` — keep it so.

2. **Keep changes absolutely minimal.** Prefer the smallest surgical diff
   that solves the problem. Do not reformat, refactor, rename, or "tidy"
   surrounding code. Do not split the file into a package. Match the
   existing style exactly.

3. **It handles private keys.** This code reads secret key material. Never
   add anything that writes keys/seeds/passphrases to disk, logs, or the
   network. Be conservative; when in doubt, ask.

## Modernization goal

Runs on current Python 3 (3.9+, tested on 3.14) **without MacPorts or other
legacy tooling**. The code is now **Python 3 only**: the former Python 2/3
compatibility shims (`raw_input`, `xrange`, `long`, `unicode`, `reduce`, and
the `PY3` flag) have been removed in favour of the Python 3 builtins. Do not
reintroduce Python 2 shims.

Dependency notes:
- Core key/BIP32/BIP39 logic uses pywallet's **embedded** elliptic curve
  implementation and needs **no external packages**. Offline key inspection
  (`--info`, `--random_key`) works with zero deps installed.
- Key recovery (`--recover`) *scans* with no deps, but it always writes the
  recovered keys into a new `wallet.dat` via `create_new_wallet()`, which needs
  a Berkeley DB binding — so end-to-end `--recover` requires one too.
- Reading/writing a real `wallet.dat` needs a Berkeley DB binding. Import
  order tried in code: `bsddb` (legacy), then `bsddb3`, then `berkeleydb`
  (the maintained successor — install with `pip install berkeleydb`).
- Signing/verifying messages needs `ecdsa` (`pip install ecdsa`).
- Missing optional deps are recorded in the `missing_dep` list and degrade
  gracefully; do not make them hard import errors.

Install: use a virtualenv (modern system Python is PEP 668
"externally-managed", so bare `pip install` fails):

```sh
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt   # both deps are optional; see the file
```

The online features (`--balance`, `--whitepaper`) call `urllib.urlopen` (the
old spelling). A small wrapper aliases it to `urllib.request.urlopen` and
decodes responses to text, so every call site works unchanged on Python 3.
This is purely an online-path fix — offline operations never reach `urllib`.

## Running

```sh
python3 pywallet.py --help                     # full, current option list
python3 pywallet.py --tests                    # offline self-tests (no deps)
python3 pywallet.py --info --importprivkey 1   # inspect a key, fully offline
python3 pywallet.py --dumpwallet --wallet /path/to/wallet.dat  # needs bsddb
```

Note: the key value for `--info` is supplied via `--importprivkey` (its
option `dest` is `key`); `--info` itself (`dest=keyinfo`) just selects display
mode. This is existing CLI behavior — don't change it without being asked.

## Testing

Self-tests live in the `TestPywallet` class and run via:

```sh
python3 pywallet.py --tests
```

They cover private-key parsing, BIP32/BIP39 test vectors, and the Python 3
online str/bytes handling (the urllib decode shim, `balance()`, `md5_2`; the
network is mocked, so no real requests). They require **no** external
dependencies or network. Note: the recovery test is currently a placeholder.
Run them after any change to the crypto/derivation or online paths. Expected
result: `OK`.

## Code style

- **Tabs** for indentation (not spaces). Match the file.
- Python 3 only (3.9+). Use Python 3 builtins directly; do not add back
  Python 2 compatibility shims or a `PY3` flag.
- No new third-party dependencies unless unavoidable, and only as optional
  (caught) imports that append to `missing_dep`.

## Before you finish

- Run `python3 pywallet.py --tests` and confirm `OK`.
- Run `python3 pywallet.py --help` and confirm it still parses.
- Re-read your diff: is it the minimal change? Did you avoid touching
  unrelated lines? Did you keep it offline?
