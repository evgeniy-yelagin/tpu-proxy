# TPU Transaction Proxy Patch

## Overview

This patch adds the ability to proxy all incoming TPU transactions to external services via UDP fire-and-forget datagrams. Transactions are forwarded **after signature verification**, so only valid, non-discarded packets are proxied — significantly reducing spam compared to raw pre-sigverify forwarding. Supports multiple target addresses, hostnames (DNS resolution), and hot-reload via admin RPC without validator restart.

## Releases


| Patch | Versions |
|-------|----------|
| `tpu-proxy-v3.1.7.patch` | `v3.1.7-jito`, `v3.1.8-jito` |
| `tpu-proxy-v3.1.9.patch` | `v3.1.9-jito`, `v3.1.10-jito`, `v3.1.11-jito` |
| `tpu-proxy-v3.1.12.patch` | `v3.1.12-jito`, `v3.1.13-jito` |
| `tpu-proxy-v3.1.14.patch` | `v3.1.14-jito` |
| `tpu-proxy-v4.0.0.patch` | `v4.0.0-jito` |


## Applying the patch

```bash
# git clone https://github.com/jito-foundation/jito-solana.git
cd jito-solana
git checkout tags/v4.0.0-jito         # or your target version
git apply tpu-proxy-v4.0.0.patch      # use the matching patch
```

## Usage

**Start validator with proxying enabled:**

```bash
agave-validator ... --tpu-proxy-address 1.2.3.4:9000 --tpu-proxy-address 5.6.7.8:9001
```

Both IP addresses and hostnames are supported (DNS is resolved at startup):

```bash
agave-validator ... --tpu-proxy-address stream.example.com:9000
```

Comma-separated format also works:

```bash
agave-validator ... --tpu-proxy-address 1.2.3.4:9000,stream.example.com:9001
```

**Hot-reload addresses at runtime (no restart required):**

```bash
agave-validator --ledger /path/to/ledger set-tpu-proxy-addresses \
  --tpu-proxy-address "1.2.3.4:9000,stream.example.com:9001"
```

**Disable proxying at runtime:**

```bash
agave-validator --ledger /path/to/ledger set-tpu-proxy-addresses --tpu-proxy-address ""
```

## UDP datagram format

Each UDP datagram has the following layout:

```
[ transaction bytes (up to 1232 bytes) ][ validator identity pubkey (32 bytes) ]
```

- **Transaction bytes** -- raw serialized `VersionedTransaction`, same bytes as received on QUIC TPU.
- **Validator identity** -- 32-byte Ed25519 public key of the validator that forwarded the packet. Allows the receiving side to identify the source validator.

Total datagram size: up to 1264 bytes (fits within standard MTU).

## Scope

The proxy covers transactions arriving via the **QUIC TPU** path (direct client submissions). Transactions routed through BAM or Relayer bypass this path and are not proxied.

### Filtering applied before proxy

- Ed25519 signature verification (invalid signatures → discarded)
- Duplicate detection (dedup within sigverify window)
- Packet size validation (< 134 bytes or > 1232 bytes → skipped)
- `discard()` flag check (packets marked discard by earlier stages → skipped)