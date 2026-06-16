---
name: pywallet
description: Use this skill to inspect, dump, recover, or manipulate Bitcoin (and related) wallet.dat files and private keys offline with pywallet.py. Covers dumping wallet keys to JSON, decrypting encrypted wallets with a passphrase, inspecting/converting a single key, BIP32 (xprv/xpub) and BIP39 mnemonic derivation, recovering keys from raw/corrupt data, and creating a shareable minimal encrypted copy. Applies whenever the user mentions wallet.dat, dumping or recovering bitcoin keys, xprv/xpub, BIP32/BIP39, or pywallet by name.
---

# pywallet

`pywallet.py` is a single self-contained script for working with Bitcoin
`wallet.dat` files and keys. It runs **fully offline** — never route any of
these workflows through a network service, and never write recovered keys,
seeds, or passphrases anywhere the user didn't ask for.

## Setup

Nothing is required for key inspection, BIP32/BIP39 derivation, and recovery —
these run with zero dependencies. Optional extras need a virtualenv (modern
system Python rejects bare `pip install` under PEP 668):

```sh
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt   # both lines are optional
```

- Read/write a real `wallet.dat`: `berkeleydb` (needs the Berkeley DB system
  lib — `brew install berkeley-db` on macOS, `apt install libdb-dev` on
  Debian). Legacy `bsddb3` also works as a fallback.
- Sign/verify messages: `ecdsa` (pure Python).

Confirm it works: `python3 pywallet.py --tests` should print `OK`.
Always check `python3 pywallet.py --help` for the authoritative option list.

## Core workflows

Dump a wallet to JSON (addresses + keys):
```sh
python3 pywallet.py --dumpwallet --wallet /path/to/wallet.dat
```

Dump only addresses:
```sh
python3 pywallet.py --dumpwallet --dumpformat addr --wallet /path/to/wallet.dat
```

Decrypt an encrypted wallet during dump:
```sh
python3 pywallet.py --dumpwallet --wallet /path/to/wallet.dat --passphrase 'THE_PASSPHRASE'
```

Inspect / convert a single key (WIF, hex, or '1'-style int), fully offline —
works with no dependencies. The key is passed via `--importprivkey`:
```sh
python3 pywallet.py --info --importprivkey 5HpHagT65TZzG1PH3CSu63k8DbpvD8s5ip4nEB3kEsreAnchuDf
```
Generate a random key (also offline, no deps; use it alone, not with --info):
`python3 pywallet.py --random_key`

Import a private key into a wallet:
```sh
python3 pywallet.py --importprivkey KEY --wallet /path/to/wallet.dat
```

Derive keys from a BIP32 xprv along a path:
```sh
python3 pywallet.py --dump_bip32 xprv9s21ZrQH143K... "m/0H/1-2/2H/2-4"
```

Find info about an address in a wallet:
```sh
python3 pywallet.py --find_address ADDR --wallet /path/to/wallet.dat
```

Create a shareable minimal encrypted copy (one unused key only, so others can
help brute-force a forgotten passphrase without exposing funds):
```sh
python3 pywallet.py --minimal_encrypted_copy --wallet /path/to/wallet.dat
```

## Networks

Default is Bitcoin mainnet. Use `--testnet`, `--namecoin`, `--eth`, or
`--otherversion` (a P2PKH prefix like `111`, or full
`name,p2pkh,p2sh,wif,segwithrp`) to target another network.

## Safety rules

- Treat every wallet path, key, mnemonic, and passphrase as a live secret.
- Do not paste secrets into commands that get logged, or echo them back
  unnecessarily.
- Keep everything offline. Online options (`--balance`, `--dumpwithbalance`)
  exist but should only be used on explicit request.
- Operate on copies of `wallet.dat`, never the user's only backup, when an
  operation writes (`--importprivkey`, `--minimal_encrypted_copy`).
