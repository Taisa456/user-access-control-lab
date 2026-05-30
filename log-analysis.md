# Log Analysis & Filters

Reference for analyzing traffic in this lab — Wireshark filters for captive portal behavior, and Squid log interpretation.

---

## Wireshark — Captive Portal Analysis

| Filter | What it shows |
|--------|--------------|
| `http` | All HTTP traffic — useful to observe the portal redirect |
| `http.response.code == 302` | HTTP redirects — this is how the captive portal sends clients to the login page |
| `http.response.code == 200` | Successful responses — confirms portal page loaded |
| `http.request.method == "POST"` | Login form submission to the portal |
| `ip.addr == <firewall-lan-ip>` | All traffic to/from the firewall — shows portal interaction |
| `tcp.port == 8080` | Common captive portal redirect port on pfSense |
| `dns` | DNS queries — before auth, these may be blocked or redirected |
| `icmp` | Ping traffic — useful to confirm client/firewall reachability |

**Observing the redirect in Wireshark:**
```
1. Start capture on client NIC
2. Navigate to any HTTP site
3. Filter: http.response.code == 302
4. Follow HTTP stream → observe Location: header pointing to portal IP
```

**Confirming post-auth traffic:**
```
Filter: ip.src == <client-ip> && http
Before login: only portal-related traffic visible
After login:  full outbound HTTP visible
```

---

## Wireshark — Squid Proxy Analysis

| Filter | What it shows |
|--------|--------------|
| `tcp.port == 3128` | All traffic to/from Squid (default port) |
| `http && tcp.port == 3128` | HTTP requests going through the proxy |
| `ip.dst == <firewall-lan-ip> && tcp.port == 3128` | Client → Proxy requests |
| `http.request.method == "CONNECT"` | HTTPS CONNECT tunnels through proxy |
| `http.response.code == 403` | Squid/SquidGuard access denied response |

**Observing a blocked request:**
```
Filter: http.response.code == 403
Follow HTTP stream → response body contains SquidGuard block page
```

---

## Squid Real-Time Logs (pfSense GUI)

**Location:** Services > Squid Proxy Server > Real Time

**Log format:**
```
timestamp  response_time  client_ip  status/http_code  bytes  method  URL  peer  content_type
```

**Example entry:**
```
1234567890.123   45  192.168.1.15  TCP_MISS/200  12345  GET  http://example.com/  DIRECT/93.184.216.34  text/html
```

| Field | Meaning |
|-------|---------|
| `TCP_MISS` | Cache miss — request forwarded to origin |
| `TCP_HIT` | Cache hit — served from Squid cache |
| `TCP_DENIED` | Blocked by ACL or SquidGuard |
| `DIRECT` | Request went directly to origin server |
| `200` | Successful |
| `403` | Forbidden (blocked by SquidGuard) |

**Useful string filters in Real Time tab:**

| String | Finds |
|--------|-------|
| `192.168.1.15` | All requests from a specific client |
| `facebook` | Any request containing "facebook" in the URL |
| `DENIED` | All blocked requests |
| `TCP_MISS` | All cache misses (live requests) |

---

## SquidGuard Block Verification

After applying social network blacklist:

```
Expected behavior:
  Client → facebook.com  →  ERR_TUNNEL_CONNECTION_FAILED  (blocked)
  Client → example.com   →  200 OK                        (allowed)

In Squid logs:
  TCP_DENIED/403  GET  http://www.facebook.com/
```

If the block is not working:
- Confirm SquidGuard is **enabled** (Services > SquidGuard > General Settings > Enable)
- Click **Apply** in SquidGuard after any rule change — changes don't apply until this is clicked
- Confirm the client is routing through the proxy (check proxy settings)
- Check that the blacklist downloaded successfully (100% progress bar)

---

## Quick Reference

```
Portal redirect observed?    →  http.response.code == 302
Login POST captured?         →  http.request.method == "POST"
Squid receiving traffic?     →  tcp.port == 3128
Request blocked by Guard?    →  http.response.code == 403
Client-specific traffic?     →  ip.addr == <client-ip>
SquidGuard not blocking?     →  Check: enabled + Apply clicked + proxy configured on client
```
