<!--
SPDX-FileCopyrightText: 2026 Gioacchino Mazzurco / AlterMundi
SPDX-License-Identifier: CC-BY-SA-4.0
-->
# Shared State — merge strategy and TTL convergence

> **Author:** Gioacchino Mazzurco (`G10h4ck`), AlterMundi.
> **Status:** technical working note. Supporting document for §1.4 (Shared State) of the ARDC 2024
> report. Originally drafted on CryptPad; this is the stable, citable snapshot.
>
> *Sanitization: the illustrative MAC address is pseudonymized and the two community-member node names
> in the office-bench example are generalized to `node-A` / `node-B`. The MonteNet fleet node names
> (jime, balcon, tronco, e-bob, nodo-suri) are retained — they match the published field dataset.*

---

In a network with four LibreRouters in an office but with a distance set to 1000, the latencies and
retransmission times are not the best. This is a very hard environment for testing Shared State. After
some seconds of sync the daemons start reporting TTL discrepancies.

For a sample bat-hosts Key `02:59:d8:13:06:f3` — Author `node-A` has lower TTL (1996) than Reader
`node-B` (1999); in this setup the TTL error is around 3.

This means that node `node-B` has TTL=1999 for an entry that `node-A` authored, but `node-A` itself only
has TTL=1996 for the same entry. This is a problem because Shared State relies on the biggest TTL to
assume information as newer. But TTL is handled locally by each peer. TTL was improved to not depend on
the time it was called but to decrease taking into account the actual time passed between calls. This
fixed some problems and now the differences remain constant in time. What this TTL is not taking into
account is the time it took to deliver the sync until the remote starts "bleaching". This is critical for
faster messaging and faster convergence.

Example

    Node "A" as node-A (Author)
    Node "B" as node-B

    A->>A: Insert entry (TTL=2000)
    A->>B: Sync entry (TTL=2000) (B stores TTL=2000)

    4 seconds pass until end of sync.
    A->>A: bleach() (TTL=1996)

    B->>B: bleach() delayed (only 1 second)
    Note over B: node-B still has TTL=1999

In MonteNet Libre something similar happened:

```bash
root@jime:~# shared-state-async sync wifi_links_info
D 1769611667.838 std::task<int> SharedState::merge(...) wifi_links_info got 5 significative changes out of 5 input slice size: 5 state size: 5
D 1769611667.851 ReturnType AwaitableSyscall<...>::await_resume() [...] syscall failed on resume mReturnValue: -1 error: 146 Connection refused category: generic
I 1769611667.851 std::task<void> SharedStateCli::sync(...) Failure synchronizing with peer: ipv6://[fe80::df:5aff:fef6:6eb3%25br-lan]:3490 error: 146 Connection refused category: generic
D 1769611667.878 std::task<int> SharedState::merge(...) wifi_links_info got 0 significative changes out of 4 input slice size: 5 state size: 5
W 1769611667.920 std::task<int> SharedState::merge(...) Discarding received known entry: jime authored by this node with higher TTL from remote peer: ipv6://[fe80::11:e2ff:feb3:de83%25wlan1-mesh]:3490 is remote peer ill?
```

As you can see there is a message *"Discarding received known entry: jime authored by this node with
higher TTL from remote peer ... is remote peer ill?"*

The remote peer is `tronco`. In `tronco`, all of `jime`'s keys have higher TTL values; from the point of
view of another node, `tronco`'s word is stronger than `jime`'s even though `jime` was the author. And
all the nodes that are behind `tronco` won't be updated.

Comparing simultaneously-taken snapshots of the dumps of 4 nodes:

    jime:       10.130.115.142   LiMe final-release development (rev. 99d02c2 20260117_0538)
    balcon:     10.130.73.222    LiMe final-release development (rev. 99d02c2 20260117_0538)
    tronco:     10.130.73.46     LiMe final-release development (rev. 99d02c2 20260117_0538)
    nodo-suri:  10.130.113.154   LiMe final-release development (rev. 99d02c2 20260117_0538)
    e-bob:      10.130.39.194    LiMe final-release development (rev. 99d02c2 20260108_0356)

    WIFI_LINKS_INFO
    ============================================================
    Key            jime     balcon    tronco    e-Bob    Max diff
    ----------------------------------------------------------------
    balcon         1213     1189      1214      1214     25 ⚠️
    e-Bob          1137     1137      1138      1111     27 ⚠️
    jime           1134     1135      1156      1156     22 ⚠️
    nodo-suri      1209     1210      1210      1210      1
    tronco         1190     1191      1190      1190      1

In this case `jime`'s links have 1134 TTL in the local node but `tronco` and `e-bob` have higher values.
This means that `jime` won't be able to send updates for at least 22 seconds.
