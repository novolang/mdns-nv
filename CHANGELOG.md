# Changelog

All notable changes to mdns-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-12

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `mdnsfault` — the faults, `[]` throughout.  A conflict is NOT in the
  enum: renaming is a state transition and not an error, and what is
  here is the case where fifteen renames did not help.
- `mdnstime` — every timing rule as integer arithmetic and the two bits
  RFC 6762 steals from the class field, `[]` throughout and
  `@tier(embedded)` on all of it.  The module names nothing else in the
  world, on purpose: it is the one this package's device probe can be
  built against.
- `mdnsrec` — dns-codec-nv's records read the way mDNS means them, `[]`
  throughout: the cache-flush bit and the unicast-response bit are the
  SAME bit with two meanings and get two pairs of functions, a goodbye
  is a TTL of zero, and a legacy query gets a different answer entirely.
- `mdnssd` — RFC 6763's names and the four records of a service, `[]`
  throughout: an instance name is a human-readable label and is never
  escaped, a TXT key with no value is not a key with an empty value,
  and PTR SRV TXT and A go in one packet because that is the difference
  between a list that appears and one that fills in.
- `mdnsclaim` — probing, announcing and conflict resolution, `[]`
  throughout with the clock an argument: `MdnsClaimWait(at_ms)` as the
  load-bearing variant, and the tiebreak as a lexicographic comparison
  over wire bytes rather than a first-come rule that cannot converge.
- `mdnsquery` — the cache and known-answer suppression, `[]`
  throughout: a bounded cache because anybody on the link can fill it,
  four refreshes rather than one, and the half-life rule that keeps a
  querier from suppressing the response that would have refreshed it.
- `mdnsresp` — answering and the four reasons not to, `[]` throughout:
  a plan rather than a message, `suppress` for the answer somebody else
  gave first, and unique against shared deciding the timing.
- `mdnssock` — the host half: `[net]` for the socket and the group,
  `[time]` for the ONE clock read in the package.
- `tests/embedded_probe.nv` — twenty-four checks over `mdnstime`, built
  for `--target=nrf52-qemu` by hand, because the audit's
  `core-embedded` row belongs to `core` packages and this one is `host`.
- API tests in `tests/mdnsclaim_tests.nv` and
  `tests/mdnsbrowse_tests.nv`, neither of which sleeps.  Red until the
  bodies land.

### Named as missing

**Multicast group membership, which this package cannot work without.**
`std.net`'s datagram surface has no socket-option call of any kind, so
there is no `IP_ADD_MEMBERSHIP`, no `IP_MULTICAST_TTL` of 255, no
`SO_REUSEPORT` on port 5353 and no bind to one interface address.
Without the first of those a responder receives nothing at all.  Also
named: UDP answering `Bytes` rather than a NUL-terminated `Str` (ntp-nv
named the same row), an IPv6 datagram socket, and a way to read a
received packet's IP TTL.

**`mdns-core-nv`**, which the README recommends taking: the commonest
mDNS responder in the world is a device, and a device cannot depend on a
package that names `std.net` anywhere in its assembly.

### Design notes

**Seven of the eight modules declare no effects, and the split is
arranged so they can move.** Only `mdnssock` needs the network and the
clock; it depends on the other seven and nothing depends on it. An
`mdns-core-nv` would be `mdnsfault`, `mdnstime`, `mdnsrec`, `mdnssd`,
`mdnsclaim`, `mdnsquery` and `mdnsresp` unchanged, leaving `mdnssock`
here. The consumer for it is a device, which cannot depend on a package
that names `std.net` anywhere in its assembly. Only `mdnstime` can carry
the embedded tier as things stand, because the other six name
dns-codec-nv's types and a probe cannot be built against a dependency's
sources on this toolchain.

**`[rand]` was not taken.** mDNS needs three random draws: the delay
before the first probe, the deferral on a shared answer, and the jitter
on a cache refresh. All three are arguments — `jitter_ms`, `draw`,
`jitter_percent` — so the declared effects stay `[net, time]` and every
schedule is reproducible in a test. ntp-nv made the same choice for its
nonce and tls-nv for its handshake randomness.

**`[time]` is one function and it is monotonic.** That is a different
clock from the civil-time reads tls-nv and s3-nv make.

**Three things changed against the reference implementations**, which
are the `mdns` Rust crate for browsing and Avahi for responding. Both
own a thread and a socket and hand the caller a stream of events; here
`poll` answers a step and an instant instead. Avahi treats the class
field as a class and special-cases the cache-flush bit at each of the
half-dozen places it matters; here the bit is split off once, in
`mdnsrec`. Both hide the probe; here `mdnsclaim` is a public state
machine with a public tiebreak function, because "did this name get
claimed, and against whom" is the question an operator asks when two
devices are fighting.
