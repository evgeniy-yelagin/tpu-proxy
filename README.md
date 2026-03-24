# TPU Transaction Proxy Patch

## Overview

This patch adds the ability to proxy all incoming non-vote TPU transactions to external services via UDP fire-and-forget datagrams. Supports multiple target addresses and hot-reload via admin RPC without validator restart.

## Releases


| Patch                     | Versions                       |
| ------------------------- | ------------------------------ |
| `tpu-proxy-v3.1.8.patch`  | `v3.1.7-jito`, `v3.1.8-jito`   |
| `tpu-proxy-v3.1.9.patch`  | `v3.1.9-jito`                  |
| `tpu-proxy-v3.1.11.patch` | `v3.1.10-jito`, `v3.1.11-jito` |


## Applying the patch

```bash
# git clone https://github.com/jito-foundation/jito-solana.git
cd jito-solana
git checkout tags/v3.1.11-jito   # or your target version

git am < tpu-proxy-v3.1.11.patch # use the matching patch
```

## Usage

**Start validator with proxying enabled:**

```bash
agave-validator ... --tpu-proxy-address 1.2.3.4:9000 --tpu-proxy-address 5.6.7.8:9001
```

Comma-separated format is also supported:

```bash
agave-validator ... --tpu-proxy-address 1.2.3.4:9000,5.6.7.8:9001
```

**Hot-reload addresses at runtime (no restart required):**

```bash
agave-validator --ledger /path/to/ledger set-tpu-proxy-addresses --tpu-proxy-address "1.2.3.4:9000,5.6.7.8:9001"
```

**Disable proxying at runtime:**

```bash
agave-validator --ledger /path/to/ledger set-tpu-proxy-addresses --tpu-proxy-address ""
```

## Receiving side

Each UDP datagram contains raw bytes of a single serialized `VersionedTransaction` (up to 1232 bytes, fits within standard MTU). Transactions are sent as-is with no framing or headers.

## Scope

The proxy covers transactions arriving via the **QUIC TPU** path (direct client submissions). Transactions routed through BAM or Relayer bypass this path and are not proxied.