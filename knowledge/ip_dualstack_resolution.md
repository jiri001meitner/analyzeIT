# IP Address Retention and Dual-Stack (IPv4 & IPv6) Resolution

This document describes how analyzeIT processes client IP addresses for
geolocation and explains the dual-stack probe mechanism to capture both IPv4
and IPv6 addresses from visitors.

## 1. IP Capture and Configurable Retention

Capturing the client IP address at ingestion is necessary for geolocation
lookup (Country, Region, City, and ASN / ISP).

analyzeIT provides configurable IP retention modes (via `IP_STORAGE_MODE`),
giving administrators full control over privacy compliance:

- `full`: Stores the complete IPv4 / IPv6 address (allows displaying full IPs
  in the administrative UI, similar to Matomo).
- `mask_1_byte`: Masks the last octet of IPv4 (e.g., `192.168.1.0/24`) and
  64 bits of IPv6.
- `mask_2_bytes`: Masks the last two octets (e.g., `192.168.0.0/16`).
- `hash_only`: Retains only a salted one-way hash (strict GDPR compliance).

Regardless of the storage mode, GeoIP resolution is executed immediately during
ingestion before masking or hashing is applied.

## 2. Dual-Stack Browser Behavior (Happy Eyeballs)

Modern operating systems and browsers implement RFC 8305 (Happy Eyeballs). A
dual-stack client will choose either IPv4 or IPv6 for an HTTP connection based
on connection latency. A standard tracking request therefore exposes only the
chosen protocol.

## 3. Dual-Stack IP Probing Architecture

To discover both the visitor's public IPv4 and IPv6 addresses and associate
them with the same session, analyzeIT supports dual-stack probing.

### Topology and Routing

Rather than running multiple backend services, a single Ingestion Service
handles all traffic. The separation is achieved at the DNS and reverse proxy
layer:

```text
               ┌──► ipv4.analytics.domain (DNS A only)    ──┐
[ Browser JS ] │                                            │ (Unix Socket)
               ├──► ipv6.analytics.domain (DNS AAAA only) ─┼──► [ Ingestion ]
               │                                            │
               └──► analytics.domain (Dual-stack A + AAAA) ─┘
```

1. **DNS Records**:
   - `analytics.domain`: Both A and AAAA records (standard primary endpoint).
   - `ipv4.analytics.domain`: Only an A record pointing to host IPv4.
   - `ipv6.analytics.domain`: Only an AAAA record pointing to host IPv6.
1. **Reverse Proxy (Caddy)**:
   - Listens on all three hostnames.
   - Proxies all incoming requests over the same Unix Domain Socket
     (`/var/run/analyzeit.sock`) to the Go Ingestion Service.

### Client-Side Execution and Resilience

1. **Primary Hit**: The browser tracker sends the full pageview payload
   (URL, referer, screen metrics) to the primary dual-stack endpoint.
1. **Asynchronous Probe**: If enabled, the tracking script performs an
   asynchronous, non-blocking ping to the complementary IP endpoint:
   - Uses `fetch` with `keepalive: true` or `navigator.sendBeacon`.
   - Constrained by a strict timeout (e.g., 1,500 ms) so clients without IPv6
     connectivity experience zero delay.
   - Passes the same `visitor_id` and `session_id`.
1. **Aggregation**: The Batch Worker merges both IP records into the visitor's
   session profile in Databend.

## 4. Threat Intelligence: IPv6 VPN Leak Detection

A major operational benefit of dual-stack probing is unmasking visitors behind
misconfigured or partial VPN tunnels (IPv6 leak vulnerability):

1. **The IPv6 Leak Phenomenon**: Many commercial VPN clients route only IPv4
   traffic (`0.0.0.0/0`) through the encrypted tunnel while leaving local IPv6
   interfaces unrouted (`::/0`) or enabled on the host network.
1. **ASN and Geolocation Discrepancy**:
   - **IPv4**: Originates from a known datacenter or commercial VPN provider
     (e.g., M247, Datacamp, DigitalOcean, NordVPN).
   - **IPv6**: Originates from the visitor's genuine residential internet
     service provider (e.g., Deutsche Telekom, Comcast, O2).
1. **Unmasking True Location**: By comparing the ASNs and GeoIP attributes of
   both addresses within the same session, analyzeIT can flag sessions with
   `vpn_leak: true`, exposing the visitor's actual physical location.
1. **Fraud and Bot Scoring**: Discrepancies between IPv4 and IPv6 routes serve
   as high-confidence heuristics for detecting click fraud, scrapers, and ad
   manipulation.

## 5. Network Topology: DS-Lite and CGNAT Classification

In modern ISP deployments (e.g., Vodafone, Deutsche Telekom), many subscribers
operate under Dual-Stack Lite (DS-Lite, RFC 6333) or Carrier-Grade NAT (CGNAT):

1. **The DS-Lite Mechanism**: The subscriber router receives a native public
   IPv6 prefix, but no dedicated public IPv4 address. Outbound IPv4 traffic
   is tunneled through IPv6 to an Address Family Transition Router (AFTR),
   where CGNAT assigns a shared public IPv4 pool address.
1. **Visitor Collision in Traditional Analytics**: Legacy platforms relying on
   IPv4 group thousands of unrelated residential subscribers into a single
   IP, causing false session mergers and inaccurate unique visitor counts.
1. **Dual-Stack Probe Detection**: By collecting both protocols, analyzeIT
   observes multiple distinct residential IPv6 prefixes mapping to the same
   public IPv4 AFTR gateway.
1. **Connection Profile Tagging**: analyzeIT can categorize visitor traffic:
   - `native_dual_stack`: Dedicated/unshared public IPv4 and native IPv6.
   - `ds_lite`: Native IPv6 paired with a shared ISP CGNAT IPv4 address.
   - `ipv4_only` / `ipv6_only`: Single-protocol connectivity.

## 6. NAT Detection: Residential NAT vs. CGNAT vs. Native End-to-End

analyzeIT determines whether a visitor is operating behind a NAT gateway
through passive correlation and optional client-side probing:

1. **IPv4 vs. IPv6 End-to-End Inherent Asymmetry**: In IPv4, virtually all
   residential and corporate devices sit behind NAPT (NAT44), translating
   private RFC 1918 space to a public WAN IP. In IPv6, devices communicate
   natively using end-to-end globally unique addresses (GUA) without address
   translation.
1. **Passive Correlation (Prefix Cardinality)**: A unique public IPv4 address
   associated exclusively with a single residential IPv6 `/64` or `/56` prefix
   indicates a standard customer-premises router (CPE) performing local NAT44.
   Conversely, an IPv4 address shared across dozens of different subscriber
   IPv6 prefixes indicates an ISP Carrier-Grade NAT (CGNAT / NAT444) middlebox.
1. **Active Dual-Stack STUN Probe (IPv4 & IPv6 WebRTC)**: The tracking script
   instantiates an `RTCPeerConnection` with STUN endpoints configured for both
   families (`ipv4.stun.domain:3478` and `ipv6.stun.domain:3478`). UDP-based
   STUN requests frequently bypass HTTP and SOCKS browser proxies, unmasking
   true residential IPv4 and IPv6 WAN endpoints. Discrepancies between host
   candidates and server-reflexive candidates classify the NAT topology.
1. **Passive TTL Hop-Count Heuristic**: Initial TCP Time-to-Live (TTL) values
   differ predictably by operating system (Windows: 128, Linux/macOS: 64).
   Deviations provide an estimate of upstream routing hops and intermediate
   customer-premises gateways.
