# Tracking Pixel and No-JavaScript Telemetry

This document describes the design, implementation, and capabilities of the
analyzeIT transparent 1x1 tracking pixel (`/pixel.gif`).

## 1. Concept and Ingestion Efficiency

While the primary tracking mechanism is a modern JavaScript collector,
analyzeIT provides an ultra-lightweight 1x1 transparent GIF endpoint:

- **Zero-Allocation Ingestion**: The transparent 1x1 GIF payload is precisely
  43 bytes. The Go Ingestion Service holds this byte array statically in memory
  (`var gif1x1 = []byte{...}`) and streams it instantly with HTTP 200.
- **Cache Invalidation**: Served with strict HTTP headers (`Cache-Control:
  no-store, no-cache, must-revalidate, max-age=0`) ensuring every pageview or
  email open triggers an authentic hit.

## 2. Key Use Cases

1. **HTML-Only Dual-Stack Probing (No JS Required)**:
   A website can embed two hidden image tags:

   ```text
   <img src="https://ipv4.analytics.domain/pixel.gif?vid=XYZ"
        width="1" height="1" alt="" style="display:none;" />
   <img src="https://ipv6.analytics.domain/pixel.gif?vid=XYZ"
        width="1" height="1" alt="" style="display:none;" />
   ```

   The browser executes independent network connections across both protocol
   stacks without executing a single line of JavaScript. This enables VPN leak
   and DS-Lite detection even for privacy-hardened environments.
1. **No-Script Fallback**:
   Allows measuring visitors who disable JavaScript or utilize aggressive
   script-blocking extensions:

   ```text
   <noscript>
     <img src="https://analytics.domain/pixel.gif?site_id=ABC&noscript=1"
          width="1" height="1" alt="" />
   </noscript>
   ```

1. **External Channel Telemetry**:
   Enables analytics in contexts where JavaScript execution is prohibited:
   - Email newsletter open rates.
   - Markdown documents (e.g., GitHub README badges).
   - RSS feeds and syndication readers.

## 3. Comparison: Tracking Pixel vs. JavaScript Collector

| Feature | Tracking Pixel (`/pixel.gif`) | JavaScript Tracker (`/hit`) |
| :--- | :--- | :--- |
| **JS Needed** | None (HTML image tag) | Required |
| **Payload Size** | 43 bytes (GIF) / 204 response | ~2-5 KB script + JSON POST |
| **Dual-Stack** | Supported (dual image tags) | Supported (async fetch) |
| **DOM Data** | Referer, User-Agent, IP only | Viewport, scroll, events |
| **WebRTC STUN** | Not possible | Full ICE candidate discovery |
