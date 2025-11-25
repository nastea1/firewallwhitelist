# Firewalld Egress Lockdown (Supply Chain Mitigation)

![Security Status](https://img.shields.io/badge/Security-High-green) ![Firewall](https://img.shields.io/badge/firewalld-Direct_Rules-orange) ![Platform](https://img.shields.io/badge/Platform-RHEL%20%2F%20CentOS-red)

## 🛡️ Overview

This project implements a strict **"Walled Garden"** network security policy for Red Hat Enterprise Linux servers.

By default, servers are often configured to allow unrestricted outbound (egress) traffic. This leaves them vulnerable to **Supply Chain Attacks** (e.g., malicious npm packages, "Shai Hulud" style worms) which rely on "phoning home" to Command & Control (C2) servers or exfiltrating data to arbitrary IP addresses.

**This configuration blocks ALL outbound traffic by default**, allowing only:
1.  Established connections (replies).
2.  Specific internal administrative subnets.
3.  Specific internal DNS resolvers.
4.  A strict whitelist of external services (OS Updates, DB Repos, IMAP, API) maintained via IP Sets.

## 🏗️ Architecture

**Traffic Flow Logic:**
1.  **Priority 0:** Allow Loopback & Established Connections.
2.  **Priority 2:** Allow Internal Subnets & Internal DNS.
3.  **Priority 3:** Allow External Whitelists (HTTPS/IMAP) -> *Managed by Script*.
4.  **Priority 999:** **DROP EVERYTHING ELSE.**

## 📋 Prerequisites

* RHEL / Rocky / AlmaLinux 8 or 9
* `firewalld` (active and running)
* `ipset`
* `bind-utils` (for `dig` command)

## 🚀 Installation

### 1. Install Dependencies
```bash
sudo dnf install ipset bind-utils -y
```
### 2. Configure The Automation Script
Create the whitelist updater. This script resolves the dynamic IPs of allowed services (CDNs) and populates the kernel IP sets.
```bash
#!/bin/bash

# --- CONFIGURATION ---
# Trusted HTTPS Domains (OS Updates, Database Repos, APIs)
# Includes: Red Hat, TextBee, MongoDB, PostgreSQL
DOMAINS_HTTPS="cdn.redhat.com subscription.rhsm.redhat.com api.access.redhat.com api.textbee.dev repo.mongodb.org pgp.mongodb.com fastdl.mongodb.org download.postgresql.org"

# Trusted IMAP Domains
DOMAINS_IMAP="imap.mail.yahoo.com"

# Flush & Recreate IP Sets (Clears stale IPs)
firewall-cmd --permanent --delete-ipset=whitelist_https >/dev/null 2>&1
firewall-cmd --permanent --delete-ipset=whitelist_imap >/dev/null 2>&1
firewall-cmd --permanent --new-ipset=whitelist_https --type=hash:ip
firewall-cmd --permanent --new-ipset=whitelist_imap --type=hash:ip

echo "Resolving HTTPS domains..."
for domain in $DOMAINS_HTTPS; do
    for ip in $(dig +short $domain | grep -E '^[0-9.]+'); do
        firewall-cmd --permanent --ipset=whitelist_https --add-entry=$ip
    done
done

echo "Resolving IMAP domains..."
for domain in $DOMAINS_IMAP; do
    for ip in $(dig +short $domain | grep -E '^[0-9.]+'); do
        firewall-cmd --permanent --ipset=whitelist_imap --add-entry=$ip
    done
done

firewall-cmd --reload
echo "Firewall whitelist updated successfully."
```
Make it executable:
```bash
chmod +x /usr/local/bin/whitelistfirewall.sh
```
### 3. Apply Permanent Firewall Rules
Run these commands once to set the base policy.

```bash
# Allow Loopback & Established Connections
firewall-cmd --permanent --direct --add-rule ipv4 filter OUTPUT 0 -o lo -j ACCEPT
firewall-cmd --permanent --direct --add-rule ipv4 filter OUTPUT 0 -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow Internal Subnets
firewall-cmd --permanent --direct --add-rule ipv4 filter OUTPUT 2 -d 1.1.1.1/23 -j ACCEPT
firewall-cmd --permanent --direct --add-rule ipv4 filter OUTPUT 2 -d 2.2.2.2/23 -j ACCEPT

# Allow Internal DNS Only (Update IPs as needed)
for dns_ip in 1.3.2.8 1.3.3.8 1.3.3.7 1.3.4.3; do
  firewall-cmd --permanent --direct --add-rule ipv4 filter OUTPUT 2 -d $dns_ip -p udp --dport 53 -j ACCEPT
done

# Link the Whitelists (HTTPS & IMAP)
firewall-cmd --permanent --direct --add-rule ipv4 filter OUTPUT 3 -p tcp --dport 443 -m set --match-set whitelist_https dst -j ACCEPT
firewall-cmd --permanent --direct --add-rule ipv4 filter OUTPUT 3 -p tcp --dport 993 -m set --match-set whitelist_imap dst -j ACCEPT

# LOCKDOWN: Drop All Other Outbound Traffic
firewall-cmd --permanent --direct --add-rule ipv4 filter OUTPUT 998 -j LOG --log-prefix "FW_DROP_OUT: "
firewall-cmd --permanent --direct --add-rule ipv4 filter OUTPUT 999 -j DROP

# Initial Population
/usr/local/bin/whitelistfirewall.sh
```

### 4. Setup Cron Job 
Since Red Hat and MongoDB use CDNs (Akamai/Fastly), their IPs change frequently. You must update the whitelist daily. NOTE: Recommended to run before an update as well due to the changing IPs.
```bash
# Add to root crontab (crontab -e)
0 4 * * * /usr/local/bin/whitelistfirewall.sh >/dev/null 2>&1
```

## 🛡️Verification
### Check active Direct Rules:
```bash
firewall-cmd --direct --get-all-rules
```
Ensure priority 999 -j DROP is present.
### Test Blocked Traffic (Should Fail/Timeout):
```bash
curl -v https://www.google.com
# Output: Connection timed out (or hanging)
```
## Test Allowed Traffic (Should Succeed):
```bash
curl -I https://cdn.redhat.com
# Output: HTTP/1.1 200 OK
```
