# mdns-nv

Multicast DNS lets a host answer DNS queries for its own name with no
DNS server anywhere on the network. It is specified in
[RFC 6762](https://www.rfc-editor.org/rfc/rfc6762). DNS-based Service
Discovery, [RFC 6763](https://www.rfc-editor.org/rfc/rfc6763), builds on
it to name and describe a service, which is how a printer appears in a
list of printers. This package brings both to novo-lang: the probe that
claims a name, conflict resolution, a responder for a service instance,
a querier with a cache, and the socket half over UDP multicast. The
messages themselves are
[dns-codec-nv](https://novo-lang.org/packages/dns-codec-nv)'s, and this
package depends on it.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What multicast DNS is

Ordinary DNS sends a question to a server and gets an answer back.
Multicast DNS sends the same question, in the same message format, to a
**multicast group** on the local link instead. A multicast group is an
address every interested host on the link listens to. Every host hears
the question, the host that owns the name answers, and every host hears
the answer too. Section 2 of RFC 6762 reserves the top-level domain
`.local` for names that work this way.

Nothing assigns those names, so a host has to take one. Section 8.1
calls that **probing**: three queries, 250 milliseconds apart, each
carrying in its authority section the records the host proposes to
claim. If nobody objects, the host **announces** — it multicasts those
records twice, a second apart, telling every cache on the link to
replace what it had. If somebody does object, the host renames and
probes again.

DNS-based Service Discovery names a running service rather than a
machine. A **service instance name** is three parts joined:
`Klaus's Printer._ipp._tcp.local`. The first label is a human-readable
instance name, the middle two are the service type and the transport,
and the last is the domain. Four records describe one instance
(RFC 6763 section 4): a PTR record listing the instance under its
service type, an SRV record giving the host and port, a TXT record
holding whatever else a client needs, and an A or AAAA record giving the
host's address.

Every rule in mDNS is a rule about time. Probes are spaced, answers to
shared records are deferred so that several responders aggregate instead
of colliding, an answer already sent is suppressed for a second, and a
cached record is refreshed before it expires. These are the numbers.

| Constant | Value | What it is |
| --- | --- | --- |
| `MDNS_PORT` | 5353 | The port both querier and responder bind (section 2) |
| `MDNS_GROUP_V4` | 224.0.0.251 | The IPv4 multicast group, as four octets in one integer (section 3) |
| `MDNS_IP_TTL` | 255 | The IP header TTL every mDNS packet must carry (section 11) |
| `MDNS_TTL_HOST` | 120 seconds | The lifetime of an A, AAAA or SRV record (section 10) |
| `MDNS_TTL_OTHER` | 4500 seconds | The lifetime of a PTR or TXT record (section 10) |
| `MDNS_PROBE_COUNT` | 3 | Probes sent before a name is claimed (section 8.1) |
| `MDNS_PROBE_INTERVAL_MS` | 250 | Milliseconds between probes (section 8.1) |
| `MDNS_PROBE_JITTER_MS` | 250 | The width of the random delay before the first probe (section 8.1) |
| `MDNS_ANNOUNCE_COUNT` | 2 | Announcements after a successful probe (section 8.3) |
| `MDNS_ANNOUNCE_INTERVAL_MS` | 1000 | Milliseconds to the second announcement, doubling after that (section 8.3) |
| `MDNS_DEFER_MIN_MS` | 20 | The shortest deferral of an answer to a shared record (section 6) |
| `MDNS_DEFER_MAX_MS` | 120 | The longest (section 6) |
| `MDNS_DUPLICATE_WINDOW_MS` | 1000 | How long an answer stays suppressed after it was multicast (section 6) |
| `MDNS_QUERY_FIRST_MS` | 1000 | The first interval of a continuous query (section 5.2) |
| `MDNS_QUERY_MAX_MS` | 3600000 | The ceiling that interval doubles up to, one hour (section 5.2) |
| `MDNS_MAX_CONFLICTS` | 15 | Renames before a host gives up and waits five seconds (section 9) |
| `MDNS_CLASS_BIT` | 0x8000 | The top bit of the class field, which mDNS takes for itself (sections 10.2 and 5.4) |
| `MDNS_TXT_KEY_MAX` | 9 | The longest a TXT key should be (RFC 6763 section 6.4) |
| `MDNS_TXT_SOFT_MAX` | 1300 bytes | The size a TXT record should stay under, so four records fit one datagram |

The state machines here perform nothing and wait for nothing. Each takes
the current time in milliseconds as an argument and answers what to do
and when. The caller's loop reads the clock once a turn and passes the
number down. A complete conformance sequence — three probes, a conflict,
a rename, three more probes and two announcements — is therefore a test
that runs in microseconds with nothing sleeping in it.

## Install

```
novo pkg add mdns-nv
```

## Example

```novo
use mdnsclaim
use mdnsfault
use mdnssd

// The claim on the name `Klaus's Printer._ipp._tcp.local`, with the four
// records that describe the printer behind it.
fn printer_claim(now_ms: Int) -> Result<MdnsClaim, MdnsFault>
    let service = mdnssd.service("ipp", "tcp")!
    let instance = mdnssd.instance("Klaus's Printer", service, mdnssd.MDNS_DOMAIN)!
    let target = MdnsTarget { hostname: "printer", port: 631, priority: 0, weight: 0 }
    let records = mdnssd.records_for(instance, target,
                                     [mdnssd.txt_pair("rp", "printers/one")],
                                     [192 as u8, 168 as u8, 1 as u8, 47 as u8])!
    // 137 is the caller's random draw for the delay before the first
    // probe, which the protocol puts in 0 to 250 milliseconds.
    Ok(mdnsclaim.claim(mdnssd.full_name(instance)!, records, now_ms, 137))

fn main() [io]
    match printer_claim(0)
        Err(f) => println("cannot advertise: ${f.message()}")
        Ok(c)  =>
            // Ask what to do at this instant. The answer is one
            // instruction and one time; nothing here sends or sleeps.
            match mdnsclaim.poll(c, 0)
                MdnsClaimSend(_, at_ms) => println("multicast this at ${at_ms}")
                MdnsClaimWait(at_ms)    => println("nothing to do until ${at_ms}")
                MdnsClaimReady(_)       => println("the name is held")
                MdnsClaimGone           => println("finished leaving")
                MdnsClaimFailed(f)      => println("gave up: ${f.message()}")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: mdns-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `mdnsfault` | Everything that can go wrong, as one enum with a message. A name conflict is not among them, because renaming is a state change rather than a failure. |
| `mdnstime` | Every timing rule as integer arithmetic over milliseconds, and the two bits mDNS takes from the class field. It names nothing outside itself. |
| `mdnsrec` | dns-codec-nv's records and questions read the way mDNS means them: the cache-flush bit, the unicast-response bit, goodbyes, legacy queries, and the headers a query, a response and a probe carry. |
| `mdnssd` | RFC 6763's names and the four records that describe one service instance, with the rules for TXT keys and values and for renaming after a conflict. |
| `mdnsclaim` | Probing, announcing and leaving, and the comparison that settles two hosts probing for the same name at the same time. |
| `mdnsquery` | The querier: a bounded cache with lifetimes and refreshes, continuous queries with backoff, and known-answer suppression. |
| `mdnsresp` | The responder: which records answer a question, when to send them, how to split a reply that does not fit, and the four reasons to stay quiet. |
| `mdnssock` | The datagram socket, the multicast group and the clock. The only module that needs network or time access. |

## How to choose an entry point

**To be found, use `mdnsclaim` and then `mdnsresp`.** `mdnsclaim` takes
the name and the records and pumps the probe-and-announce sequence until
the name is held. `mdnsresp` then answers the queries that arrive for
it. `mdnssd` builds the name and the four records both of them take.

**To find other hosts, use `mdnsquery`.** `browse` is a continuous query
for one name and type, `feed` puts a response into the cache, and
`changes_of` says what appeared, changed or went away.

**`mdnssock` is the transport for both.** It opens the socket, joins the
group, sends and receives, and reads the clock. A program with its own
event loop and its own socket can use the other seven modules and leave
this one out.

**A device driving its own schedule can use `mdnstime` alone.** It holds
every rule with a number in it, as integer arithmetic, and it names no
other module and no other package.

## The rules a user needs

1. **A name is not yours until you have probed for it** (section 8.1).
   Three queries, 250 milliseconds apart, each carrying the proposed
   records in the authority section. A responder that announced without
   probing takes a name somebody else is already answering for, and the
   two then answer the same question differently forever.
2. **Two hosts probing at once are separated lexicographically, not by
   who asked first** (section 8.2). `wins_tiebreak` compares class, then
   type, then the uncompressed RDATA byte by byte, and the numerically
   greater wins. Both hosts compute the same answer, which is what a
   first-come rule cannot do.
3. **The class field is not a class.** Section 10.2 takes its top bit in
   a record to mean "flush the cache for this name and type", and
   section 5.4 takes the same bit in a question to mean "answer me by
   unicast". dns-codec-nv reads what it is given, so an mDNS answer's
   class arrives as `DnsClassOther(32769)`. Use `mdnsrec.class_of` and
   `.cache_flush_of` for a record, `.qclass_of` and `.unicast_wanted`
   for a question. Nothing in this package compares a class as an
   integer.
4. **A goodbye is a lifetime of zero** (section 10.1). A responder
   leaving the link multicasts its records with a TTL of 0, which tells
   every cache to drop them. A responder that just stopped leaves itself
   in every list on the link for seventy-five minutes.
5. **There are two record lifetimes and not one** (section 10). 120
   seconds for A, AAAA and SRV, 4500 seconds for PTR and TXT. The long
   one on an address keeps a stale answer in every cache for an hour
   after the machine moved.
6. **A query carries the answers the asker already has** (section 7.1).
   The answer section of a query is its known-answer list, and a
   responder that refused a query with answers in it refuses every
   well-behaved querier on the link. A responder whose own record is in
   that list stays quiet.
7. **A known answer may only be sent while more than half its lifetime
   remains** (section 7.1). An entry the querier is about to refresh
   anyway would otherwise suppress the very response that would have
   refreshed it. `mdnstime.may_suppress` is that rule.
8. **The truncated bit does not mean "retry over TCP"** (section 7.2).
   There is no TCP in mDNS. It means more known answers follow in
   another packet, and a responder waits 400 to 500 milliseconds for
   them before answering.
9. **A unique record is answered at once and a shared record is
   deferred** (section 6). Nobody else can answer an SRV, a TXT or an
   address for a probed name. The PTR that lists an instance under its
   service type is answered by every instance, so it waits a random 20
   to 120 milliseconds and several responders aggregate into one packet.
10. **An answer multicast in the last second is not sent again**
    (section 6), and a deferred answer another responder sends first is
    dropped. `mdnsresp.suppress` is the second of those, applied between
    planning a response and sending it.
11. **A query from a source port other than 5353 is a legacy query**
    (section 6.7). It came from an ordinary resolver, so the reply goes
    back by unicast, echoes the question, clears the cache-flush bit and
    caps the lifetime at ten seconds.
12. **An incoming packet's IP TTL should be 255** (section 11). mDNS is
    link-local by definition, and anything less has been forwarded.
    `MdnsDatagram.ip_ttl` is `-1` when the platform did not report it,
    which means unknown and not bad.
13. **A service instance name is human-readable and is never escaped**
    (RFC 6763 section 4.1.1). `Klaus's Printer (upstairs)` is one legal
    DNS label, apostrophe and spaces and parentheses included. A caller
    that turned the space into `\032` publishes a name with the escape
    in it, in every browser, for as long as the service runs.
14. **A TXT key with no value is not a key with an empty value**
    (RFC 6763 section 6.4). `key` means present without a value, `key=`
    means present with an empty one. Keys are compared without regard to
    case, and where a key appears twice the first occurrence wins.
15. **All four records of an instance go in one packet** (RFC 6763
    section 12). The PTR is the answer and the SRV, TXT and address ride
    along in the additional section. Answering only what was asked works
    and takes three round trips, which is the difference between a list
    that appears and a list that fills in.
16. **The clock and the random draws are yours.** Every state machine
    takes `now_ms`, and `mdnssock.now_ms` is the one function in the
    package that reads a clock. The three random numbers the protocol
    needs — the delay before the first probe, the deferral on a shared
    answer, and the jitter on a cache refresh — are arguments as well, so
    a test pins them.
17. **The clock must be monotonic.** mDNS measures elapsed time, and a
    responder whose intervals jumped when something stepped the wall
    clock would probe twice or not at all.
18. **Fifteen renames is the link saying no** (section 9). After that a
    host waits five seconds before trying again, and
    `MdnsFaultTooManyConflicts` is what a caller is told. Renaming in a
    tight loop makes the link worse for everybody.

## Running on a microcontroller

novo-lang lets a package state which of its modules can run on a device
with no heap allocator, and the compiler checks that claim on every
build. Here the claim covers `mdnstime`, and the reason it covers that
module is worth saying: a lamp, a sensor or a controller that wants to
be `kitchen.local` rather than `192.168.1.47` is the commonest mDNS
responder there is, and every rule it has to obey is arithmetic over
milliseconds.

`tests/embedded_probe.nv` is that claim as a program that either builds
or does not. It runs twenty-four checks over `mdnstime` — both stolen
class bits, the probe and announcement schedules, the deferral window,
the duplicate window, the two record lifetimes, the half-life rule, the
four refreshes, the query backoff and its ceiling — and then parks.

```bash
novo build --target=nrf52-qemu src/probe.nv
```

The other seven modules are outside the claim. Six of them name
dns-codec-nv's types, and a probe cannot be built against a dependency's
sources on this toolchain, so a claim over them is one nothing checks.
A device's own responder therefore holds its records as bytes it built
once and drives `mdnstime` for the schedule.

## What is not included

- **Multicast group membership, which this package cannot work
  without.** `std.net`'s datagram surface is `udp_socket`,
  `udp_bind(port)`, `udp_send_to` and `udp_recv_from`, with no
  socket-option call of any kind. That leaves four things unavailable:
  joining 224.0.0.251, without which the socket receives nothing at all;
  an outgoing IP TTL of 255, without which receivers that check section
  11 drop the packets; address reuse on port 5353, without which a
  second program on the machine cannot start; and a bind to one
  interface's address rather than to all of them. `mdnssock.join_group`
  answers `MdnsFaultNoMulticastMembership` today, and
  `tests/mdnsbrowse_tests.nv` asserts that rather than describing it.
  The signatures here are written against the interface a working
  standard library would provide.
- **A datagram surface that answers bytes.** `udp_recv_from` answers a
  NUL-terminated string, so a datagram is cut at its first zero byte. A
  DNS header is mostly zeroes and every name ends in a zero-length
  label, so no mDNS packet survives that call.
- **IPv6.** `udp_socket` is IPv4 only, so the `ff02::fb` group is out of
  reach.
- **DNS-SD over unicast DNS.** RFC 6763 works over ordinary DNS as well,
  with a different way of discovering which domains to browse. That
  needs a resolver, and this package has none.
- **Interface hot-plug.** `mdnssock.interface_addresses` answers what is
  there now. Noticing that a laptop joined a different network is a
  platform-specific notification.
- **Sleep proxy, DNS Long-Lived Queries and DNS Push.** Each is its own
  specification, and none is needed to find a printer.
- **A thread, a scheduler and a loop.** The state machines say when; the
  waiting is the caller's. A library that owned a loop is one a device's
  event loop could not host.
- **A `.local` resolver stub.** Turning `printer.local` into an address
  for a program that knows nothing about mDNS is an operating system's
  job. `mdnssd.is_local` is the check that decides which resolver a name
  belongs to, published for whoever writes one.

## Related packages

- [dns-codec-nv](https://novo-lang.org/packages/dns-codec-nv) is the
  codec half a reader wants when there is no multicast involved. It is
  RFC 1035's wire format with no socket under it: the header, the
  sections, the record types, name compression and EDNS(0). A program
  that needs to parse or build a DNS message wants that package alone. A
  program that needs to claim a name on the local link, or to find what
  is on it, wants this one and gets that one with it.
- [ipaddr-nv](https://novo-lang.org/packages/ipaddr-nv) turns the octets
  of an A or AAAA record into an address value that prints and parses.
  This package passes addresses as the strings the socket takes and the
  octets the record holds.
- [ntp-nv](https://novo-lang.org/packages/ntp-nv) is the other UDP
  client on the registry. It has the same shape: the packet arithmetic
  is separate from the socket, and the caller owns the clock.
- `std.net` in the standard library is the datagram socket `mdnssock`
  calls, and `std.time` is where a monotonic clock would come from.

## Tests

```bash
novo test tests/mdnsclaim_tests.nv       # 15 tests: claiming and the timing rules
novo test tests/mdnsbrowse_tests.nv      # 15 tests: browsing, the cache and answering
```

The expected values are RFC 6762's and RFC 6763's own: the probe count
and interval of section 8.1, the announcement schedule of section 8.3,
the deferral bounds and duplicate window of section 6, the refresh
percentages and query backoff of section 5.2, the two lifetimes of
section 10, and the class bit of sections 10.2 and 5.4. The instance
names, service types and TXT rules are RFC 6763 section 4 and
section 6.

Not one of these tests sleeps. A complete claim — three probes 250
milliseconds apart, a conflict, a rename, three more probes and two
announcements — is a sequence of calls with numbers in them, and it runs
in microseconds. The suite also asserts that the multicast group cannot
be joined on this toolchain, so the gap named above is a failing
assertion rather than a paragraph.

The tests compile today and fail at run, each on the
`not implemented: mdns-nv.<module>.<fn>` panic that is its body. That is
the expected state of an interface release. They turn green one at a
time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `mdnstime`'s constants: the class bit and mask, the port, the group, the IP TTL, the two record lifetimes, the probe and announce counts and intervals, the deferral bounds, the duplicate window, the query bounds and the conflict ceiling | yes (they are constants) |
| `mdnssd.MDNS_DOMAIN`, `.MDNS_SERVICE_ENUM`, `.MDNS_TXT_KEY_MAX`, `.MDNS_TXT_SOFT_MAX` | yes (they are constants) |
| `mdnsfault.MdnsFault`, `.rename_would_help`, the `message` implementation | no |
| `mdnstime.class_bits`, `.has_class_bit`, `.with_class_bit` | no |
| `mdnstime.probe_at_ms`, `.announce_at_ms`, `.defer_ms`, `.is_suppressed` | no |
| `mdnstime.refresh_at_ms`, `.expires_at_ms`, `.may_suppress`, `.remaining_ttl` | no |
| `mdnstime.next_query_ms`, `.conflict_backoff_ms`, `.default_ttl_for` | no |
| `mdnsrec.class_of`, `.cache_flush_of`, `.with_cache_flush`, `.qclass_of`, `.unicast_wanted`, `.with_unicast_wanted` | no |
| `mdnsrec.is_goodbye`, `.as_goodbye`, `.is_legacy_query`, `.for_legacy` | no |
| `mdnsrec.query_message`, `.response_message`, `.legacy_response`, `.probe_message` | no |
| `mdnsrec.is_probe`, `.is_response`, `.has_more_known_answers` | no |
| `mdnsrec.same_key`, `.same_record`, `.canonical_rdata`, `.default_ttl` | no |
| `mdnssd.service_of_text`, `.service`, `.with_subtype`, `.service_text`, `.service_name`, `.subtype_name` | no |
| `mdnssd.instance`, `.full_name`, `.instance_of`, `.host_name`, `.is_local` | no |
| `mdnssd.txt_pairs`, `.txt_get`, `.txt_record`, `.txt_pair`, `.txt_flag` | no |
| `mdnssd.records_for`, `.additionals_for`, `.enumeration_record`, `.renamed`, `.renamed_host` | no |
| `mdnsclaim.claim`, `.poll`, `.sent_at`, `.saw_message`, `.rename`, `.leave` | no |
| `mdnsclaim.wins_tiebreak`, `.conflicts_with`, `.state_of`, `.conflicts_of`, `.is_held`, `.records_of` | no |
| `mdnsclaim.probe_message_of`, `.announcement_of`, `.goodbye_of` | no |
| `mdnsquery.cache`, `.feed`, `.changes_of`, `.expire`, `.lookup`, `.entries_of`, `.entry_count` | no |
| `mdnsquery.due_for_refresh`, `.refresh_sent` | no |
| `mdnsquery.browse`, `.browse_due`, `.browse_message`, `.browse_sent` | no |
| `mdnsquery.known_answers`, `.is_known`, `.instances_of`, `.accept_response` | no |
| `mdnsresp.responder`, `.plan`, `.suppress`, `.message_of`, `.sent_at` | no |
| `mdnsresp.answers_question`, `.answers_for`, `.is_unique`, `.recently_sent` | no |
| `mdnsresp.split_response`, `.split_known_answers`, `.records_of`, `.with_record`, `.without_name` | no |
| `mdnssock.open`, `.join_group`, `.close`, `.send_multicast`, `.send_to`, `.recv` | no |
| `mdnssock.now_ms`, `.is_link_local`, `.interface_addresses`, `.group_address` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
