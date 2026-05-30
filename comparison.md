# Comparison — Captive Portal vs Squid Proxy vs SquidGuard

Three access control mechanisms configured in this lab. Each operates at a different layer, controls different things, and fails in different ways.

---

## At a Glance

| | Captive Portal | Squid Proxy | SquidGuard |
|-|---------------|-------------|-----------|
| **Purpose** | Authenticate users before granting network access | Proxy and log outbound web traffic | Filter content by category/domain |
| **OSI Layer** | Layer 7 (HTTP redirect) + Layer 3/4 (firewall rules) | Layer 7 (application proxy) | Layer 7 (URL/domain filter) |
| **What it controls** | Who can access the network | What traffic passes through the proxy | Which domains/categories are allowed |
| **Requires client config** | No (transparent redirect) | Yes (explicit mode) or No (transparent mode) | No (works through Squid) |
| **Encrypts traffic** | No | No | No |
| **Logs traffic** | Session start/end | Full request log (URL, IP, timestamp, bytes) | Blocked requests |
| **Works on HTTPS** | Partially (blocks, can't redirect cleanly) | Partially (CONNECT tunnels, not full inspection) | Partially (hostname only, not full URL) |
| **Bypassable by** | VPN, MAC spoof, HTTPS timing | Disabling proxy settings (explicit mode) | Direct IP, non-standard port, HTTPS (without SSL bump) |

---

## Captive Portal

**What it does:** Intercepts all outbound traffic from unauthenticated clients and redirects HTTP requests to a login page. After successful authentication, the firewall opens traffic for that client's IP/MAC.

**Best for:**
- Guest Wi-Fi (hotels, cafes, universities)
- Temporary network access with identity tracking
- Compliance scenarios where you need a record of who accessed the network and when

**What it does NOT do:**
- Encrypt traffic
- Filter content after authentication
- Prevent authenticated users from visiting any site
- Handle HTTPS cleanly without additional configuration

**Fails when:**
- Client uses a VPN before the portal fires
- Client spoofs the MAC of an authenticated device
- Browser only uses HTTPS and doesn't trigger HTTP redirect
- Admin leaves outbound DNS unrestricted (minor gap)

**Real-world usage:** Almost every public Wi-Fi network. Enterprise guest VLANs. Education networks for BYOD.

---

## Squid Proxy

**What it does:** Acts as a forward proxy — all client web requests pass through Squid before reaching the internet. Squid logs every request (URL, client IP, timestamp, bytes transferred, response code).

**Best for:**
- Traffic visibility and auditing
- Bandwidth management and caching
- Enforcing that all web traffic is logged
- Foundation layer that SquidGuard sits on top of

**What it does NOT do:**
- Authenticate users on its own (relies on captive portal or LDAP integration)
- Block content on its own (needs SquidGuard or similar)
- Inspect HTTPS body content without SSL bump

**Fails when:**
- Client is in explicit mode and simply removes the proxy setting
- Client uses a non-standard port Squid isn't configured to handle
- HTTPS traffic is not configured for inspection (SSL bump not enabled)

**Explicit vs Transparent mode:**

| | Explicit | Transparent |
|-|---------|------------|
| Client config needed | Yes | No |
| Bypassable by client | Yes (remove proxy setting) | No (firewall redirects) |
| HTTPS support | CONNECT tunnels only | Requires SSL bump |
| Setup complexity | Low | Medium-High |

**Real-world usage:** Corporate networks for web logging and compliance. ISPs for caching. Schools and libraries for content filtering.

---

## SquidGuard

**What it does:** Sits on top of Squid as a URL rewriter/filter. Checks every request against category blacklists (social networks, adult content, malware domains, etc.) and blocks matched categories before the connection is established.

**Best for:**
- Category-based content filtering (block social media, streaming, gambling, etc.)
- Compliance with acceptable use policies
- School/library environments

**What it does NOT do:**
- Work without Squid (it's a plugin, not a standalone tool)
- Inspect HTTPS content (only sees the hostname from the CONNECT request)
- Block based on page content — only domain/URL matching

**Fails when:**
- Client accesses blocked site by IP instead of domain
- Site uses HTTPS and SquidGuard is only matching HTTP URLs
- Blacklists are outdated (categories need periodic updates)
- Client bypasses Squid entirely (SquidGuard has no effect)

**Real-world usage:** pfSense/OPNsense deployments in schools, corporate environments, and small businesses as a simple content filter.

---

## How They Work Together

These three controls are complementary, not redundant:

```
Step 1 — Captive Portal
  Answers: "Who is this user? Are they allowed on the network?"
  Controls: Network access (Layer 3/4 + HTTP redirect)

Step 2 — Squid Proxy
  Answers: "What are they doing? Where are they going?"
  Controls: Traffic visibility and logging (Layer 7)

Step 3 — SquidGuard
  Answers: "Should this specific request be allowed?"
  Controls: Content filtering by category/domain (Layer 7)
```

A complete access control stack uses all three:
- **Captive portal** ensures only authenticated users are on the network
- **Squid** ensures all their traffic is visible and logged
- **SquidGuard** ensures they can only reach permitted content

Removing any one layer creates a gap — authenticated users can bypass content filters if Squid isn't enforced, and Squid logs are useless if anyone can access the network without authenticating.

---

## When to Use Each

| Scenario | Use |
|----------|-----|
| Guest Wi-Fi, need to know who's on the network | Captive Portal |
| Corporate network, need full web traffic audit trail | Squid |
| School network, need to block social media and adult content | Squid + SquidGuard |
| Small office, want basic logging without content filtering | Squid only |
| Need to enforce authentication AND content policy | All three |
| Need deep HTTPS inspection | Squid with SSL bump (more complex) |
