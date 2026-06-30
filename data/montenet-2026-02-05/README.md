# MonteNet field captures — 2026-02-05

Five `lime-report` captures from five LibreRouter v1 nodes on the MonteNet community network in
Molinari, Córdoba. These are the primary field evidence cited in §1.4 and §3.4 of the report.

| File | Node |
|---|---|
| `report-balcon.txt` | balcon |
| `report-e-bob.txt` | e-bob |
| `report-jime.txt` | jime |
| `report-loopin.txt` | loopin |
| `report-tronco.txt` | tronco |

## Sanitization applied

These are **sanitized** versions of the raw captures, safe for public release. They are produced by a
deterministic, salted sanitizer. MAC addresses are low-sensitivity identifiers (they are broadcast over the
air), pseudonymized here as a hygiene measure. The sanitizer script and its salt are **not** part of this
public bundle — publication is the git-tracked tree only, and the tooling lives outside it.

- **MAC addresses** — every hardware MAC is replaced with a consistent pseudonym in the
  locally-administered `02:*` range, in **both** representations it appears in: colon form
  (`xx:xx:xx:xx:xx:xx`) and IPv6 link-local EUI-64 form (`fe80::…ff:fe…`, which encodes the MAC).
  The mapping is deterministic (a salted SHA-256), so the **same physical device maps to the same
  pseudonym across all five files and across both representations** — network topology and neighbour
  relationships are preserved, while the real hardware identifiers are removed from this dataset.
- **SSIDs (network names) are kept.** The MonteNet SSIDs are public community-node network names in
  Molinari, broadcast over the air; they are not secrets (data-owner decision, 2026-06-28).
- **Encryption-mode strings** (`psk-mixed`) are kept — they are configuration modes, not secrets.
- The raw captures contained **no passwords or pre-shared keys**; any `key`/`password` option value
  is redacted as a precaution regardless.
- Internal addresses (`10.130.x.x`, IPv6 link-locals) are preserved as non-routable network detail;
  the MAC-derived bits of the EUI-64 link-locals are pseudonymized as above.

**Verified (2026-06-30):** no real (non-`02:*`) MAC address remains in these files in either colon or
`fe80::` EUI-64 form.

## Licence

CC-BY-4.0 (see `/LICENSES/CC-BY-4.0.txt` and `/.reuse/dep5`).
