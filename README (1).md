# User Access Control Lab

Hands-on lab configuring two network access control mechanisms — a Captive Portal and a Squid Proxy with content filtering — on a pfSense/OPNsense firewall in an isolated virtual lab environment.

> All configuration was performed on a self-hosted firewall VM within a private lab network. No production systems were involved.

---

## Topics Covered

- Captive portal deployment and authentication on pfSense/OPNsense
- Local user creation and privilege assignment
- Firewall rule configuration for portal enforcement
- Squid Proxy installation, configuration, and ACL setup
- Content filtering with SquidGuard and category-based blacklists
- Real-time traffic monitoring via Squid logs

---

## Tools & Technologies

| Tool | Purpose |
|------|---------|
| pfSense / OPNsense | Firewall and network gateway |
| Captive Portal (built-in) | Enforce authentication before granting network access |
| Squid Proxy | Transparent HTTP proxy with access control |
| SquidGuard | Content filtering plugin for Squid |
| Windows VM | Client machine for testing |

---

## Environment

- **Firewall:** pfSense/OPNsense (VM)
- **Client machine:** Windows VM
- **Network:** Isolated internal LAN (192.168.x.0/24)
- **Virtualization:** VMware

---

## Lab Walkthrough

### Activity 1 – Captive Portal

**Goal:** Force all LAN clients to authenticate through a login page before being granted internet access.

#### Step 1 — Enable Captive Portal

1. Navigate to **Services > Captive Portal**.
2. Click **Add**, select the **LAN interface**, and check **Enable**.
3. Set authentication method to **Local Database** — users must log in with credentials stored on the firewall itself.
4. Save and apply.

#### Step 2 — Create a Test User

1. Go to **System > User Manager > Add**.
2. Fill in username and password.
3. Scroll to **Effective Privileges**, click **Add**, and assign the **Captive Portal login** privilege.
4. Ensure the new user is **not** in the `admins` group — portal users should have minimum required privileges only.

#### Step 3 — Configure Firewall Rules

1. Go to **Firewall > Rules > LAN**.
2. Add a **Pass** rule:
   - Action: `Pass`
   - Source: `LAN subnet`
   - Destination: `Any`
3. Save and apply.

#### Step 4 — Configure Client Static IP

On the Windows client:
1. Open **Network and Sharing Center > Change adapter settings**.
2. Right-click the adapter → **Properties > IPv4**.
3. Set a static IP within the LAN range.
4. Set Default Gateway to the pfSense LAN IP.
5. Set DNS to the firewall IP or a public resolver.

#### Step 5 — Test

1. Open a browser on the Windows client and navigate to any site (e.g., google.com).
2. Verify you are **redirected to the Captive Portal login page** instead.
3. The firewall blocks all traffic except communication with the portal page — this is by design.
4. Enter the credentials created in Step 2.
5. Confirm the original site now loads successfully after login.

**How it works:** Before authentication, the firewall intercepts all outbound HTTP/HTTPS requests and redirects them to the portal. It temporarily allows only TCP traffic to the login page IP. Everything else is blocked at the interface level until the client authenticates.

**Key takeaway:** Captive portals are a Layer 7 access control mechanism. They don't encrypt traffic — they gate it. Common in hotels, airports, and enterprise guest networks. The security model relies on the firewall's ability to intercept and redirect HTTP, and on users not bypassing it via VPN or static DNS.

---

### Activity 2 – Squid Proxy with Content Filtering

**Goal:** Deploy a forward proxy that logs and controls outbound web traffic, then add category-based content filtering to block specific site categories.

#### Step 1 — Install Squid

1. Navigate to **System > Package Manager > Available Packages**.
2. Search for `squid` and click **Install**.
3. Wait for the success confirmation.

#### Step 2 — Enable and Configure Squid

1. Go to **Services > Squid Proxy Server**.
2. Check **Enable Squid Proxy**.
3. Set **Proxy Interface** to `LAN`.
4. Save.

#### Step 3 — Configure ACLs

1. Go to the **ACLs** tab within Squid settings.
2. In the **Allowed Subnets** field, enter your LAN subnet (e.g., `192.168.1.0/24`).
3. Save.

#### Step 4 — Firewall Rule

1. Go to **Firewall > Rules > LAN**.
2. Add a Pass rule:
   - Protocol: `TCP`
   - Source: `LAN subnets`
   - Destination: `Any`
3. Save and apply.

#### Step 5 — Configure Client Proxy Settings

On the Windows client:
1. Open **Settings > Network > Proxy**.
2. Toggle **Use a proxy server** to **On**.
3. Enter the firewall LAN IP as the address and `3128` as the port (Squid default).
4. Save.

#### Step 6 — Verify Proxy is Working

1. Browse to any site — confirm it loads normally.
2. In pfSense, go to **Services > Squid Proxy Server > Real Time**.
3. Filter by your client IP or a domain string to see live traffic entries.
4. Confirm your requests appear in the log.

#### Step 7 — Install SquidGuard for Content Filtering

1. Go to **System > Package Manager > Available Packages**.
2. Search for `squidGuard` and install it.

#### Step 8 — Load Blacklists

1. Go to **Services > SquidGuard Proxy Filter > Blacklist**.
2. Enter the blacklist archive URL (University of Toulouse blacklist — commonly used with pfSense).
3. Click **Download** and wait for completion.

#### Step 9 — Apply Category Rules

1. Go to the **Common ACL** tab in SquidGuard.
2. Expand **Target Rules List**.
3. Find `blk_blacklists_social_networks` and set it to **deny**.
4. Set **Default Access** to **allow** — block only the specific category, allow everything else.
5. Save and apply.

#### Step 10 — Test Content Filtering

1. On the Windows client, navigate to a blocked site (e.g., a social network).
2. Confirm the page fails with `ERR_TUNNEL_CONNECTION_FAILED`.
3. Navigate to a non-blocked site and confirm it loads normally — the filter is category-specific, not global.

**How it works:** Squid acts as a forward proxy — all client HTTP requests pass through it before reaching the internet. SquidGuard adds a filtering layer that checks each request against category blacklists. Matched categories are denied before the connection is established.

**Key takeaway:** Proxy-based content filtering operates at Layer 7. It can inspect URLs, domains, and content categories. The limitation is that it requires clients to route through the proxy — either via explicit proxy settings (as configured here) or via transparent proxy mode where the firewall intercepts traffic automatically.

---

## Summary

| Activity | Mechanism | Enforcement Point |
|----------|-----------|------------------|
| Captive Portal | Authentication gate before network access | Firewall (Layer 7 redirect) |
| Squid Proxy | Forward proxy with traffic logging | Proxy server (Layer 7) |
| SquidGuard | Category-based URL/domain blacklisting | Proxy filter plugin |

---

## Disclaimer

All configuration documented here was performed in a controlled, isolated lab environment for educational purposes only. Do not apply these techniques to networks you do not own or have explicit written permission to administer.
