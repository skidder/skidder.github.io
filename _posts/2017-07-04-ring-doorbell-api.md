---
layout: post
title: The Ring Smart Doorbell API
---

# Background
The [Ring smart doorbell](https://ring.com/videodoorbells) does far more than your ordinary wired doorbell; it's able to deliver audio & video recordings in real-time over a WiFi link connected to the Internet. When your doorbell is pressed or motion is detected, the doorbell automatically begins streaming video to the Ring servers. Ring will send a push notification to your phone and archive the video, where it can be downloaded or viewed through the Ring website or app.

I purchased the [original Ring doorbell](https://ring.com/video-doorbell) in December 2015. It has worked pretty well, both as a way to get notifications of guests or deliveries at the house and as a effective deterrant to people snooping around the house.

With the increasing number of Internet-of-Thing (IoT) devices ending up on our home and business networks, it's important to understand how they behave and ensure that they follow industry best-practices. The security of other devices on our networks and our privacy are at stake.

# Hardware Setup

My Ring doorbell is connected to an Apple Airport Express that's configured as a 802.11G WiFi bridge, delegating DHCP leases and router functions to my Ubiquiti router. The Ring doorbell is the only device on this particular WiFi access-point.

To analyze the Ring doorbell traffic, I placed a [Netgear DS104 Ethernet hub](https://www.netgear.com/support/product/DS104.aspx) in-line between the Airport Express and the Ethernet switch that connects to the rest of my home network. Next, I connected my laptop to a port on the hub; my laptop is now able to view all traffic to-and-from the Airport Express (namely, the Ring doorbell). Ethernet hubs are infrequently used these days. Hubs broadcast all packets to all ports; this is extremely inefficient, but can be very useful for troubleshooting network problems and reverse-engineering.

# Capturing Traffic

I used Wireshark to capture TCP & UDP traffic to/from the IP address used by the Ring doorbell.

# What's it Saying?

Here I'll document some of the messages I've seen the Ring doorbell exchange with the Ring servers.

## Polling for requests to watch live video
The Ring doorbell polls the Ring API every 17 seconds to check whether to stream live video. If no request has been made then an `HTTP 404` response code is returned.

There are a couple things worth noting here:
 * The request is sent over plain HTTP (no encryption) to http://vod.ring.com/doorbots_api/vod/ready
 * Authentication is performed using digest auth over a non-encrypted link, very bad.

### Request where server doesn't need streaming video
```
GET /doorbots_api/vod/ready HTTP/1.1
Host: vod.ring.com
Authorization: Basic <removed>
Content-Length: 0
X-RingDeviceType: doorbell

HTTP/1.1 404 Not Found
Content-Type: text/plain
Date: Mon, 03 Jul 2017 16:17:08 GMT
Server: nginx
Content-Length: 0
Connection: keep-alive

```

### Request where server responds with a request to stream video
```
GET /doorbots_api/vod/ready HTTP/1.1
Host: vod.ring.com
Authorization: Basic <removed>
Content-Length: 0
X-RingDeviceType: doorbell

HTTP/1.1 200 OK
Content-Type: text/plain
Date: Mon, 03 Jul 2017 16:18:12 GMT
Server: nginx
Content-Length: 0
Connection: keep-alive

```

## Streaming Live Video Upon Request from App
If the polling response indicates that live video should be sent, then the Ring doorbell will send a request to the Ring servers. The request indicates that the [Session Initiation Protocol (SIP)](http://www.cs.columbia.edu/sip/) be used. SIP is often used in real-time conference connection between client and server. That analogy works well in the scenario, where the doorbell and Ring servers are participants in a video conference.

### Initiating Session
When the `/doorbots_api/vod/ready` request responds with a `200 OK`, the doorbell will `POST` to the `/doorbots_api/vod` endpoint to get details needed to establish a video streaming session. The response is a Base64-encoded binary blob that I didn't recognize.

```
POST /doorbots_api/vod?api_version=3 HTTP/1.1
Host: fw.ring.com
Authorization: Basic <removed>
Content-Length: 12
X-RingDeviceType: doorbell
Content-Type: application/x-www-form-urlencoded

protocol=sip

HTTP/1.1 200 OK
Cache-Control: max-age=0, private, must-revalidate
Content-Type: text/plain; charset=utf-8
Date: Mon, 03 Jul 2017 16:18:12 GMT
ETag: W/"a91a7b77bcfaaa5a298decf9d5b6624b"
Server: nginx
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
X-Request-Id: 7665c610-59a4-42c6-b368-3e7f349af5f2
X-XSS-Protection: 1; mode=block
Content-Length: 1241
Connection: keep-alive

<Base64-encoded binary data>
```

### SIP
The following SIP messages were sent over a UDP connection held open for 14 seconds while I was streaming video to the Ring smartphone app.

I enjoyed the Easter egg that Ring created with the User-Agent string `Hi Zero Cool. Job?'); DROP TABLE servers;--`. A shout out to the hacker handle [Zero Cool from the movie Hackers](https://en.wikipedia.org/wiki/Hackers_(film)#Plot), and an attempt at SQL injection to drop a database table named `servers`.

```
INVITE sip:3pqc8qa-201003n4ifk7iq@52.53.75.176 SIP/2.0
Via: SIP/2.0/UDP <LAN IP>:15063;rport;branch=z9hG4bK677133703
From: <sip:<Ring MAC Address>@ring.com>;tag=RBT00000000F3A6234A
To: <sip:3pqc8qa-201003n4ifk7iq@52.53.75.176>
Call-ID: NTZNHCUGMGCYIORRLXCQYEGUEKKKXJURG00000000F3A6234A
CSeq: 1 INVITE
Contact: sip:<Ring MAC Address>@<LAN IP>
Content-Type: application/sdp
Allow: INVITE, ACK, CANCEL, OPTIONS, BYE, MESSAGE
Max-Forwards: 70
User-Agent: Device/dpdv2/1.8.90
Subject: Ding
X-SSRC-A: 0225916571
X-SSRC-V: 0443028147
Content-Length:   387

v=0
o=sip:<Ring MAC Address>@ring.com 285007105 0 IN IP4 <LAN IP>
s=Ding
c=IN IP4 <LAN IP>
t=0 0
m=audio 51506 RTP/AVP 0
a=rtpmap:0 PCMU/8000
a=rtcp-mux
m=video 51504 RTP/AVP 97
a=rtpmap:97 H264/90000
a=rtcp-mux
a=sendonly

SIP/2.0 100 Trying
Via: SIP/2.0/UDP <LAN IP>:15063;rport=15063;branch=z9hG4bK677133703;received=71.202.56.97
From: <sip:<Ring MAC Address>@ring.com>;tag=RBT00000000F3A6234A
To: <sip:3pqc8qa-201003n4ifk7iq@52.53.75.176>
Call-ID: NTZNHCUGMGCYIORRLXCQYEGUEKKKXJURG00000000F3A6234A
CSeq: 1 INVITE
User-Agent: Hi Zero Cool. Job?'); DROP TABLE servers;--
Content-Length: 0

SIP/2.0 183 Session Progress
Via: SIP/2.0/UDP <LAN IP>:15063;rport=15063;branch=z9hG4bK677133703;received=71.202.56.97
From: <sip:<Ring MAC Address>@ring.com>;tag=RBT00000000F3A6234A
To: <sip:3pqc8qa-201003n4ifk7iq@52.53.75.176>;tag=ya4jjj47B0gZm
Call-ID: NTZNHCUGMGCYIORRLXCQYEGUEKKKXJURG00000000F3A6234A
CSeq: 1 INVITE
Contact: <sip:3pqc8qa-201003n4ifk7iq@52.53.75.176:15063;transport=udp>
User-Agent: Hi Zero Cool. Job?'); DROP TABLE servers;--
Accept: application/sdp
Allow: INVITE, ACK, BYE, CANCEL, OPTIONS, MESSAGE, INFO, UPDATE, NOTIFY
Supported: path, replaces
Allow-Events: talk, hold, conference, refer
Content-Type: application/sdp
Content-Disposition: session
Content-Length: 330
Remote-Party-ID: "3pqc8qa-201003n4ifk7iq" <sip:3pqc8qa-201003n4ifk7iq@52.53.75.176>;party=calling;privacy=off;screen=no

v=0
o=Wantajobinstead? 1499054147 1499054148 IN IP4 52.53.75.176
s=Wantajobinstead?
c=IN IP4 52.53.75.176
t=0 0
m=audio 44546 RTP/AVP 0
a=rtpmap:0 PCMU/8000
a=ptime:20
a=rtcp-mux
a=rtcp:44546 IN IP4 52.53.75.176
m=video 45360 RTP/AVP 97
a=rtpmap:97 H264/90000
a=recvonly
a=rtcp-mux
a=rtcp:45360 IN IP4 52.53.75.176
SIP/2.0 200 OK
Via: SIP/2.0/UDP <LAN IP>:15063;rport=15063;branch=z9hG4bK677133703;received=71.202.56.97
From: <sip:<Ring MAC Address>@ring.com>;tag=RBT00000000F3A6234A
To: <sip:3pqc8qa-201003n4ifk7iq@52.53.75.176>;tag=ya4jjj47B0gZm
Call-ID: NTZNHCUGMGCYIORRLXCQYEGUEKKKXJURG00000000F3A6234A
CSeq: 1 INVITE
Contact: <sip:3pqc8qa-201003n4ifk7iq@52.53.75.176:15063;transport=udp>;isfocus
User-Agent: Hi Zero Cool. Job?'); DROP TABLE servers;--
Allow: INVITE, ACK, BYE, CANCEL, OPTIONS, MESSAGE, INFO, UPDATE, NOTIFY
Supported: path, replaces
Allow-Events: talk, hold, conference, refer
Content-Type: application/sdp
Content-Disposition: session
Content-Length: 330
Remote-Party-ID: "3pqc8qa-201003n4ifk7iq" <sip:3pqc8qa-201003n4ifk7iq@52.53.75.176>;party=calling;privacy=off;screen=no

v=0
o=Wantajobinstead? 1499054147 1499054148 IN IP4 52.53.75.176
s=Wantajobinstead?
c=IN IP4 52.53.75.176
t=0 0
m=audio 44546 RTP/AVP 0
a=rtpmap:0 PCMU/8000
a=ptime:20
a=rtcp-mux
a=rtcp:44546 IN IP4 52.53.75.176
m=video 45360 RTP/AVP 97
a=rtpmap:97 H264/90000
a=recvonly
a=rtcp-mux
a=rtcp:45360 IN IP4 52.53.75.176
ACK sip:3pqc8qa-201003n4ifk7iq@52.53.75.176 SIP/2.0
Via: SIP/2.0/UDP <LAN IP>:15063;rport;branch=z9hG4bK48801564
From: <sip:<Ring MAC Address>@ring.com>;tag=RBT00000000F3A6234A
To: <sip:3pqc8qa-201003n4ifk7iq@52.53.75.176>;tag=ya4jjj47B0gZm
Call-ID: NTZNHCUGMGCYIORRLXCQYEGUEKKKXJURG00000000F3A6234A
CSeq: 1 ACK
Contact: sip:<Ring MAC Address>@<LAN IP>
Allow: INVITE, ACK, CANCEL, OPTIONS, BYE, MESSAGE
Max-Forwards: 70
User-Agent: Device/dpdv2/1.8.90
Subject: Ding
X-SSRC-A: 0225916571
X-SSRC-V: 0443028147
Content-Length:     0

!BYE sip:<Ring MAC Address>@<LAN IP> SIP/2.0
Via: SIP/2.0/UDP 52.53.75.176:15063;rport;branch=z9hG4bKX0pFHyc88ZZDQ
Max-Forwards: 70
From: <sip:3pqc8qa-201003n4ifk7iq@52.53.75.176>;tag=ya4jjj47B0gZm
To: <sip:<Ring MAC Address>@ring.com>;tag=RBT00000000F3A6234A
Call-ID: NTZNHCUGMGCYIORRLXCQYEGUEKKKXJURG00000000F3A6234A
CSeq: 109213289 BYE
User-Agent: Hi Zero Cool. Job?'); DROP TABLE servers;--
Allow: INVITE, ACK, BYE, CANCEL, OPTIONS, MESSAGE, INFO, UPDATE, NOTIFY
Supported: path, replaces
Reason: Q.850;cause=16;text="NORMAL_CLEARING"
Content-Length: 0

SIP/2.0 200 OK
Via: SIP/2.0/UDP 52.53.75.176:15063;rport;branch=z9hG4bKX0pFHyc88ZZDQ
From: <sip:3pqc8qa-201003n4ifk7iq@52.53.75.176>;tag=ya4jjj47B0gZm
To: <sip:<Ring MAC Address>@ring.com>;tag=RBT00000000F3A6234A
Call-ID: NTZNHCUGMGCYIORRLXCQYEGUEKKKXJURG00000000F3A6234A
CSeq: 109213289 BYE
Contact: sip:<Ring MAC Address>@<LAN IP>
User-Agent: Device/dpdv2/1.8.90
Content-Length: 0

SIP/2.0 200 OK
Via: SIP/2.0/UDP 52.53.75.176:15063;rport;branch=z9hG4bKX0pFHyc88ZZDQ
From: <sip:3pqc8qa-201003n4ifk7iq@52.53.75.176>;tag=ya4jjj47B0gZm
To: <sip:<Ring MAC Address>@ring.com>;tag=RBT00000000F3A6234A
Call-ID: NTZNHCUGMGCYIORRLXCQYEGUEKKKXJURG00000000F3A6234A
CSeq: 109213289 BYE
Contact: sip:<Ring MAC Address>@<LAN IP>
User-Agent: Device/dpdv2/1.8.90
Content-Length: 0

```

## Responding to Doorbell Presses
This is almost identical to the request to stream live video on demand, except it's sent to the `dings` path:

```
POST /doorbots_api/dings?api_version=3 HTTP/1.1
Host: fw.ring.com
Authorization: Basic <removed>
Content-Length: 12
X-RingDeviceType: doorbell
Content-Type: application/x-www-form-urlencoded

protocol=sip

HTTP/1.1 200 OK
Cache-Control: max-age=0, private, must-revalidate
Content-Type: text/plain; charset=utf-8
Date: Tue, 04 Jul 2017 17:45:51 GMT
ETag: W/"b3914fa92a693d13bd3401dd306d2915"
Server: nginx
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
X-Request-Id: dc93b10b-7e6d-43ae-9f04-32db1e6c2060
X-XSS-Protection: 1; mode=block
Content-Length: 1153
Connection: keep-alive

<Base64-encoded binary data>
```

The SIP session is then initiated and streaming happens over RTSP in the same manner as with live video on demand.

## Responding to Motion
Again, this is almost identical to the doorbell-pressed and vod requests, except it goes to the `motions` path:

```
POST /doorbots_api/motions?api_version=3 HTTP/1.1
Host: fw.ring.com
Authorization: Basic <removed>
Content-Length: 12
X-RingDeviceType: doorbell
Content-Type: application/x-www-form-urlencoded

protocol=sip

HTTP/1.1 200 OK
Cache-Control: max-age=0, private, must-revalidate
Content-Type: text/plain; charset=utf-8
Date: Tue, 04 Jul 2017 17:47:03 GMT
ETag: W/"fc2f1a4674071018e906066d265082e5"
Server: nginx
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
X-Request-Id: 2689a0ef-d365-49e6-b646-6e96d584642b
X-XSS-Protection: 1; mode=block
Content-Length: 1369
Connection: keep-alive

<Base64-encoded binary data>
```

## Comparison to Earlier Protocol Versions
I did a packet capture of the Ring doorbell events in November 2016 and compared some of the messages to what I observed recently (July 2017). Clearly changes have been made.

The request from the doorbell to the Ring API upon detecting motion used to include a JSON-encoded response that was in the clear:

### Motion

```shell
curl -X POST \
  'http://fw.ring.com/doorbots_api/motions' \
  -H 'authorization: Basic <removed>' \
  -H 'cache-control: no-cache' \
  -H 'content-type: application/x-www-form-urlencoded' \
  -H 'x-ringdevicetype: doorbell'
```

#### Response
```
{
    "motion": {
        "id": "4109110674",
        "state": "ringing",
        "motion_snooze": 5,
        "sip_server_ip": "52.9.18.238",
        "sip_server_port": "15063",
        "sip_server_tls": "false",
        "sip_session_id": "3qeo3ci-201003n4ifk7im",
        "sip_server_tls_port": "15064",
        "sip_from": "sip:<Ring MAC Address>@ring.com",
        "sip_to": "sip:3qeo3ci-201003n4ifk7im@52.9.18.238",
        "button_press_path": "/doorbots_api/motions/4109110674/button_pressed",
        "mic_volume": 11,
        "voice_volume": 11,
        "stream_profile": 2,
        "udp_ping_server": null,
        "udp_ping_port": null,
        "enable_recording": 1
    },
    "settings": {
        "utc_offset": "-07:00",
        "keep_alive": 15,
        "doorbell_volume": 8,
        "enable_chime": 1,
        "enable_vod": 1,
        "exposure_control": 2,
        "theft_alarm_enable": 0,
        "pir_sensitivity_1": 3,
        "pir_sensitivity_2": 5,
        "pir_sensitivity_3": 5,
        "pir_zone_enable": 24,
        "use_cached_domain": 0,
        "use_server_ip": 0,
        "server_domain": "fw.ring.com",
        "server_ip": null,
        "enable_log": 1,
        "keep_alive_ms": 10000,
        "force_dhcp": 0
    }
}
```

### Doorbell Pressed
```shell
curl -X POST \
  http://fw.ring.com/doorbots_api/dings \
  -H 'authorization: Basic <removed>' \
  -H 'cache-control: no-cache' \
  -H 'content-type: application/x-www-form-urlencoded' \
  -H 'x-ringdevicetype: doorbell'
```

#### Response
```
{
    "ding": {
        "id": "4108467263",
        "state": "ringing",
        "sip_server_ip": "52.52.104.21",
        "sip_server_port": "15063",
        "sip_server_tls": "false",
        "sip_server_tls_port": "15064",
        "sip_session_id": "3qe4f1v-201003n4ifk7ii",
        "sip_from": "sip:<Ring MAC Address>@ring.com",
        "sip_to": "sip:3qe4f1v-201003n4ifk7ii@52.52.104.21",
        "mic_volume": 11,
        "voice_volume": 11,
        "stream_profile": 2
    },
    "settings": {
        "utc_offset": "-07:00",
        "keep_alive": 15,
        "doorbell_volume": 8,
        "enable_chime": 1,
        "enable_vod": 1,
        "exposure_control": 2,
        "theft_alarm_enable": 0,
        "pir_sensitivity_1": 3,
        "pir_sensitivity_2": 5,
        "pir_sensitivity_3": 5,
        "pir_zone_enable": 24,
        "use_cached_domain": 0,
        "use_server_ip": 0,
        "server_domain": "fw.ring.com",
        "server_ip": null,
        "enable_log": 1,
        "keep_alive_ms": 10000,
        "force_dhcp": 0
    }
}
```

The doorbell is currently including the `?api_version=3` query parameter which likely changes how the response is formatted so that it's Base64-encoded and possibly encrypted.

## Suggestions
I'd like for all communications with the Ring servers to use HTTPS. It's far too easy for someone to sniff requests to the Ring servers, grab the digest-auth username (MAC address) and password, and script requests that appear to be coming from the device.

It also looks like the Ring doorbell rotates the password used in digest authentication, as it was different on the two days I most recently captured traffic. Perhaps they've got a one-time pad that they rotate each day. This would significantly limit the amount of time an attacker would be able to use the sniffed password. However, if they captured the password once, then it's fair to assume they'll be able to do it again.
