---
layout: post
title: "The APRS Bots Are Winning"
date: 2026-10-03
description: "How I learned that a globally distributed multiplayer game had moved into my gateway's coverage area, and what a game does to a shared 1,200 baud packet channel."
---

I've got a VHF/UHF transceiver radio in my garage running an APRS Internet gateway. It would transmit over the radio occasionally, usually just my own periodic beacons or the rare relay of someone else's message from the Internet.

This changed significantly starting around August 2026. I would walk into the garage and hear several consecutive transmits, far more frequent than the occasional beacon I was used to.

My amateur radio callsign is [KK6DCI](https://www.qrz.com/db/KK6DCI), and I operate my APRS Internet Gateway (IGate) as KK6DCI-2. The station comprises a [Yaesu FT-7900R](https://yaesu.com/product-detail.aspx?Model=FT-7900R&CatName=Legacy) running 50 watts into a roof-mounted [Comet GP-3](https://www.cometantenna.com/product/comet-gp-3/), wired to a [Raspberry Pi Zero 2 W](https://www.raspberrypi.com/products/raspberry-pi-zero-2-w/) through a [SignaLink USB](https://www.tigertronics.com/slusbmain.htm) audio interface, sitting on [144.39 MHz](https://www.arrl.org/band-plan). That frequency is a shared channel. When my radio keys, it is 50 watts of packet radio occupying airtime that a few hundred other people in the Bay Area are also trying to use. Every transmission I add is somebody else's packet delayed.

I assumed I had broken my own configuration. I had, partly. The other half of the answer was my discovery of a globally distributed multiplayer game, running entirely over APRS text messages. This post covers my process of understanding this increase in APRS traffic, and the novel new way that HAM radio operators are putting an old protocol to use for fun.

## What an IGate is, and what it is obliged to do

APRS stations beacon position, weather, and status. Digipeaters repeat those packets across a region. An IGate (internet gateway) is the bridge in both directions: it takes what it hears on RF and forwards it to [APRS-IS](http://www.aprs-is.net/), the internet backbone where the world's traffic is collected, and it takes a filtered slice of internet traffic and transmits it back out over local RF.

![A normal regional interaction. A station beacons to a digipeater, the digipeater and the gateway carry it to APRS-IS, and messages come back down the same chain in reverse.](/public/images/aprs-topology-regional.webp)
The receive direction is nearly without constraints. The transmit direction is heavily gated, and this is where every subtle problem in this post lives. The critical detail:

**The APRS-IS server decides delivery before your client filter runs.**

When a message is addressed to some station, the server asks which of its gateways have heard that station on RF recently. It hands the message to those gateways, and each one keys its own radio. The formal version lives in the [APRS-IS IGate gating specification](http://www.aprs-is.net/IGateDetails.aspx): message packets go to RF when the receiving station has been heard within range, the sending station has *not* been heard via RF, and the receiving station has not been heard via the internet. Your own filter is a *second* gate, not the first.

Two config objects implement that second gate, and conflating them is a classic trap:

- `FILTER IG 0 (...)` selects which APRS-IS packet types are eligible for RF at all. Mine is `( t/m & ! g/BLN* )`: message-type packets, excluding bulletins.
- `IGFILTER ...` is a [server-side filter](http://www.aprs-is.net/javaprsfilter.aspx) telling APRS-IS what to send *this* IGate in the first place.

The [Dire Wolf User Guide](https://github.com/wb2osz/direwolf/blob/master/doc/User-Guide.pdf) is blunt about the first of those. Its default IS-to-RF filter is `i/180`, which passes messages only for a station heard over the radio in the last 180 minutes, and the guide says plainly that `t/m` "should not be used for IS>RF IGate". I had swapped `i/180` out for `t/m`, which is exactly the mistake the documentation warns against.

The delivery guarantee, stated plainly: a message addressed to a station that has been heard on RF in the region will be transmitted over RF in the region. Distance is not part of the addressing, and for message packets it cannot be, because third-party message frames carry no position to measure. This is what enables someone in North America to send an APRS message over RF to someone in Europe.

## Issue 1: I broke it myself

The troubleshooting began 2026-08-28. I went through the Dire Wolf log and sorted every transmitter key by cause. Every `PTT 0 = 1` traced back to an `[ig>tx]` line, internet to RF. None traced to RF-heard digipeating. So my digipeat rule was behaving. The transmit was all coming in from the internet side, and it was all message traffic.

Here is what was in my config:

```
IGFILTER  t/m/KK6DCI-2/160  b/KK6DCI
```

`t/m` means message-type packets. `b/KK6DCI` is a budlist, the sensible part, passing traffic whose source or destination is my own callsign. The problem is `/160`.

I had read that number as a limit on how far away the message's destination was. It is not that. In the [APRS-IS filter language](http://www.aprs-is.net/javaprsfilter.aspx) the form `t/<types>/<call>/<km>` puts a radius limit around the named station, in kilometres, for the requested packet types. Mine named my own station, so it matched every message-type packet within 160 km of Benicia whether or not the station it was addressed to had ever been heard on RF. A 160 km circle, about 100 miles, centred there covers Sacramento, Stockton, the South Bay, and most of the Bay Area. I was relaying every chatty APRS messaging-bot exchange in a wide slice of Northern California, then keying each one out over the FT-7900R at 50 watts.

The default I overrode was already correct: Dire Wolf relays a message to RF only if its addressee has actually been heard on RF, which is exactly the condition under which RF delivery can work. My distance clause replaced it with a rule that over-relays by design.

The fix, applied that day:

```
IGFILTER  b/KK6DCI
```

Drop the distance clause, keep the budlist, let the default stand. If I ever want a relay safety net beyond that, the radius belongs at fill-in digi range, roughly 30 to 40 km (twenty to twenty-five miles), never 160.

One transferable sentence: **a radius on a type filter is a radius around the station you name in it, not a filter on who the message is for.**

## Issue 2: the radio did not go quiet

Transmit volume dropped after the fix. Still, it did not return to the earlier baseline.

A week and a half later, on 2026-09-09, I pulled a fresh log. Every `[ig>tx]` line was a message to or from the same local operator, [`KO6PAX-7`](https://www.qrz.com/db/KO6PAX), pinging a bot I had never heard of. `Ping`, `ack`, `PONG heard on RF`, `test -7 rf`. The traffic was not from some distant station the distance filter had been sweeping up. It was a station twenty miles away, genuinely local, and my IGate was correctly obliged to relay it.

The thing it was talking to was called `OTA`.

## What APRS OTA actually is

`OTA` is the bot for [aprsota.org](https://aprsota.org), "APRS On The Air": a [POTA](https://parksontheair.com/)-style activation game carried entirely on APRS message packets. Anyone capable of sending and receiving APRS messages and beaconing can play, and the service's [handbook](https://aprsota.org/handbook) is the rulebook.

The mechanics are all text packets:

- A **host** sends `HOST`, beacons a position within five minutes, and is on the air for 60 minutes, with each completed contact adding time. For hosting, the RF leg is mandatory: messages must be sent and received over the air, which is what keeps the game from being an internet chat room.
- A **chaser** sends `CQ` for the active list, then `CHASE <call>`, then beacons a position.
- The bot relays between them, prompts for the beacon, the host replies `QSL`, and the contact lands on a live map, a logbook, and a leaderboard.

Commands: `HOST`, `CQ`, `CHASE`, `QSL`, `PING`, `DONE`, `BLAST`, `PLAN`, `STAY`, `UNPLAN`, `CHASERS`, `TIME`, `INFO`, `HELP`.

Here's a real exchange using the graywolf APRS client messaging [during an APRS-OTA activation](https://aprsota.org/N1BAM/ops/18#qso-KK6DCI):

![My side of that QSO, in graywolf's messaging client. CQ asks for the active list, CHASE N1BAM-2 claims the contact, and N1BAM's QSL comes back.](/public/images/aprs-graywolf-chase.webp)
Here's how APRS OTA recorded the exchange:

![The same exchange as the service logged it, line for line. OTA is the bot. Every message here crossed my local RF link.](/public/images/aprs-ota-transcript.webp)
![The resulting contact: KK6DCI-2 in CM88vc worked N1BAM-2 in FN31ll, 4,157 km, in 1 minute 49 seconds. That distance is grid-to-grid great-circle, not radio distance. Both radio legs were local; the gap in between was the internet.](/public/images/aprs-ota-qso-card.webp)
Two stations 4,157 km apart, neither able to hear the other, completing a contact on a 1980s packet protocol over two metres. That is the whole game in one screenshot.

The scale of interest in the APRS OTA game is real. As compiled by the service's [stats page](https://aprsota.org/stats) on 3 October 2026 at 2202z, aprsota.org reports 21,872 QSOs across 2,823 ops by 518 operators in 33 countries, with 845 six-digit grids and 255 four-digit grids worked, a biggest day of 769 QSOs across 78 ops (12 September 2026), and a longest logged contact of 18,931 km ([9W3FAB](https://www.qrz.com/db/9W3FAB) worked [HJ4AGL](https://www.qrz.com/db/HJ4AGL), 27 September). All-time command counters show the shape of the traffic: `CHASE` 29,448, `QSL` 26,187, `CQ` 11,984, `HOST` 3,784, `PING` 3,962.

It's worth mentioning the activity is built not to clog the channel. The rate limits are enforced: `CHASE` and `PING` at 2 per minute and 5 per 10 minutes, `HOST` at 2 per minute and 3 per 10 minutes, with repeated errors making the bot fall silent, first for an hour and then for 24 hours.

## Measuring it properly

Log samples only show what my station *transmitted*. To see what the network was actually doing I went to the corpus: a Postgres database I maintain that stores every RF packet this IGate has demodulated since February 2026, one row per unique packet, with receive timestamp, source, path, and information field. By today's standards, this volume of data is still tiny and inexpensive to store and query.

Monthly RF message packets heard here, 2026:

| Month | Message packets | of which OTA |
|---|---:|---:|
| March | 271 | 0 |
| April | 258 | 0 |
| May | 246 | 0 |
| June | 321 | 0 |
| July | 255 | 0 |
| August | 719 | 270 |
| September | 1,351 | 836 |
| October (through 3 Oct) | 89 | 12 |

![Monthly RF message volume, with OTA game traffic highlighted. It is exactly zero before August.](/public/images/aprs-monthly-volume.webp)
Traffic sat between roughly 250 and 320 packets a month for five months, then went 719, then 1,351. That is a 5x increase in two months. Distinct message senders went from 46 in July to 91 in September, so this is not one station retrying. The first OTA packet this IGate ever heard was on 16 August 2026 at 10:25 PT, from [`KO6MBI-9`](https://www.qrz.com/db/KO6MBI) sending a `QSL`. Before that date: zero.

![August, day by day. The first OTA packet arrives 16 August. By 28 August, the day I applied the filter fix, OTA is 57 of 63 packets. Note 30 August: the busiest day of the month, 117 packets, was mostly NOT the game.](/public/images/aprs-august-daily.webp)
I want to be careful about cause and effect, because it is easy to get backwards. The filter fix is why I *noticed* the OTA traffic, not why it started. The bug had been spraying a region's message traffic for months. Fixing it left behind the real, legitimate, local load, and that load was new.

Over the last 30 days (3 September to 2 October), this IGate heard 1,372 RF message packets:

| Category                                  | Packets | Share |
| ----------------------------------------- | ------: | ----: |
| OTA game traffic (CHASE/CQ/PING/QSL/HOST) |     739 |   54% |
| Person to person                          |     202 |   15% |
| Ack / handshake overhead                  |     195 |   14% |
| Other bots and services                   |     117 |    9% |
| Weather bots                              |      84 |    6% |
| Gateway bots (SMS/email/Winlink)          |      35 |    3% |

By counterparty the picture is starker: the OTA bot is on the other end of 813 packets, 59 percent of all message traffic, against 202 packets (15 percent) from human stations.

![1,372 RF message packets over 30 days, by category.](/public/images/aprs-packet-categories.webp)
![The same window by counterparty. One game bot: 59 percent.](/public/images/aprs-source-breakdown.webp)
Broken down by command, the messages addressed to the bot in that window were `CHASE` 265, `QSL` 199, `PING` 133, `CQ` 88, `HOST` 30, plus 74 bare acknowledgements.

All time, this IGate has logged 3,510 message packets, 509,536 position packets, and 1,604 distinct stations, of which 1,118 messages have been addressed to the OTA bot since 16 August. And the human load at the centre of it is two people: [`K6BOY-7`](https://www.qrz.com/db/K6BOY) sent 272 packets in the last 30 days, [`KO6PAX-7`](https://www.qrz.com/db/KO6PAX) sent 187. Counting every SSID those two callsigns use, that is 438 and 399 packets. Twenty-eight distinct stations have messaged the bot through this gateway since August. This is not an attack, not a botnet, and not a wave of automation ruining ham radio. It is two neighbours having fun, plus a global game that happens to be reachable from a local radio.

## The multiplier

Here is the part that should make every gateway operator sit up: a single logical "hello" is not a single packet on the air.

APRS on two metres runs at 1,200 baud on one shared frequency. A TNC listens for a clear channel and then transmits blind: there is no collision detection, no retransmission on collision, no scheduler and no priority. Airtime is the resource, and every packet is a small tax on everyone listening. That was fine for decades because the traffic was light and bursty. Message traffic behaves differently, because it is conversational and because it multiplies.

I measured the multiplier at my own receiver. Over 30 days:

| Traffic type | Physical RF packets | Logical units | Amplification |
|---|---:|---:|---:|
| Message packets | 1,372 | 1,096 | 1.25x |
| Position packets | 80,785 | 57,037 | 1.42x |

The position number is the cleaner one. Beacons are sent blind and never retried, so every extra physical copy is pure network duplication: **roughly 30 percent of all position RF copies heard here were redundant repeats of something already heard another way.**

The mechanism, concretely, from the corpus. On 17 September 2026 between 14:00 and 14:05 UTC, one station made a single contact attempt:

```
14:00:23  K6BOY-7>APGRWO,WA6TOW-2*,WIDE1*,WIDE2-1::OTA      :CHASE KJ5IMV-2{110
14:00:25  K6BOY-7>APGRWO,WA6TOW-2*,WIDE1*,BKELEY*,WIDE2*::OTA      :CHASE KJ5IMV-2{110
14:02:11  K6BOY-7>APGRWO,WA6TOW-2*,WIDE1*,WIDE2-1::OTA      :CHASE KJ5IMV-2{111
14:02:13  K6BOY-7>APGRWO,WA6TOW-2*,WIDE1*,BKELEY*,WIDE2*::OTA      :CHASE KJ5IMV-2{111
14:04:55  K6BOY-7>APGRWO,WA6TOW-2*,WIDE1*,WIDE2-1::OTA      :CHASE KJ5IMV-2{112
14:04:56  K6BOY-7>APGRWO,WA6TOW-2*,WIDE1*,BKELEY*,WIDE2*::OTA      :CHASE KJ5IMV-2{112
```

Six RF copies of one contact attempt at one receiver, over four and a half minutes.

Read the frames and the whole post collapses into one picture. The message sequence number `{110`, `{111`, `{112` shows **three** logical transmissions: the app retried twice, about two minutes apart, because no acknowledgement came back. Each of those three was then heard **twice** here, because two different digipeater chains repeated the same frame and this IGate is in earshot of both. `WA6TOW-2*` appears in every path, having consumed its hop first; then the frame forked, one branch riding the generic `WIDE2-1` alias and the other going through `BKELEY` and consuming its own `WIDE2`. Same packet, two families of relays, two receptions.

Three logical messages, six physical receptions, at one receiver. Every other gateway that heard `K6BOY-7` that morning did its own version of the same thing.

![One CHASE, retried twice, heard six times at one station through two digipeater chains.](/public/images/aprs-amplification.webp)
![The OTA topology. Host and chaser are on RF; the bot lives on APRS-IS. One message from the bot fans out to every gateway that has heard the target station, and each gateway keys its own radio.](/public/images/aprs-topology-ota.webp)
The relay hops over 30 days tell the same story: [`WA6TOW-2`](https://aprs.fi/info/a/WA6TOW-2) 868, `WIDE2` 797, [`BKELEY`](https://aprs.fi/info/a/BKELEY) 616, `WIDE1` 600, `WIDE2-1` 469, [`WR6ABD`](https://aprs.fi/info/a/WR6ABD) 432, all of them doing exactly their jobs in the path of traffic that is mostly a game. Those are relay elements counted across the 1,372 message frames, so a single frame with two used hops contributes two.

None of this is congestion collapse, and the mitigation is already in place on both sides. The bot rate-limits hard, as above. My IGate was already limiting too: `IGTXLIMIT 20 60` caps internet-to-RF at 20 per minute and 60 per five minutes, and many `[ig>tx]` events produced few actual transmissions, which means the limiter was shedding load. Tightening it to `10 30` is the one further knob, at the cost of dropping real messages. I left it alone.

## How to reproduce this

"How do you know it was 1.42x and not just more beacons" decides whether any of this stands up, so here is the recipe.

**Capture.** Log every packet your IGate demodulates with a receive timestamp and the full raw AX.25 frame. Dire Wolf's log, a KISS interface, or graywolf's packet log all produce this. Land it in Postgres or SQLite as it arrives; a local database of the last few months is the entire asset.

```sql
CREATE TABLE rf_packets (
  id       BIGSERIAL PRIMARY KEY,
  rx_utc   TIMESTAMPTZ NOT NULL,
  rx_epoch BIGINT NOT NULL,
  src      TEXT NOT NULL,   -- source callsign + SSID
  dst      TEXT NOT NULL,   -- destination field, abused as a device banner
  path     TEXT NOT NULL,   -- digipeater path, asterisks intact
  info     TEXT NOT NULL,   -- APRS information field
  raw      TEXT NOT NULL
);
CREATE INDEX ON rf_packets (rx_epoch);
CREATE INDEX ON rf_packets (src);
```

**Classify.** A message packet's information field starts with `:`. A position starts with `!`, `=`, `/`, `@`, or `;` (object). Weather is a position plus an appendix, or the positionless `_MMDDHHMM` form. A one-line classification on the first character goes a long way; you do not need a full [APRS protocol](http://www.aprs.org/doc/APRS101.PDF) parser.

**Separate logical from physical, which is the whole trick.** Group by `(source, message body)` and collapse rows whose timestamps fall inside a short window, because a client retry after a missing ack is a *different logical message*, while a digipeater repeat is the same one. About a minute works. Then:

```
amplification = COUNT(physical rows) / COUNT(collapsed groups)
```

Run the same collapse on position reports by `(source, info minus timestamp)` with a tighter window to get the 1.42x figure independently. Positions are never retried, so any extra copy is pure duplication, which makes it a good cross-check on the message number.

**And read the path field.** When one `(source, body)` group contains frames whose consumed paths differ, you have caught a redundant repeat through a second chain, citable verbatim, exactly as in the six-frame block above.

What this measures that a grep of your own transmit log cannot: a grep tells you what your station *sent*. The database tells you what the *network* did, how many physical copies existed for how few logical acts, which relays carried them, and how the load is spread across the people in your coverage area.

## The other thing that changed: the platform

There is a second story underneath this one, and it is the part other operators can act on directly.

On 13 September 2026 I started moving this station off DigiPi plus Dire Wolf and onto [graywolf](https://github.com/chrissnell/graywolf), on the same Raspberry Pi Zero 2 W. Its beacons now carry the [`APGRWO`](https://github.com/aprsorg/aprs-deviceid) TOCALL, so the Dire Wolf configuration quoted earlier is historical.

Two reasons.

**Performance.** A Pi Zero 2 W is not a powerful computer, and the software modem is the whole ballgame. Graywolf's modem is written in Rust and includes a port of the AFSK demodulator from [Dire Wolf](https://github.com/wb2osz/direwolf) by [WB2OSZ](https://www.qrz.com/db/WB2OSZ), with the decision-feedback AGC and hard-limiter correlator techniques credited to [Ion Todirel (W7ION)](https://www.qrz.com/db/W7ION) and his [libmodem](https://github.com/iontodirel/libmodem). AX.25 decoding, the APRS operations (beacons, digipeater, iGate) and the web API are a service written in Go, with a Svelte front end. The project's README claims its demodulator beats Dire Wolf's best mode (`-P AD+`) on every track of the [WA8LMF TNC test CD](http://www.wa8lmf.net/TNCtest/) while using about 5 percent of a Raspberry Pi 5, and describes the modem itself as using about 19 percent of a single CPU core on a Pi 5. Those are the maintainer's benchmarks, not independent ones, so treat them as claims, but the direction is right: it decodes marginally more and costs a fraction of what the old stack did on very small hardware. It is also a single binary with one SQLite config database, a systemd unit, and packages for Debian/Ubuntu, RHEL, Arch, Windows, and macOS, which removes a whole layer of hand-edited YAML and sound-card plumbing.

**Messaging.** This is the reason that actually mattered to me. If you run an IGate that relays a game's worth of APRS messages, you eventually want to *look* at them. Dire Wolf is a modem, a digipeater, and an IGate; to read and send messages as a human I was fighting Dire Wolf plus a separate application. Graywolf ships the messaging client. Per its [messaging handbook](https://github.com/chrissnell/graywolf/blob/main/docs/handbook/messaging.html), outbound messages go as APRS11 text packets over RF with an APRS-IS fallback; direct messages get auto-ACK and reply-ack matching, retransmitting until acknowledged or the retry budget runs out; messages longer than the 67-byte APRS cap are split into numbered segments and reassembled on receive; and tactical broadcast chats handle one-to-many group traffic with no acks. Every APRS text frame the station hears, whether it arrived over the air or through APRS-IS, appears as a message in the matching conversation.

![Graywolf's dashboard on the same Pi Zero 2 W: receive and transmit counters, the live packet stream, and the station's own position.](/public/images/aprs-graywolf-dashboard.webp)
After months of watching my radio relay contacts I had never seen, having a real inbox on the station was the difference between running a relay and understanding it. The corpus has KK6DCI-2 sending its first `CHASE` frame on 13 September 2026 at 20:35 UTC, the same day as the platform move, now tagged `APGRWO`; the DigiPi frames from that morning are in there too, under the `APDR16` TOCALL. Draw your own conclusions.

The configuration carried over cleanly: PTT via GPIO, IGate server `noam.aprs2.net`, IS-to-RF path `WIDE1-1,WIDE2-1`, and the same `b/KK6DCI` filter I had arrived at the hard way, now expressed in graywolf's iGate settings.

## What I would tell another gateway operator

**Do not assume you broke something, but do check whether you did.** The answer is often both. My first fix was correct, necessary, and only half the story.

**Audit your message filter for distance clauses.** If you have `t/m/.../<radius>`, the radius is around the station named in the filter, in kilometres, not a limit on the message destination. It will relay traffic for stations you have never heard. Default heard-station gating is the safe behaviour. If you want a safety net, size it to fill-in range, roughly 30 to 40 km (twenty to twenty-five miles).

**Know your transmit limit, and watch it fire.** `IGTXLIMIT` is the rate cap on everything you pull from APRS-IS to RF. Keep it where it is unless volume itself, not correctness, becomes a problem, because tightening it drops real messages along with the noisy ones.

**Identify the counterparty before you get annoyed.** Query your own log for top senders and top addressees. In my case two callsigns and one bot explained nearly all of it, and knowing that changed my assessment from "something is wrong" to "two people are playing a game."

**Remember a new service can appear overnight.** The channel has no changelog. The first OTA packet here arrived on 16 August and it was a majority of message traffic by September. Monthly monitoring means you are reading history.

**Understand how much you actually control, and how much you do not.** Your filter, your transmit limit, your digipeat rules, and whether you run at all. Not who shows up. A gateway is shared infrastructure that happens to live in your garage, and the demand on it is set by whoever is operating within earshot and whatever services they choose to talk to. The traffic that keyed my radio was mostly not caused by my settings and mostly could not have been fixed by them.

## A new game is here, and more might be coming

APRS is a forty-year-old protocol: unconnected [AX.25](https://files.tapr.org/tech_docs/AX25/AX25.2.2.1997.pdf), 67-byte text messages, ack-and-retry, digipeaters repeating frames so the next hop hears them. Nobody designed it for a real-time multiplayer game with a leaderboard and an API.

And yet there it is, running on the same primitive that carried email over the air in the 1980s. The chasers showing up in my local RF traffic include Germany ([`DL7PJ`](https://www.qrz.com/db/DL7PJ)), Romania ([`YO3SP`](https://www.qrz.com/db/YO3SP)), and Borneo ([`9W3FAB`](https://www.qrz.com/db/9W3FAB)), all trying to work two operators twenty miles from my antenna. Their callsigns appear in 49, 28, and 47 frames of my corpus respectively, because a message addressed to a local station gets gated out over local RF no matter where the sender is sitting. My 50-watt radio in a garage in [Benicia](https://en.wikipedia.org/wiki/Benicia,_California) is a node in that game whether or not I ever play it.

Bot traffic on APRS is decades old. WXBOT, email and SMS gateways, Winlink: all of it predates this. What is new is the *form*. A real-time game with a global player base, arriving with no announcement and no negotiation with the locals, rate-limiting itself where the old bots did not. Decentralized communication over a hybrid Internet/RF medium has never been easier or more fun!