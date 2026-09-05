# Tor Network Detection and Visitor Tracking

This document outlines how analyzeIT identifies Tor network traffic, handles
anonymized visitors, and evaluates tracking capabilities within the Tor
ecosystem.

## 1. Tor Exit Node Identification

Incoming traffic from the public Tor network passes through designated exit
relays:

1. **Public Exit Lists**: The Tor Project publishes dynamic lists of all active
   exit node IP addresses (`check.torproject.org/torbulkexitlist`).
1. **Real-Time Classification**: During the Ingestion Service phase, incoming
   client IPs are compared against an in-memory bitmap or Bloom filter of known
   Tor exit nodes.
1. **Metric Tagging**: The event is tagged with `is_tor: true` and the exit
   node's country/ASN in Databend.

## 2. Official Tor Browser vs. Custom Proxy Leaks

Tracking resilience depends heavily on the client's software configuration:

### Scenario A: Official Tor Browser

The official Tor Browser implements rigorous anti-fingerprinting defenses:

- **Isolated Proxying**: All DNS queries and TCP connections route strictly
  through the SOCKS proxy.
- **WebRTC Disabled**: Prevents local interface and STUN candidate leaks.
- **Fingerprint Uniformity**: Normalizes screen resolutions (letterboxing),
  canvas outputs, audio context, and User-Agent headers.
- **Ephemeral Storage**: Cookies, cache, and LocalStorage are destroyed upon
  closing the tab or selecting "New Identity".

*Result*: Client IP address cannot be leaked through dual-stack probes.

### Scenario B: Unhardened Browsers via Tor SOCKS (e.g., Chrome/Brave)

When visitors manually configure a standard browser to proxy through Tor
(`127.0.0.1:9050`):

- **Dual-Stack Probe Bypass**: Standard browsers may route IPv4 through Tor while
  executing IPv6 requests directly over the physical network interface.
- **WebRTC Leaks**: Standard browsers allow STUN discovery, revealing genuine
  private and public IP addresses.

*Result*: analyzeIT's dual-stack probe and WebRTC checks unmask the visitor's
real residential IP and ISP.

## 3. In-Session and Behavioral Tracking in Tor

Even within the hardened Tor Browser, certain tracking mechanisms remain viable:

1. **In-Session State**: As long as a visitor remains on the site without
   closing the browser, in-memory JavaScript state (`session_id`) tracks their
   pageviews across the domain.
1. **Tor Circuit Rotation Correlation**: Tor switches circuits roughly every
   10 minutes, presenting a new exit node IP mid-session. Persistent in-memory
   connections (e.g., SSE or WebSockets) correlate the old and new exit IPs to
   the same visitor.
1. **Behavioral Biometrics**: Keystroke cadence, scroll momentum, and pointer
   acceleration patterns are physical human traits that operate independently
   of network anonymization layers.
