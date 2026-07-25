# kreeader-seeds

Carrier repo for **kreeader's signed peer seed lists** — the data layer of
the distributed tier's bootstrap ladder (kreeader-server `#42`, design in
`kreeader-server/components/distributed/seeds` and
`docs/distributed/README.md`).

A seed list only tells a fresh node *where to start looking* for peers. It
never grants trust in data: the consensus gate protects the content itself,
so a forged or stale list can at worst delay/eclipse a node, never fabricate
promotions. Signing (ed25519, minisign-style — **not** GPG) exists so that a
compromise of a carrier (this GitHub repo, a DNS zone) cannot silently steer
nodes, and `seq`+`expires` refuse replay of stale lists.

## Files

| File | What it is |
|------|------------|
| `seeds.json` | The current signed seed list — the exact bytes every carrier ships. |
| `seed_verify.pub` | The publisher's ed25519 **public** verify key (hex). Baked into the server binary / set as `[distributed] seeds_pubkey`. |

The matching **private** signing key is held only by the publisher (Andrew)
and is **never** committed here.

## Format

`seeds.json` is the signed envelope produced by `cmd/seedpub sign`:

```json
{
  "v": 1,
  "seq": 1,
  "expires": 1816485554,
  "addrs": ["/ip4/.../tcp/4001/p2p/12D3Koo..."],
  "sig": "base64-ed25519-signature"
}
```

- `v` — payload format version (currently `1`).
- `seq` — monotonic sequence number. A node refuses any list whose `seq` is
  not strictly greater than the newest it has already accepted (rollback
  defense). Every new publish MUST increment `seq`.
- `expires` — unix seconds; a node refuses an expired list.
- `addrs` — libp2p multiaddrs to bootstrap from (`/dns4/...` welcome). Each
  must carry the peer's `/p2p/<peer-id>` suffix.
- `sig` — ed25519 signature over the canonical JSON of `{v,seq,expires,addrs}`
  (field-declaration order — **do not reorder**).

The signed bytes are identical across every rung: this GitHub raw file, a DNS
TXT record (standard-base64 of the same bytes), a QR code, and the Settings
paste field.

## Carriers

- **GitHub raw (this repo)** — configure a node with:
  `[distributed] seeds_url = "https://raw.githubusercontent.com/assimilat/kreeader-seeds/master/seeds.json"`
- **DNS TXT** — paste the `dns txt:` base64 that `seedpub sign` prints into a
  TXT record; configure `[distributed] seeds_dns = "<record name>"`.
- **Embedded default** — commit a signed `seeds.json` into the server binary
  at `components/distributed/seeds/defaults/seeds.json`.
- **Manual** — `[distributed] seed_peers = [...]` (unsigned; the operator is
  the authority for their own box).

Every signature-verified rung is checked against `seeds_pubkey`
(`seed_verify.pub` here). With no pubkey configured, the signed rungs are
skipped entirely — an unsigned list is never trusted.

## Publishing a new list

From a checkout of `kreeader-server` (which provides the `seedpub` tool) with
the private signing key available:

```sh
go run ./cmd/seedpub sign \
    -key   /path/to/seed_signing.key \
    -seq   <strictly greater than the last published seq> \
    -expires 8760h \
    -addr  /ip4/<host>/tcp/4001/p2p/<peer-id> \
    [-addr ...] \
    -out   seeds.json

go run ./cmd/seedpub verify -pub seed_verify.pub seeds.json   # sanity check
git add seeds.json && git commit -m "seeds: publish seq <n>" && git push
```

The publisher keypair is generated once with `seedpub keygen -out <dir>`; the
public half becomes `seed_verify.pub` here and the server's `seeds_pubkey`,
the private half stays offline with the publisher.

## Verify key

```
92578a0a1be897b4f803020b10e7b5f06a3a613ee62e1dc154fed2b91c9c19bd
```
