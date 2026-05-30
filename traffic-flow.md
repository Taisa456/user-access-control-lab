# Configuration & Traffic Flow

How traffic is controlled at each stage — from unauthenticated client to filtered outbound access.

---

## Captive Portal Flow

```
Client opens browser → navigates to any URL
        │
        ▼
Firewall intercepts HTTP/HTTPS request
        │
        ▼
Is client authenticated?
        │
   NO ──┴── YES
   │              │
   ▼              ▼
Redirect to    Allow traffic
portal page    to internet
   │
   ▼
Client submits credentials
(username + password → Local Database)
        │
        ▼
Firewall validates credentials
        │
   FAIL ─┴─ PASS
   │               │
   ▼               ▼
Show error     Create session
               Allow all traffic
               from client IP
```

**What the firewall does under the hood:**

```
Before auth:
  [Client] ──HTTP GET google.com──► [pfSense]
                                         │
                                         └─► Block original request
                                         └─► TCP 302 redirect → portal IP:port
                                         └─► Allow ONLY: client ↔ portal page

After auth:
  [Client] ──────────────────────► [pfSense] ──► [Internet]
  (all traffic passes normally)
```

---

## Squid Proxy Flow

### Without SquidGuard (logging only)

```
[Client]
    │
    │  Explicit proxy configured (IP:3128)
    ▼
[Squid on pfSense]
    │  Log entry written: client IP, destination, timestamp
    │
    ▼
[Internet]  ←── Request forwarded as-is
```

### With SquidGuard (content filtering)

```
[Client] ──► request to social network
    │
    ▼
[Squid] receives request
    │
    ▼
[SquidGuard] checks URL against blacklist categories
    │
    ├─ Category: social_networks → DENY
    │       └─► Client receives ERR_TUNNEL_CONNECTION_FAILED
    │
    └─ Category: not matched → ALLOW
            └─► Request forwarded to internet normally
```

---

## Combined Lab Architecture

```
                    ┌─────────────────────────────────┐
                    │         pfSense / OPNsense        │
                    │                                   │
  [Windows Client]  │  ┌─────────────┐  ┌───────────┐  │
        │           │  │   Captive   │  │   Squid   │  │
        │──LAN──────┼─►│   Portal    │  │   Proxy   │  │
                    │  │  (auth gate)│  │ + Guard   │  │
                    │  └─────────────┘  └───────────┘  │
                    │                        │          │
                    └────────────────────────┼──────────┘
                                             │
                                             ▼
                                        [Internet]
                               (filtered by SquidGuard ACLs)
```

---

## Privilege Model

```
User created in System > User Manager
        │
        ├─ Privilege: "Captive Portal login"  ← minimum required
        │
        └─ Group: NOT admins                  ← principle of least privilege
```

Assigning only the **Captive Portal login** privilege means the user can authenticate through the portal but cannot log into the pfSense admin interface. This follows the principle of least privilege — users get only what they need to perform their function.

---

## Bypass Vectors & Mitigations

| Bypass | How | Mitigation |
|--------|-----|-----------|
| Captive portal skip | Set static DNS to 8.8.8.8, use VPN | Block outbound DNS (port 53) except to firewall; block VPN ports |
| Squid bypass | Use HTTPS directly (port 443 not proxied) | Configure transparent proxy mode to intercept 443 |
| SquidGuard bypass | Use IP address instead of domain | Add IP-based ACL rules; DNS filtering layer |
| MAC spoofing | Clone authenticated client's MAC | 802.1X port authentication; DHCP static leases |
