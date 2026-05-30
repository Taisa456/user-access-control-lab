# Bypass Testing

Documenting attempts to bypass the access controls configured in this lab. Testing both the captive portal and Squid proxy for gaps — verifying that controls work as intended, and identifying where they fall short.

> All tests performed on lab VMs within an isolated network.

---

## Captive Portal Bypass Tests

### Test 1 — Direct IP Access (no DNS)

**Hypothesis:** The captive portal redirect relies on intercepting HTTP requests. If the client uses a direct IP instead of a domain, the redirect may still trigger because the firewall intercepts at the interface level — not at the DNS level.

**Test:**
```
Before authenticating:
  Browser → http://93.184.216.34  (example.com's IP)
```

**Expected result:** Still redirected to the portal — the firewall intercepts all outbound HTTP regardless of whether a domain or IP was used.

**Why it matters:** Some users try to bypass captive portals by hardcoding IPs to avoid DNS-based detection. On pfSense, the portal works at the firewall rule level, so this bypass doesn't work against a correctly configured setup.

---

### Test 2 — Change DNS to External Resolver

**Hypothesis:** If the client changes DNS to 8.8.8.8 instead of the firewall, it might resolve domains before the portal redirect kicks in.

**Test:**
```
Set client DNS: 8.8.8.8
Before authenticating:
  Browser → http://google.com
```

**Expected result:** DNS resolves fine (firewall may allow DNS out by default), but the HTTP connection is still intercepted and redirected. Changing DNS doesn't bypass the portal — it only affects name resolution, not traffic filtering.

**Gap:** If the firewall does NOT block outbound DNS (port 53) to external resolvers, clients can use their own DNS. This doesn't break the portal but is a hygiene issue — all DNS should go through the firewall for logging and control.

**Mitigation:** Add a firewall rule blocking outbound port 53 to any destination except the firewall's own IP.

---

### Test 3 — HTTPS Site Before Authentication

**Hypothesis:** The captive portal redirect works on HTTP (port 80). HTTPS (port 443) uses TLS — the firewall can't inject a redirect into an encrypted stream without breaking the connection.

**Test:**
```
Before authenticating:
  Browser → https://google.com
```

**Expected result:** The browser shows a connection error or a certificate warning — not a clean portal redirect. The firewall drops the connection but cannot perform a transparent redirect into HTTPS.

**Gap:** This is a known limitation of captive portals. Modern browsers default to HTTPS. The portal only cleanly intercepts HTTP. pfSense handles this by also blocking HTTPS and letting the browser time out, then the user tries an HTTP URL and gets redirected — but this depends on the user knowing to try HTTP first.

**Mitigation:** Configure a DNS intercept that resolves all domains to the portal IP before authentication, forcing the browser to hit the portal regardless of protocol. Or use a landing page that the browser's captive portal detection (NCSI/CNA) hits automatically.

---

### Test 4 — MAC Address Spoofing

**Hypothesis:** Once a device authenticates, pfSense ties the session to the client's IP and MAC. Spoofing the MAC of an authenticated device on a different machine could hijack the session.

**Test:**
```
Device A authenticates successfully.
Device B changes MAC to match Device A's MAC.
Device B attempts to browse without authenticating.
```

**Expected result:** Depending on pfSense version and configuration, Device B may be granted access because its MAC matches an active session.

**Gap:** This is a real weakness of MAC-based session tracking. MACs are trivially spoofable on most OSes.

**Mitigation:** 802.1X port-based authentication (requires a RADIUS server and managed switch). DHCP static leases + IP/MAC binding. Session timeouts to limit the window of exposure.

---

### Test 5 — VPN Tunnel Before Authentication

**Hypothesis:** If the client can establish a VPN connection before the captive portal fires, all traffic will be tunneled and the portal will never see it.

**Test:**
```
Before authenticating:
  Client opens WireGuard/OpenVPN client → connects to external VPN server
```

**Expected result:** Likely fails — the firewall blocks all outbound traffic except the portal redirect before authentication. UDP 1194 (OpenVPN) and UDP 51820 (WireGuard) are not in the allowed pre-auth ruleset.

**Gap:** If the admin has not explicitly blocked VPN ports pre-auth, a client could tunnel out. Also, VPNs over port 443 (TCP) are harder to block without breaking HTTPS entirely.

**Mitigation:** Pre-auth firewall rules should be as restrictive as possible — allow only TCP to the portal IP on the portal port. Block everything else explicitly.

---

## Squid Proxy Bypass Tests

### Test 6 — Remove Proxy Settings on Client

**Hypothesis:** In explicit proxy mode (as configured in this lab), the client must manually point to Squid. Simply removing the proxy settings bypasses it entirely.

**Test:**
```
Client proxy settings: disabled
Browser → facebook.com
```

**Expected result:** Page loads — Squid is completely bypassed. Traffic goes directly to the internet without passing through the proxy.

**Gap:** This is the fundamental weakness of explicit proxy mode. It relies entirely on client cooperation or enforcement via GPO/MDM.

**Mitigation:** Configure Squid in **transparent proxy mode** — the firewall intercepts port 80/443 and redirects to Squid automatically, regardless of client proxy settings. This removes the bypass entirely for HTTP. For HTTPS transparent interception, SSL bump/splice must be configured (with associated cert trust implications).

---

### Test 7 — Access by IP Address (SquidGuard bypass)

**Hypothesis:** SquidGuard blacklists work on domain names. Accessing a blocked site by its IP address may bypass the category filter.

**Test:**
```
Blocked domain: www.facebook.com
Client tries: http://157.240.22.35  (Facebook's IP)
```

**Expected result:** May load — SquidGuard matches on URLs/domains, not IP addresses. If the IP is not in a separate IP blacklist, the request passes through.

**Gap:** Domain-based blacklists are bypassable via direct IP access.

**Mitigation:** Add IP-based ACL rules in Squid. Use DNS filtering (e.g., pfBlockerNG) as a complementary layer — if Facebook's domain doesn't resolve, the client can't easily find the IP either.

---

### Test 8 — HTTPS Sites Not in Blacklist Scope

**Hypothesis:** SquidGuard can only inspect the domain portion of HTTPS URLs (via SNI in the CONNECT request). It cannot inspect the full URL path of HTTPS traffic without SSL bump.

**Test:**
```
Blocked: www.facebook.com (HTTP)
Client tries: https://www.facebook.com
```

**Expected result:** The CONNECT request to facebook.com is intercepted by Squid, and SquidGuard can block based on the hostname in the CONNECT tunnel — so it should still be blocked. However, without SSL bump, Squid cannot inspect the full HTTPS content.

**Note:** Whether HTTPS is blocked depends on how SquidGuard is configured to handle CONNECT requests. If only HTTP URLs are matched, HTTPS slips through.

**Mitigation:** Configure SquidGuard to also match on CONNECT hostnames. For full HTTPS inspection, enable SSL bump in Squid (requires deploying a trusted CA cert to all clients).

---

### Test 9 — Use a Different Port

**Hypothesis:** If a site runs on a non-standard port (e.g., port 8443 instead of 443), Squid may not intercept it.

**Test:**
```
Client → https://somesite.com:8443
```

**Expected result:** Squid may or may not intercept depending on its `ssl_ports` ACL configuration. By default, Squid only handles standard ports.

**Mitigation:** Expand Squid's SSL port ACL to cover common alternative ports. Use firewall rules to block outbound traffic on uncommon ports entirely.

---

## Summary

| Test | Control Tested | Bypass Works? | Mitigation |
|------|---------------|---------------|-----------|
| Direct IP access | Captive portal | No | Works correctly at firewall level |
| Change DNS | Captive portal | No (partial gap) | Block outbound port 53 |
| HTTPS before auth | Captive portal | Partial | DNS intercept or CNA detection |
| MAC spoofing | Captive portal | Potentially yes | 802.1X, DHCP binding, session timeouts |
| VPN tunnel | Captive portal | No (if ports blocked) | Restrict pre-auth ruleset |
| Disable proxy settings | Squid (explicit) | Yes | Switch to transparent proxy mode |
| IP instead of domain | SquidGuard | Yes | IP ACLs + DNS filtering layer |
| HTTPS without SSL bump | SquidGuard | Partial | Match on CONNECT hostname; SSL bump |
| Non-standard port | Squid | Potentially | Restrict outbound ports at firewall |
