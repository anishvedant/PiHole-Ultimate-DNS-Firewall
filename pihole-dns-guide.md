# 🛡️ Zero-Trust Home DNS: Pi-hole + Unbound + DoH fallback (Quad9) + Hagezi Multi Pro

Turn a Raspberry Pi into a privacy-hardened, network-wide ad, tracker, and malware blocker that you control.  
This guide is the full story: how ads actually work, why DNS filtering helps, what I built, how I built it, how I tested it, results, and key takeaways.

> Hardware: Raspberry Pi 5, Ethernet to Xfinity gateway, storage on SSD  
> Network goal: filter only my devices, not the entire household, while keeping the option to run a dedicated AP for my devices

---

## Table of contents

- [How ads and trackers actually work](#how-ads-and-trackers-actually-work)
- [Why DNS filtering by itself cannot reach 100 percent](#why-dns-filtering-by-itself-cannot-reach-100-percent)
- [What I built](#what-i-built)
- [Architecture diagrams](#architecture-diagrams)
- [Roles of each tool](#roles-of-each-tool)
- [Step by step setup](#step-by-step-setup)
  - [0. Give the Pi a stable IP](#0-give-the-pi-a-stable-ip)
  - [1. Install Pi-hole](#1-install-pi-hole)
  - [2. Install Unbound as a local recursive resolver](#2-install-unbound-as-a-local-recursive-resolver)
  - [3. Point Pi-hole to Unbound and force strict order](#3-point-pi-hole-to-unbound-and-force-strict-order)
  - [4. Add DNS over HTTPS fallback using cloudflared with Quad9](#4-add-dns-over-https-fallback-using-cloudflared-with-quad9)
  - [5. Load high-quality blocklists (Hagezi Multi Pro, optional OISD)](#5-load-high-quality-blocklists-hagezi-multi-pro-optional-oisd)
  - [6. Optional hardening: regex blocks and DoH bypass reduction](#6-optional-hardening-regex-blocks-and-doh-bypass-reduction)
  - [7. Client configuration and browser settings](#7-client-configuration-and-browser-settings)
  - [8. Optional AP-in-the-middle for just my devices](#8-optional-ap-in-the-middle-for-just-my-devices)
- [Validation and tests](#validation-and-tests)
- [Troubleshooting notes I hit and fixed](#troubleshooting-notes-i-hit-and-fixed)
- [Results, outcomes, and takeaways](#results-outcomes-and-takeaways)
- [Screenshots](#screenshots)
- [Appendix A. Full configs](#appendix-a-full-configs)
- [Appendix B. Command cheat sheet](#appendix-b-command-cheat-sheet)
- [Credits](#credits)

---

## How ads and trackers actually work

When you load a page or open an app, a lot happens under the hood:

1. Your device resolves domain names via DNS to reach servers.
2. The page pulls content from the main site and many third-party domains:
   - ad networks and exchanges
   - analytics and telemetry (Google Analytics, Hotjar, Mouseflow, Sentry, Bugsnag, etc)
   - content delivery networks for images, videos, scripts
3. Ads can be:
   - third-party: assets and scripts loaded from ad or tracker domains
   - first-party: served from the same domain as the site itself
   - inline and dynamic: injected by JavaScript after the DOM loads
4. Modern browsers and apps may use DNS over HTTPS or DNS over TLS to send DNS directly to a public resolver. That can bypass your local DNS filter.

Key points:
- DNS filtering can block whole domains before any connection is made.
- DNS filtering cannot remove a banner that is hosted on the same domain or hide elements in the page. That is why some ad tests grade you lower on "static image" and "GIF" checks even when DNS blocking works perfectly.

---

## Why DNS filtering by itself cannot reach 100 percent

- DNS filters block domains. They cannot modify the HTML, CSS, or JavaScript already delivered by the site.
- Tester pages often check same-site images and inline scripts. A DNS sinkhole cannot block those without breaking the whole site.
- To hit 100 on aggressive web tests, you layer a browser content blocker on the device you are testing. Example: uBlock Origin with EasyList, EasyPrivacy, and Annoyances filters. DNS + browser filter together give the best coverage.

---

## What I built

A layered, privacy-first DNS stack that I control:

- Pi-hole does network-wide filtering for my devices.
- Unbound runs on the Pi as a recursive resolver so I do not leak my DNS history to public DNS providers.
- cloudflared provides encrypted DoH fallback to Quad9 only if Unbound fails, never in a race.
- Hagezi Multi Pro list increases coverage for ads, trackers, telemetry, phishing, and more.
- Strict upstream order ensures privacy wins over speed.
- Client devices are pointed at the Pi so they benefit from filtering. Browser secure DNS is disabled so nothing bypasses my resolver.

---

## Architecture diagrams

```
[Device] → Pi-hole (filter)
```

Optional AP-in-the-middle for just my devices:

```
Internet ⇄ Xfinity Gateway ⇄ Pi eth0
                    ⇣
         Pi wlan0 (hostapd SSID for my devices)
         NAT + force DNS to Pi-hole
```

---

## Roles of each tool

- **Pi-hole**  
  DNS sinkhole and policy engine. Applies blocklists and custom rules. Gives logs and visibility. Does not modify pages. Purpose: block domains at network layer for all apps on a device.

- **Unbound**  
  Local recursive resolver. Starts at the root, walks the DNS tree, validates with DNSSEC, and returns authentic answers. Purpose: privacy. No third party logs my queries during normal operation.

- **cloudflared (proxy-dns)**  
  Lightweight DNS over HTTPS forwarder. Here it is only a fallback path, not primary. I set upstream to Quad9 DoH so even the fallback is encrypted and has threat protection. Purpose: encrypted resilience if Unbound is down.

- **Hagezi Multi Pro**  
  High quality, curated blocklist that goes beyond basic ads into trackers, telemetry, and security. Purpose: better coverage without hand-maintaining thousands of domains.

- **uBlock Origin (optional on the browser I test with)**  
  Page-level blocking and cosmetic filtering. Purpose: hide and stop same-site assets and inline scripts that DNS cannot touch, so I can score 100 on strict ad tests if I want.

---

## Step by step setup

### 0. Give the Pi a stable IP

Best practice is a DHCP reservation on the Xfinity gateway so the Pi always gets the same address. Do not set static on device and reservation at the same time. Choose one.

### 1. Install Pi-hole

```bash
curl -sSL https://install.pi-hole.net | bash
```

After install, open the admin at http://<pi-ip>/admin.
- Settings → DNS → Listen on all interfaces
- Later we will set Custom upstreams after Unbound and cloudflared are ready

### 2. Install Unbound as a local recursive resolver

```bash
sudo apt update
sudo apt install unbound -y
wget -O root.hints https://www.internic.net/domain/named.root
sudo mv root.hints /var/lib/unbound/root.hints
```

Create Unbound config:

```bash
sudo nano /etc/unbound/unbound.conf.d/pi-hole.conf
```

Paste:

```bash
server:
    interface: 127.0.0.1
    port: 5335
    do-ip4: yes
    do-udp: yes
    do-tcp: yes
    access-control: 127.0.0.0/8 allow

    root-hints: "/var/lib/unbound/root.hints"

    hide-identity: yes
    hide-version: yes
    harden-glue: yes
    harden-dnssec-stripped: yes
    use-caps-for-id: no
    prefetch: yes

    # light but safe defaults
    num-threads: 2
    msg-cache-size: 8m
    rrset-cache-size: 16m
    so-rcvbuf: 1m
```

Restart Unbound:

```bash
sudo systemctl restart unbound
sudo systemctl enable unbound
```

Quick test:

```bash
dig pi-hole.net @127.0.0.1 -p 5335
```

You should get status: NOERROR and an A record.

DNSSEC validation tests:

```bash
dig sigok.verteiltesysteme.net @127.0.0.1 -p 5335
dig sigfail.verteiltesysteme.net @127.0.0.1 -p 5335
```

- sigok should succeed and show the ad flag
- sigfail should fail. Depending on Unbound and timing you may see SERVFAIL or a timeout. Both mean Unbound rejected a bad signature and did not return an IP. That is correct.

### 3. Point Pi-hole to Unbound and force strict order

Pi-hole Admin → Settings → DNS
- Uncheck all predefined upstreams
- Set Custom 1 (IPv4) to 127.0.0.1#5335
- Save

Force strict order so Pi-hole does not race fallbacks:

```bash
sudo nano /etc/dnsmasq.d/99-strict-order.conf
```

Add:

```bash
strict-order
```

Reload Pi-hole DNS:

```bash
sudo pihole restartdns
```

### 4. Add DNS over HTTPS fallback using cloudflared with Quad9

Install cloudflared and run it in proxy-dns mode on port 5053. The tunnel mode is not needed.

Create a systemd unit:

```bash
sudo nano /etc/systemd/system/cloudflared-proxy-dns.service
```

Paste:

```bash
[Unit]
Description=cloudflared DNS over HTTPS proxy
After=network.target

[Service]
ExecStart=/usr/bin/cloudflared proxy-dns --port 5053 --upstream https://dns.quad9.net/dns-query
Restart=on-failure
User=nobody

[Install]
WantedBy=multi-user.target
```

Enable it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable cloudflared-proxy-dns
sudo systemctl start cloudflared-proxy-dns
sudo systemctl status cloudflared-proxy-dns
```

If your cloudflared build does not support --config for proxy-dns, keep the flags in ExecStart as above. That is intentional.

Add Quad9 DoH as fallback in Pi-hole:
- Settings → DNS
- Custom 1 (IPv4): 127.0.0.1#5335 Unbound
- Custom 2 (IPv4): 127.0.0.1#5053 cloudflared DoH to Quad9
- Save

Strict order ensures Unbound is always first, cloudflared only on failure.

### 5. Load high-quality blocklists (Hagezi Multi Pro, optional OISD)

Pi-hole Admin → Group Management → Lists

Add Hagezi Multi Pro:

```
https://raw.githubusercontent.com/hagezi/dns-blocklists/main/multi/pro.txt
```

Optional, also add OISD small or big:

```
https://small.oisd.nl/
```

or

```
https://big.oisd.nl/
```

Apply lists:
- Tools → Update Gravity

### 6. Optional hardening: regex blocks and DoH bypass reduction

Group Management → Domains

Add regex to quiet common telemetry used by test pages:

```
(^|\.)sentry\.io$
(^|\.)bugsnag\.com$
(^|\.)advmaker\..*$
```

Reduce easy DoH bypass by blocking common public DoH hostnames as exact domains:

```
dns.google
cloudflare-dns.com
mozilla.cloudflare-dns.com
one.one.one.one
doh.opendns.com
dns.quad9.net
dns.nextdns.io
dns.adguard.com
```

Flush tables:
- Tools → Flush network table

### 7. Client configuration and browser settings

On devices you want filtered, set DNS to your Pi-hole IP.

Windows 11 example:
- Settings → Network → Wi-Fi or Ethernet → Hardware properties → DNS → Manual
- IPv4 Preferred DNS: <Pi_IP>
- Do not enable DNS over HTTPS here. That would bypass your Pi-hole.
- Either disable IPv6 on that adapter, or set the Pi-hole IPv6 address as DNS if you run IPv6 through Pi-hole.

Browser secure DNS:
- Chrome or Edge: Settings → Privacy and security → Use secure DNS → Off
- Firefox: Settings → Network → DNS over HTTPS → Off

Optional for 100 percent on strict ad tests:
- Install uBlock Origin
- Enable EasyList, EasyPrivacy, uBlock filters, AdGuard Annoyances
- For the specific tester domain, you can add local dynamic rules to block 3rd-party scripts and frames, and images if the test uses same-site assets

### 8. Optional AP-in-the-middle for just my devices

[This section appears to be referenced but not included in the original text]

---

## Validation and tests

Unbound recursion and DNSSEC:

```bash
dig pi-hole.net @127.0.0.1 -p 5335
# expect NOERROR and an A record

dig sigok.verteiltesysteme.net @127.0.0.1 -p 5335
# expect NOERROR, "ad" flag present

dig sigfail.verteiltesysteme.net @127.0.0.1 -p 5335
# expect SERVFAIL or a timeout. No IP returned. That is correct.
```

cloudflared DoH fallback works:

```bash
dig example.com @127.0.0.1 -p 5053
```

Pi-hole is receiving and filtering your client's queries:
- Admin → Query Log
- Load a few sites and confirm queries appear from your client IP
- You should see Hagezi domains being blocked

Windows verifications:

```powershell
Get-DnsClientServerAddress -AddressFamily IPv4,IPv6
nslookup doubleclick.net <Pi_IP>
```

Ad test pages:
- adblock-tester and turtlecute. Expect partial credit with DNS alone. Install uBlock Origin to reach 100 on those pages if you care about the number. DNS still protects every app, not only the browser.

---

## Troubleshooting notes I hit and fixed

- `pihole` without a subcommand shows usage. Use `sudo pihole restartdns` to reload configs.
- DNSSEC fail domain returning timeout instead of SERVFAIL is fine. Unbound refused to return a forged answer. The key is no IP is returned.
- cloudflared confusion:
  - `cloudflared service install` and `tunnel` config are for Cloudflare Tunnels. Not needed here.
  - Use `proxy-dns` mode with flags in ExecStart and run it under a custom unit named `cloudflared-proxy-dns.service`.
- Windows secure DNS and IPv6 can bypass Pi-hole:
  - Turn secure DNS off in the browser and OS network settings if you are pointing at the Pi
  - Either disable IPv6 on the client adapter or set your Pi-hole IPv6 address as the DNS
- Score drops on tester pages usually mean:
  - Browser secure DNS turned itself back on
  - A VPN or virtual adapter is the active interface
  - You turned off the browser extension that adds page-level blocking

---

## Results, outcomes, and takeaways

- **Privacy**: DNS leak tests show traffic resolved locally by Unbound. No third party sees my queries during normal operation. The fallback, when used, is encrypted to Quad9.
- **Security**: DNSSEC protects against spoofing. Phishing and malware domains are blocked by lists and by Quad9 when fallback occurs.
- **Coverage**: DNS filtering works across browsers, apps, smart TV, game consoles, and IoT. This is bigger than a browser extension.
- **Performance**: Pages feel snappier without ad calls and trackers. Fewer megabytes wasted on junk.
- **Control**: I can see exactly what every device queries and I can enforce policy per device if I want.
- **Reality check**: DNS alone cannot remove same-site banners or inline scripts. For 100 percent on aggressive ad tests you add a browser blocker on the device you test with. That is by design, not a flaw.

---

## Screenshots

Add your screenshots into a `screenshots/` folder and link them here.

- Pi-hole dashboard  
  ![Pi-hole Dashboard](screenshots/dashboard.png)

- Lists page with Hagezi Multi Pro  
  ![Adlists](screenshots/adlists-hagezi.png)

- DNS settings showing Unbound primary and Quad9 DoH fallback  
  ![Pi-hole DNS Upstreams](screenshots/pihole-dns-upstreams.png)

- Query log while running tests  
  ![Query Log](screenshots/querylog-tests.png)

- Windows DNS settings pointing to Pi and secure DNS off  
  ![Windows DNS Settings](screenshots/windows-dns.png)

- Ad test results before and after browser blocker  
  ![Ad Test](screenshots/adtest-before-after.png)

---

## Appendix A. Full configs

### Unbound

`/etc/unbound/unbound.conf.d/pi-hole.conf`

```bash
server:
    interface: 127.0.0.1
    port: 5335
    do-ip4: yes
    do-udp: yes
    do-tcp: yes
    access-control: 127.0.0.0/8 allow

    root-hints: "/var/lib/unbound/root.hints"

    hide-identity: yes
    hide-version: yes
    harden-glue: yes
    harden-dnssec-stripped: yes
    use-caps-for-id: no
    prefetch: yes

    num-threads: 2
    msg-cache-size: 8m
    rrset-cache-size: 16m
    so-rcvbuf: 1m
```

### Pi-hole dnsmasq strict order

`/etc/dnsmasq.d/99-strict-order.conf`

```bash
strict-order
```

### cloudflared DoH to Quad9

`/etc/systemd/system/cloudflared-proxy-dns.service`

```bash
[Unit]
Description=cloudflared DNS over HTTPS proxy
After=network.target

[Service]
ExecStart=/usr/bin/cloudflared proxy-dns --port 5053 --upstream https://dns.quad9.net/dns-query
Restart=on-failure
User=nobody

[Install]
WantedBy=multi-user.target
```

### Optional nftables for AP clients

`/etc/nftables.conf`

```bash
flush ruleset
table inet pihole {
  chain prerouting {
    type filter hook prerouting priority -300;
    iif "wlan0" udp dport 53 redirect to 53
    iif "wlan0" tcp dport 53 redirect to 53
    iif "wlan0" tcp dport 853 drop
  }
  chain forward {
    type filter hook forward priority 0;
    ct state established,related accept
    iif "wlan0" oif "eth0" accept
  }
  chain postrouting {
    type nat hook postrouting priority 100;
    oif "eth0" masquerade
  }
}
```

---

## Appendix B. Command cheat sheet

```bash
# Update packages
sudo apt update && sudo apt upgrade -y

# Install Pi-hole
curl -sSL https://install.pi-hole.net | bash

# Install Unbound and root hints
sudo apt install unbound -y
wget -O root.hints https://www.internic.net/domain/named.root
sudo mv root.hints /var/lib/unbound/root.hints
sudo systemctl restart unbound
sudo systemctl enable unbound

# Test Unbound and DNSSEC
dig pi-hole.net @127.0.0.1 -p 5335
dig sigok.verteiltesysteme.net @127.0.0.1 -p 5335
dig sigfail.verteiltesysteme.net @127.0.0.1 -p 5335

# Pi-hole DNS reload after changes
sudo pihole restartdns

# cloudflared DoH service
sudo nano /etc/systemd/system/cloudflared-proxy-dns.service
sudo systemctl daemon-reload
sudo systemctl enable cloudflared-proxy-dns
sudo systemctl start cloudflared-proxy-dns
sudo systemctl status cloudflared-proxy-dns

# Update Gravity after adding lists
pihole -g

# Quick Windows client checks
ipconfig /flushdns
nslookup doubleclick.net <Pi_IP>
```

---

## Credits

- [Pi-hole](https://pi-hole.net/)
- [Unbound by NLnet Labs](https://nlnetlabs.nl/projects/unbound/)
- [cloudflared](https://github.com/cloudflare/cloudflared)
- [Quad9](https://www.quad9.net/)
- [Hagezi DNS blocklists](https://github.com/hagezi/dns-blocklists)
- [OISD](https://oisd.nl/)
- [uBlock Origin](https://github.com/gorhill/uBlock)