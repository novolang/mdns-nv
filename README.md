# mdns-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

Multicast DNS (RFC 6762) and DNS-based service discovery (RFC 6763),
written in novo-lang over dns-codec-nv: the probe-and-announce state
machine that claims a name on the local link, conflict resolution, a
responder for a service instance, a querier with a cache and
known-answer suppression, and the socket half over UDP multicast.

It is the multicast half of dns-codec-nv, which is what that package's
own README says stays behind when the codec goes: "the resolver's
socket, retry policy and cache stay behind".

**Every timing rule is a value the caller's clock drives**, so a
complete conformance sequence — three probes, a conflict, a rename,
three more probes and two announcements — is a test that runs in
microseconds with nothing sleeping in it.

## Adding it, and checking it

```bash
novo pkg add mdns-nv           # into your novo.toml
novo pkg build                 # type- and effect-check the package
novo test --isolate tests/mdnsclaim_tests.nv
```

`novo test` is red today and that is the point of the release: every
assertion fails with `not implemented: mdns-nv.<module>.<fn>`.  They
turn green one at a time as bodies land.

## The one example that will work

```novo
use mdnsclaim
use mdnssd
use mdnssock

// Advertise a printer on the local link, and hand back the claim the
// caller's loop will pump.
//
// Nothing here sleeps: `mdnsclaim.poll` says what to send and WHEN,
// and the caller's own event loop decides how to wait.
fn advertise(local: Str, now_ms: Int, jitter: Int) -> Result<MdnsClaim, MdnsFault> [net]
    let s = mdnssd.service("ipp", "tcp")!
    let i = mdnssd.instance("Klaus's Printer", s, "local")!
    let target = MdnsTarget { hostname: "printer", port: 631, priority: 0, weight: 0 }
    let records = mdnssd.records_for(i, target, [mdnssd.txt_pair("rp", "printers/one")],
                                     [192, 168, 1, 47])!
    let _sock = mdnssock.open(local)!
    Ok(mdnsclaim.claim(mdnssd.full_name(i)!, records, now_ms, jitter))
```

## The layer, and why

`host`, and seven of the eight modules declare nothing.

| module | row | why |
| --- | --- | --- |
| `mdnsfault` | `[]` throughout | the faults |
| `mdnstime` | `[]` throughout | every timing rule, as integer arithmetic |
| `mdnsrec` | `[]` throughout | dns-codec-nv's records, read the way mDNS means them |
| `mdnssd` | `[]` throughout | the names and the four records of a service |
| `mdnsclaim` | `[]` throughout | probing, announcing, conflict resolution |
| `mdnsquery` | `[]` throughout | the cache and known-answer suppression |
| `mdnsresp` | `[]` throughout | answering, and the four reasons not to |
| `mdnssock.open`, `.join_group`, `.close`, `.send_multicast`, `.send_to`, `.recv`, `.is_link_local`, `.interface_addresses` | `[net]` | the datagram socket and the group |
| `mdnssock.now_ms` | `[time]` | the ONE clock read in the package |

**One clock read, and it is the whole design.**  Every state machine
above takes `now_ms` as an argument and nothing in them reads a clock,
so the caller's loop reads it once per turn and passes it down.  That is
what makes a conformance sequence testable and what lets a device run
the same code on its own event loop.

`layer = "host"` and **not** `layer = "core"` with `host_modules`,
because the plan's row is `host`.  The `mdns-core-nv` section below is
the argument for moving the seven.

## The load-bearing interface

**`MdnsClaimStep.MdnsClaimWait(at_ms)`** — the variant that says
"nothing until then".

Every rule in mDNS is a timing rule.  A name is claimed by three probes
250 milliseconds apart.  A response to a shared record is deferred by a
random 20 to 120 milliseconds so that several responders aggregate
rather than collide.  An answer already multicast in the last second is
suppressed.  A cached record is refreshed at 80, 85, 90 and 95 per cent
of its lifetime.  A continuous query backs off by doubling to a ceiling
of one hour.  A responder that got any one of those wrong works
perfectly on a desk with two devices and floods a conference network —
which is the failure that is hardest to reproduce and hardest to
attribute.

So the state machines here **perform nothing and wait for nothing**.
They take `now_ms` and answer what to do and when.  Three things follow,
and each one is worth more than the architecture:

- **A conformance sequence is a unit test.**  Probe, conflict, rename,
  probe, announce — five state transitions with five numbers, running
  in microseconds, deterministic.  An implementation that slept could
  not be tested at all; one that owned a timer could be tested slowly
  and flakily.
- **A device runs it unchanged.**  Firmware has its own event loop and
  its own timer, and what it needs from a library is "wake me at T",
  not a thread.
- **The random draws are arguments too**, for the same reason the clock
  is.  `mdnstime.defer_ms(draw)` and `MdnsClaim.jitter_ms` take the
  caller's number; `[rand]` stays out of this package's row and a test
  pins the draw.

The second decision is smaller and catches more people: **the class
field is not a class**.  RFC 6762 § 10.2 takes the top bit of the
sixteen-bit class for "flush the cache for this name and type", and
§ 5.4 takes the same bit in a *question* for "answer me by unicast".
dns-codec-nv reads what it is given, so an mDNS answer's class arrives
as `DnsClassOther(32769)` rather than `DnsClassInternet` — and a
responder that compared `DnsClassInternet` matches nothing, on a network
where everything looks fine.  `mdnsrec` publishes **two pairs of
functions and not one**, because the same bit means two different things
and collapsing them is how a responder answers a multicast question by
unicast, leaving every other device on the link without the answer it
was about to overhear.

And the third, which is the one this package would most like a reader to
take away: **most of a responder is deciding not to answer.**
`mdnsresp.plan` answers `None` in the common case, and
`mdnsresp.suppress` exists because between planning a deferred response
and sending it another responder may have answered.

## The device claim, and how it was checked

`tests/embedded_probe.nv` is a firmware `main` that runs twenty-four
checks over `mdnstime` — both stolen class bits, the probe schedule, the
announcement schedule, the deferral window, the duplicate window, the
cache lifetimes in the two units the protocol mixes, the half-life rule,
the four refreshes, the query backoff and its ceiling — and parks.  It
builds:

```
novo build --target=nrf52-qemu src/probe.nv
novo: built probe.elf
```

`mdnstime` exists as a module because of this.  Every other module here
holds a `DnsRecord`, and a probe can only be built against the package's
own sources today — so the module that names nothing else is the module
that can be probed, and it is where every rule with a number in it
lives, on purpose.  A device's own responder holds its records as bytes
it built once and drives `mdnstime` for the schedule.

**The shard audit does not build it.**  The `core-embedded` row is a
`core` package's, and this package is `host`, so the audit passes it
with "`host` makes no device claim — the embedded probe is `core`'s".
The build above was done by hand.

## `mdns-core-nv` should be a row

**Take it**, and the argument is stronger here than for any other
package this lane staged.

- **The consumer is the reason the package exists.**  A lamp, a sensor
  or a controller wanting to be `kitchen.local` rather than
  `192.168.1.47` is the commonest mDNS responder there is, by a very
  large margin, and it cannot depend on a package that names `std.net`
  anywhere in its assembly.
- **Seven modules out of eight are `[]`**, and `mdnssock` is about a
  hundred lines of socket calls over an interface the other seven
  define.
- **The split is already made.**  `mdnssock` depends on the others and
  nothing depends on it; `mdnstime` names nothing at all, which is why
  the probe exists.
- **And the audit would then build the probe on every run**, instead of
  the claim being checked when somebody remembers.

What stops it here is that a second name on the grid is the grid's
decision.  The modules are arranged for the day it is taken:
`mdns-core-nv` is `mdnsfault`, `mdnstime`, `mdnsrec`, `mdnssd`,
`mdnsclaim`, `mdnsquery` and `mdnsresp` unchanged, and `mdns-nv` keeps
`mdnssock`.

One note for whoever takes it.  Only `mdnstime` can carry
`@tier(embedded)` as things stand: the other six name dns-codec-nv's
types, dns-codec-nv makes no device claim of its own, and a probe cannot
be built against a dependency's sources today.  A `mdns-core-nv` that
wanted the whole state machine on a device needs dns-codec-nv to claim
the tier first — which its own README says it is shaped for, since a
name there is wire bytes rather than a `Str` precisely so that it links
on a device.

## What is missing, by name

**Multicast group membership, which is the row this package cannot
work without.**  `std.net`'s datagram surface is `udp_socket`,
`udp_bind(port)`, `udp_send_to` and `udp_recv_from`, and there is **no
socket-option call of any kind**.  An mDNS socket needs four things
none of them can express:

| what | why it matters |
| --- | --- |
| `IP_ADD_MEMBERSHIP` on 224.0.0.251 | without it the socket receives nothing at all, which presents as a network problem |
| `IP_MULTICAST_TTL` of 255 | RFC 6762 § 11 lets a receiver refuse a packet whose TTL is not 255, and the default is 1 |
| `SO_REUSEPORT` on port 5353 | without it the second program on the machine cannot start |
| a bind to one interface address | `udp_bind` binds `INADDR_ANY`, so a laptop with three interfaces answers on one and advertises another's address |

So `mdnssock.join_group` has a fault of its own —
`MdnsFaultNoMulticastMembership` — and
`tests/mdnsbrowse_tests.nv` asserts it rather than only describing it.
The signatures here are written against the API the row would provide.

**And `udp_recv_from` answers a NUL-terminated `Str`**, so a datagram is
truncated at its first zero byte.  A DNS message's twelve-byte header is
mostly zeroes and every name ends with a zero-length label, so in
practice no mDNS packet survives that API at all.  The row is UDP in
`std.net` answering `Bytes`, beside the `send_bytes` / `recv_bytes` pair
TCP already has.  ntp-nv named the same one from the other side, for the
same reason.

**Also named**: an IPv6 datagram socket (`udp_socket` is `AF_INET`
only), which is what `ff02::fb` needs, and a way to read a received
packet's IP TTL, which is what RFC 6762 § 11's check needs —
`MdnsDatagram.ip_ttl` is `-1` for "the platform did not say" rather than
for "bad", because refusing every packet on a platform that cannot
report it would be worse than not checking.

## Where a row wanted to widen

**`[rand]`, and the design answered by not taking it.**  mDNS needs
three random draws: the jitter before the first probe (so that a room
full of devices powering up together does not collide), the 20-to-120
millisecond deferral on a shared answer, and up to two per cent of
jitter on each cache refresh.  All three are arguments — `jitter_ms`,
`draw`, `jitter_percent` — so the package's row stays `[net, time]` and
every schedule is reproducible in a test.  Same trade ntp-nv made for
its nonce and tls-nv for its handshake randomness; this is the cohort's
third instance.

**`[time]` is one function and it is monotonic.**  mDNS's rules are all
about elapsed time, and a responder whose intervals jumped when
something stepped the wall clock would probe twice or not at all.  That
is a different clock from the one tls-nv's `now_civil` and s3-nv's
`now_civil` read, and it is worth noting that the cohort now has two
kinds of `[time]` function for two different reasons.

**No row wanted to be wider than the plan wrote it.**  `[net, time]` is
what the plan said and what the package declares.

## What this does not do, on purpose

- **No DNS-SD over unicast DNS.**  RFC 6763 works over ordinary DNS
  too, with a different discovery mechanism for the domains to browse.
  That is a resolver's job and this package has no resolver.
- **No hot-plug interface tracking.**  `mdnssock.interface_addresses`
  answers what is there now; noticing that a laptop joined a different
  network is a platform-specific notification and a row of its own.
- **No sleep proxy, no DNS Long-Lived Queries, no DNS Push.**  Each is
  its own specification and none is needed to find a printer.
- **No responder threading, no scheduler and no loop.**  The state
  machines say when; the loop is the caller's.  A library that owned
  one would be a library a device's event loop could not host.
- **No `.local` resolver stub.**  Turning `printer.local` into an
  address for a program that does not know about mDNS is an operating
  system's job; `mdnssd.is_local` is the check that decides which
  resolver a name belongs to, and it is published for whoever writes
  that.
- **It does not print.**  Every failure is a value with a `message()`.

## The reference implementation

`mdns` (the Rust crate) for the browse side and `zeroconf` / Avahi for
the responder side; RFC 6762 and RFC 6763 are the specifications, and
Apple's own Bonjour is what every other implementation is measured
against on the wire.

Three things change in the port.

Both references own a thread and a socket and hand the caller a stream
of events.  Here there is no thread, no socket inside the state machine
and no stream: `poll` answers a step and an instant, for the reasons the
load-bearing section gives.

Avahi treats the class field as a class and special-cases the
cache-flush bit at each of the half-dozen places it matters.  Here the
bit is split off once, in `mdnsrec`, and nothing else in the package
compares a class as an integer.

And both references hide the probe.  Here `mdnsclaim` is a public state
machine with a public tiebreak function, because "did this name get
claimed, and against whom" is the question an operator asks when two
devices are fighting, and a library that answered it only in a log line
cannot be asked.

## Status

| item | implemented |
| --- | --- |
| `mdnsfault` — `MdnsFault` | types only |
| `mdnsfault.rename_would_help`, the `message` impl | no |
| `mdnstime` — every constant: the class bit and mask, the port, the group, the IP TTL, the two record lifetimes, the probe and announce counts and intervals, the deferral bounds, the duplicate window, the query bounds, the conflict ceiling | yes — they are constants |
| `mdnstime.class_bits`, `.has_class_bit`, `.with_class_bit` | no |
| `mdnstime.probe_at_ms`, `.announce_at_ms`, `.defer_ms`, `.is_suppressed` | no |
| `mdnstime.refresh_at_ms`, `.expires_at_ms`, `.may_suppress`, `.remaining_ttl` | no |
| `mdnstime.next_query_ms`, `.conflict_backoff_ms`, `.default_ttl_for` | no |
| `mdnsrec.class_of`, `.cache_flush_of`, `.with_cache_flush`, `.qclass_of`, `.unicast_wanted`, `.with_unicast_wanted` | no |
| `mdnsrec.is_goodbye`, `.as_goodbye`, `.is_legacy_query`, `.for_legacy` | no |
| `mdnsrec.query_message`, `.response_message`, `.legacy_response`, `.probe_message` | no |
| `mdnsrec.is_probe`, `.is_response`, `.has_more_known_answers` | no |
| `mdnsrec.same_key`, `.same_record`, `.canonical_rdata`, `.default_ttl` | no |
| `mdnssd` — `MdnsService`, `MdnsInstance`, `MdnsTarget`, `MdnsTxtPair` | types only |
| `mdnssd.MDNS_DOMAIN`, `.MDNS_SERVICE_ENUM`, `.MDNS_TXT_KEY_MAX`, `.MDNS_TXT_SOFT_MAX` | yes — they are constants |
| `mdnssd.service_of_text`, `.service`, `.with_subtype`, `.service_text`, `.service_name`, `.subtype_name` | no |
| `mdnssd.instance`, `.full_name`, `.instance_of`, `.host_name`, `.is_local` | no |
| `mdnssd.txt_pairs`, `.txt_get`, `.txt_record`, `.txt_pair`, `.txt_flag` | no |
| `mdnssd.records_for`, `.additionals_for`, `.enumeration_record`, `.renamed`, `.renamed_host` | no |
| `mdnsclaim` — `MdnsClaimState`, `MdnsClaimStep`, `MdnsClaim` | types only |
| `mdnsclaim.claim`, `.poll`, `.sent_at`, `.saw_message`, `.rename`, `.leave` | no |
| `mdnsclaim.wins_tiebreak`, `.conflicts_with`, `.state_of`, `.conflicts_of`, `.is_held`, `.records_of` | no |
| `mdnsclaim.probe_message_of`, `.announcement_of`, `.goodbye_of` | no |
| `mdnsquery` — `MdnsEntry`, `MdnsCache`, `MdnsBrowse`, `MdnsChange` | types only |
| `mdnsquery.cache`, `.feed`, `.changes_of`, `.expire`, `.lookup`, `.entries_of`, `.entry_count` | no |
| `mdnsquery.due_for_refresh`, `.refresh_sent` | no |
| `mdnsquery.browse`, `.browse_due`, `.browse_message`, `.browse_sent` | no |
| `mdnsquery.known_answers`, `.is_known`, `.instances_of`, `.accept_response` | no |
| `mdnsresp` — `MdnsPlan`, `MdnsResponder` | types only |
| `mdnsresp.responder`, `.plan`, `.suppress`, `.message_of`, `.sent_at` | no |
| `mdnsresp.answers_question`, `.answers_for`, `.is_unique`, `.recently_sent` | no |
| `mdnsresp.split_response`, `.split_known_answers`, `.records_of`, `.with_record`, `.without_name` | no |
| `mdnssock` — `MdnsSocket`, `MdnsDatagram` | types only |
| `mdnssock.open`, `.join_group`, `.close`, `.send_multicast`, `.send_to`, `.recv` | no |
| `mdnssock.now_ms`, `.is_link_local`, `.interface_addresses`, `.group_address` | no |
