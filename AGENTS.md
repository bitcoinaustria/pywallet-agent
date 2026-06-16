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

Make it run cleanly on current Python 3 (3.8+, tested on 3.14) **without
MacPorts or other legacy tooling**, while keeping the diff tiny. The script
already carries `PY3` compatibility shims near the top (`raw_input`, `xrange`,
`long`, `unicode`, `reduce`, etc.) — extend those patterns rather than
inventing new ones.

Dependency notes:
- Core key/BIP32/BIP39/recovery logic uses pywallet's **embedded** elliptic
  curve implementation and needs **no external packages**. Offline key
  inspection (`--info`, `--random_key`) works with zero deps installed.
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

The online features (`--balance`, `--whitepaper`) use `urllib.urlopen`, which
is Python-2 spelling and will `AttributeError` on Python 3 — left as-is on
purpose because they are out of scope for offline use. Do not "fix" them
unless explicitly asked; doing so does not affect offline operation.

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

They cover private-key parsing, BIP32 test vectors, BIP39 test vectors and
recovery, and require **no** external dependencies or network. Run them after
any change to the crypto/derivation paths. Expected result: `OK`.

## Code style

- **Tabs** for indentation (not spaces). Match the file.
- Python 2/3 compatible where the existing code is; gate 3-only code behind
  the `PY3` flag like the existing shims.
- No new third-party dependencies unless unavoidable, and only as optional
  (caught) imports that append to `missing_dep`.

## Before you finish

- Run `python3 pywallet.py --tests` and confirm `OK`.
- Run `python3 pywallet.py --help` and confirm it still parses.
- Re-read your diff: is it the minimal change? Did you avoid touching
  unrelated lines? Did you keep it offline?
