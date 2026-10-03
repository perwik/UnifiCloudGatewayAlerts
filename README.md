[README.md](https://github.com/user-attachments/files/33013151/README.md)
# UniFi alerts → syslog-ng → Zentyal → Outlook

Public setup guide — based on a deployment verified working on 4 October 2026.

**All installation-specific names, addresses, paths, MAC addresses and application identifiers below are examples or placeholders.** Replace them before running commands. Software versions and the diagnostic history are retained to make the guide reproducible; they describe the tested deployment, not guaranteed compatibility with future versions.

| Placeholder | Replace with |
| --- | --- |
| `YOUR_OUTLOOK_ACCOUNT@outlook.com` | Your personal Outlook.com mailbox |
| `REPLACE_WITH_APPLICATION_CLIENT_ID` | Your app registration's Application (client) ID |
| `REPLACE_WITH_TENANT_ID` / `REPLACE_WITH_OBJECT_ID` | Registration metadata; not needed in the working consumer token command |
| `lab.example`, `alerts@lab.example` | Your internal domain and local Zentyal mailbox |
| `192.168.50.x` | Your actual LAN addresses, consistently throughout the guide |
| `/share/NASShare/syslogdata` | Your actual NAS persistent storage directory |
| `YOUR_UI_ACCOUNT@hotmail.com` | An illustrative former console-notification recipient; not needed for the final pipeline |

The example MAC addresses are diagnostic illustrations only. Do not configure your devices to use them.

## 1. Overview and verified results

This document describes a working home-lab alert pipeline. It was tested with the versions listed below. `lab.example` is a reserved example domain used here for the private LAN namespace; replace it with your internal domain. The external mailbox is a **personal Outlook.com account**, not a Microsoft 365 mailbox.

The working route is:

1. UniFi exports activity events as CEF over UDP 514 to a syslog-ng container on QNAP.
2. syslog-ng filters selected event names and submits email over LAN SMTP port 25 to Zentyal.
3. Zentyal retains a local copy in `alerts@lab.example` and forwards another copy to `YOUR_OUTLOOK_ACCOUNT@outlook.com`.
4. Zentyal's Postfix SMTP client uses the `sasl-xoauth2` plugin to authenticate to Outlook.com over STARTTLS on port 587.
5. The outgoing SMTP address map rewrites `alerts@lab.example` to `YOUR_OUTLOOK_ACCOUNT@outlook.com`.
6. Custom Zentyal templates preserve the relay, authentication, forwarding and syslog HELO exception when Zentyal regenerates its configuration.

**Verified:** real WiFi connection events produced emails; a synthetic `|Internet Down|` event reached both local and Outlook mailboxes; the synthetic test still worked after `sudo zs mail restart` regenerated the configuration.

**Not yet verified:** an actual WAN outage producing the final filtered CEF event and email; every additional event name matching the router's actual CEF spelling; long-term token refresh through expiry/revocation; the original log bind-mount path. Do not present those as tested.

The browser's OAuth success page alone is not proof of SMTP delivery. We subsequently verified delivery into Outlook and then verified the complete syslog-to-Outlook path.

There is no remote access implied by this guide. Run commands on the named host. Tokens, passwords and private keys must never be pasted into a chat.

## 2. Inventory and addresses

| Component | Name / address | Recorded software or purpose |
| --- | --- | --- |
| Router | `LabRouter`, `192.168.50.1` | UniFi Cloud Gateway Max; UniFi Network `10.6.106` seen in CEF |
| NAS | `LabNAS`, `192.168.50.2` | QNAP TS-464, Container Station |
| Syslog container | Hostname `syslog`, `192.168.50.55/24` | `balabit/syslog-ng:latest`; running syslog-ng `4.12.0` |
| Local syslog DNS | `syslog.lab.example` → `192.168.50.55` | LAN DNS A record |
| Mail server | `zentyal.lab.example`, `192.168.50.10` | Ubuntu `24.04.5 LTS` (Noble), Zentyal, Postfix `3.8.6`, Dovecot |
| Local mailbox | `alerts@lab.example` | Zentyal mailbox; local delivery retained |
| External mailbox and authenticated sender | `YOUR_OUTLOOK_ACCOUNT@outlook.com` | Personal Microsoft account |
| AP | U7 Pro, `192.168.50.47` | Firmware `8.7.11`; useful WiFi event test source |

The image tag `latest` can change. `4.12.0` is the version observed in this deployment, not a promise about future pulls. When rebuilding, record the actual image digest and syslog-ng version; use a known image version/digest if reproducibility is important.

### Microsoft app registration

| Field | Example registration metadata |
| --- | --- |
| Display name | `Zentyal SMTP relay` |
| Application/client ID | `REPLACE_WITH_APPLICATION_CLIENT_ID` |
| Tenant ID containing the registration | `REPLACE_WITH_TENANT_ID` |
| Object ID | `REPLACE_WITH_OBJECT_ID` |
| Supported account type | Personal Microsoft accounts only |
| Allow public client flows | Yes |
| Client secret | None; empty string in plugin configuration |
| Token authority | `https://login.microsoftonline.com/consumers/oauth2/v2.0/token` |

These IDs are identifiers, not credentials. The refresh/access tokens are credentials and are intentionally absent from this document.

The Entra tenant contains the **application registration**. The **mailbox** remains a personal Outlook.com mailbox. The personal mailbox owner authorizes that app through the consumer login endpoint. Do not substitute the application's tenant ID for `consumers` in the working flow. Reuse your own existing registration if appropriate, or create one and use its client ID consistently below.

## 3. Scope, prerequisites and rebuild order

This guide rebuilds the alert/email integration. It does not reconstruct the entire NAS, router or Zentyal installation. Install/configure those products normally first, restore LAN DNS and networking, and create the local mailbox `alerts@lab.example` using Zentyal's mail module.

Rebuild in this order:

1. Establish unique addresses and correct LAN reachability.
2. Create persistent syslog configuration/log folders and the container.
3. Configure UniFi remote activity/syslog export; prove receipt before configuring email.
4. Prove local Zentyal mail delivery; apply the narrowly scoped HELO exception.
5. Create/reuse the Microsoft app registration, install OAuth support and obtain a token.
6. Configure Outlook relay, sender mapping and mailbox forwarding.
7. Test direct Outlook delivery and the full syslog pipeline.
8. Persist the working values in Zentyal's custom template and retest after regeneration.

Only the NAS/router/syslog container need the appropriate LAN paths. Outlook delivery needs working DNS, HTTPS to Microsoft token endpoints, and outbound TCP 587 from Zentyal. SMTP port 25 here is used **inside the LAN**; opening it on the internet is unnecessary.

## 4. QNAP container and persistent storage

The user created `syslogdata` on the QNAP share `NASShare`, with `conf/syslog-ng.conf` and empty `conf/conf.d` and `conf/patterndb.d` directories.

Use the following rebuild layout:

| Host directory | Container directory | Purpose |
| --- | --- | --- |
| `/share/NASShare/syslogdata/conf` | `/etc/syslog-ng` | Persistent configuration |
| `/share/NASShare/syslogdata/log` | `/var/log` | Persistent log files |

**Storage qualification:** the configuration share/folder was confirmed, but the final original second bind-mount source was not printed in the conversation. The `log` path above is the recommended explicit rebuild layout. Verify the actual QNAP absolute share path in File Station/Container Station; QNAP may use a different resolved volume path. Create both host directories before adding the mounts. Mount directories, not a missing file path accidentally turned into a directory.

Container Station settings:

- Image: `balabit/syslog-ng` (observed `latest` running version 4.12.0).
- Hostname: `syslog`. The Container Station hostname UI rejected dots; an FQDN hostname was not necessary after the HELO fix.
- Give the container its own LAN address `192.168.50.55/24`, reachable by the router, APs, switches and Zentyal. This deployment had that address directly on container `eth0`; it did not rely on sending syslog to the NAS's `192.168.50.2`.
- Set the default gateway/DNS appropriate to the LAN (router `192.168.50.1`, local DNS as configured).
- Add the two persistent mounts above and an appropriate automatic restart policy.
- The image advertised `514/udp`, `601/tcp`, `6514/tcp`. These image “exposed ports” are metadata, not proof of host publishing or a firewall rule. Preserve the actual reachable LAN networking arrangement. The working UniFi export uses **UDP 514**.

The config below also opens TCP 601 and TLS 6514 through `default-network-drivers()`. No syslog TLS certificate/key was configured, so 6514 is not a working TLS service. Do not select it for the router without setting up TLS properly.

### Minimal tools

Inside this Debian-based container, tools can be installed if needed:

```bash
apt-get update
apt-get install -y iproute2 tcpdump procps util-linux
```

`ss`/`ip` are supplied by `iproute2`; `ps` by `procps`; `logger` by `util-linux`. Diagnostic packages installed interactively disappear when the container is recreated unless incorporated into a custom image. A container terminal is sufficient; SSH inside the container is not required. If SSH is deliberately wanted, the Debian/Ubuntu server package is `openssh-server`, but merely installing it does not arrange a persistent running SSH service.

Do **not** run these Debian package commands on the QNAP host. The NAS host did not have `apt-get` or `tcpdump` in this session.

## 5. Complete syslog-ng configuration

Save this as the host-mounted `conf/syslog-ng.conf` (container `/etc/syslog-ng/syslog-ng.conf`). It retains the original container logging structure and adds the final alert filter and SMTP destination.

```conf
@version: 4.12
@include "scl.conf"

source s_local {
    internal();
};

source s_network {
    default-network-drivers();
};

destination d_local {
    file("/var/log/messages");
    file("/var/log/messages-kv.log"
         template("$ISODATE $HOST $(format-welf --scope all-nv-pairs)\n")
         frac-digits(3));
};

log {
    source(s_local);
    source(s_network);
    destination(d_local);
};

filter f_email_test {
    match("|Internet Down|" value("MESSAGE") type("string") flags("substring"))
    or match("|VPN User Connected|" value("MESSAGE") type("string") flags("substring"))
    or match("|Fan Issue Detected|" value("MESSAGE") type("string") flags("substring"))
    or match("|Device Update|" value("MESSAGE") type("string") flags("substring"))
    or match("|Power Supply Failure|" value("MESSAGE") type("string") flags("substring"))
    or match("|IP Address Conflict|" value("MESSAGE") type("string") flags("substring"));
};

destination d_email_test {
    smtp(
        host("192.168.50.10")
        port(25)
        from("Syslog alerts" "alerts@lab.example")
        to("Administrator" "alerts@lab.example")
        subject("[UniFi alert] ${HOST}")
        body("Host: ${HOST}\nTime: ${ISODATE}\n\n${MESSAGE}\n")
    );
};

log {
    source(s_network);
    filter(f_email_test);
    destination(d_email_test);
};
```

The historic names `f_email_test` and `d_email_test` are retained intentionally for continuity with counters and troubleshooting commands. They are now used for the actual selected alerts.

The pipe delimiters match a CEF event-name field. Example observed real CEF:

```text
CEF:0|Ubiquiti|UniFi Network|10.6.106|400|WiFi Client Connected|1|UNIFIcategory=Client Devices ...
```

The additional filter names are those requested by the administrator. Verify actual CEF spellings if an event is not matched. Internet Restored was not added. The final filter does not email WiFi connection events; WiFi was only the earlier test condition.

There is no deduplication or alert cooldown configured here. Every matching received event is an email candidate. Log rotation/retention was not configured during this session; add an appropriate retention policy separately rather than allowing unbounded log growth.

### Validate and activate

Inside the syslog container:

```bash
syslog-ng --no-caps -s
timeout 10 syslog-ng-ctl reload
syslog-ng-ctl config --preprocessed | grep -A 15 'filter f_email_test'
```

Inspect command results. If the reload hangs or fails, restart the container in Container Station and verify the active configuration again. Reload eventually worked in the final session, but earlier control commands hung. The active configuration is authoritative: an edited file can be correct while the process still runs the old filter.

syslog-ng runs as the main container process. Stopping it can stop the container; there need not be a separate `systemctl` service manager inside the image.

The warning `Error setting capabilities ... Operation not permitted` occurred and was not the cause of delivery failure. The missing X.509 keypair warning related to unused TLS syslog 6514.

## 6. Configure and verify UniFi export

In UniFi Network, configure the remote syslog/SIEM destination as `192.168.50.55`, UDP port `514`, and select the relevant activity-log categories. The administrator selected all 12 available activity-log types. UI labels may differ in future versions; preserve **activity/CEF export**, not just low-level AP/device debug logging.

We received both raw device logs (e.g. `hostapd`, `dnsmasq`) and CEF activity events. A router GUI activity entry is not itself proof that a packet reached syslog-ng.

Inside the container:

```bash
grep -F 'CEF:' /var/log/messages | tail -n 10
grep -F 'CEF:' /var/log/messages | cut -d '|' -f 6 | sort | uniq -c | sort -nr
```

To watch future CEF events:

```bash
tail -n 0 -F /var/log/messages | grep --line-buffered -F 'CEF:'
```

For packet-level diagnosis, run in the container while generating an event:

```bash
tcpdump -ni any -nn -A 'src host 192.168.50.1 and udp dst port 514'
```

To compare with the router's outgoing packets, run on **LabRouter**:

```bash
tcpdump -ni any -nn -A 'dst host 192.168.50.55 and udp dst port 514'
```

The `any` interface on the router showed the same packet on `br0` and `switch0.1`. That is not proof of duplicate application sends. Some CEF packets appeared as `(invalid)` in tcpdump's syslog decoding, but syslog-ng accepted them and emails were delivered. Do not diagnose loss solely from that label.

Container timestamps were UTC while router message timestamps were Stockholm local time (two hours ahead during the test). Match timestamps accordingly; this difference did not indicate delayed delivery.

## 7. Critical networking fix: duplicate IP address

The intermittent delivery problem was a **real duplicate address**, not a syslog filter or SMTP problem. Router traffic sometimes went to another MAC using `192.168.50.55`.

Observed during diagnosis:

| Address owner | MAC at the time |
| --- | --- |
| Working syslog container | `02:00:00:00:00:55` |
| Conflicting device/container | `02:00:00:00:00:99` |

A rebuilt container may have a different MAC; these are historical evidence, not values to force.

On the router:

```bash
ip route get 192.168.50.55
ip neigh show 192.168.50.55
```

Inside the container:

```bash
ip -br addr
ip link show eth0
```

Compare the router's neighbour MAC with the current container MAC. Clearing the router entry temporarily selected the correct MAC but it then reverted to the conflicting MAC:

```bash
ip neigh del 192.168.50.55 dev br0
ping -c 2 192.168.50.55
ip neigh show 192.168.50.55
```

That command was diagnostic, not the permanent fix. The administrator corrected the duplicate address, after which received CEF events and real WiFi alert emails became reliable. Ensure `192.168.50.55` is unique and excluded from conflicting DHCP/static assignments.

An empty neighbour entry on the QNAP host did not disprove the router/container conflict. The decisive comparison was on the actual sender, LabRouter.

## 8. Zentyal local SMTP and HELO exception

Prerequisite: Zentyal accepts LAN SMTP on `192.168.50.10:25` and the mailbox `alerts@lab.example` exists. Keep firewall access appropriate to the LAN; test from the syslog container:

```bash
timeout 5 bash -c 'exec 3<>/dev/tcp/192.168.50.10/25; head -n 1 <&3'
```

Expected banner:

```text
220 zentyal.lab.example ESMTP
```

The syslog-ng SMTP driver introduced itself as `syslog`, and Zentyal rejected mail with:

```text
504 5.5.2 <syslog>: Helo command rejected: need fully-qualified hostname
```

Changing global syslog-ng hostname options did not resolve that SMTP HELO. We used a narrow Postfix exception for the container's IP, placed **after relay and sender validation**, rather than disabling HELO checks globally.

On Zentyal, first save the original value if rebuilding:

```bash
sudo postconf smtpd_recipient_restrictions |
  sudo tee /etc/postfix/syslog-original-recipient-restrictions.txt >/dev/null

sudo tee /etc/postfix/syslog-helo-exception.cidr >/dev/null <<'EOF'
192.168.50.55/32 OK
EOF
```

No `postmap` compilation is required for a CIDR table.

The original recorded recipient restriction list was:

```text
permit_sasl_authenticated, permit_mynetworks, reject_unauth_destination, reject_non_fqdn_sender, reject_unknown_sender_domain, reject_invalid_helo_hostname, reject_non_fqdn_helo_hostname, check_helo_access pcre:/etc/postfix/helo_checks.pcre
```

For the same Zentyal baseline, apply the working list:

```bash
sudo postconf -e 'smtpd_recipient_restrictions = permit_sasl_authenticated, permit_mynetworks, reject_unauth_destination, reject_non_fqdn_sender, reject_unknown_sender_domain, check_client_access cidr:/etc/postfix/syslog-helo-exception.cidr, reject_invalid_helo_hostname, reject_non_fqdn_helo_hostname, check_helo_access pcre:/etc/postfix/helo_checks.pcre'
sudo postfix check
sudo postfix reload
```

For a newer/different baseline, inspect `postconf smtpd_recipient_restrictions smtpd_relay_restrictions` and preserve its intended controls rather than blindly replacing unrelated restrictions. Do not move the `OK` exception ahead of relay authorization. The exception applies only to `192.168.50.55`, and only bypasses the remaining HELO-related checks in this recorded list.

## 9. Install Outlook OAuth support

On the recorded Ubuntu 24.04 server, `apt-cache policy sasl-xoauth2` initially found no package. The plugin author's Ubuntu PPA was used:

```bash
sudo add-apt-repository ppa:sasl-xoauth2/stable
sudo apt update
sudo apt install sasl-xoauth2
```

If `add-apt-repository` is absent on a fresh Ubuntu install, install `software-properties-common`. Check the upstream project and supported Ubuntu releases when rebuilding on a different version. This is a third-party plugin/PPA, not a built-in Zentyal GUI OAuth feature.

### Register or reuse the application

In your own Entra tenant's App registrations, create `Zentyal SMTP relay` (or reuse your own equivalent app):

- Supported account types: **Personal Microsoft accounts only**.
- Do not configure a redirect URI for this device-flow setup.
- Authentication: **Allow public client flows = Yes**.
- No client secret is used.

The plugin requests the Outlook SMTP delegated scope `https://outlook.office.com/SMTP.Send` and offline access for refresh tokens. A separate API-permission UI change was not recorded as necessary in this successful session. If a future flow reports a missing permission/consent problem, follow the current upstream and Microsoft documentation for delegated SMTP.Send, not Graph Mail.Send or application-only Exchange permissions.

Creating the app in a tenant does not migrate your personal mailbox to that tenant, and does not require an M365 mailbox.

### Configuration and token acquisition — on Zentyal

Replace `REPLACE_WITH_APPLICATION_CLIENT_ID` below with your own app client ID in both the JSON and the token command:

```bash
sudo tee /etc/sasl-xoauth2.conf >/dev/null <<'EOF'
{
  "client_id": "REPLACE_WITH_APPLICATION_CLIENT_ID",
  "client_secret": "",
  "token_endpoint": "https://login.microsoftonline.com/consumers/oauth2/v2.0/token"
}
EOF

sudo install -d -m 700 -o postfix -g postfix \
  /var/spool/postfix/etc/tokens

sudo cp /etc/sasl-xoauth2.conf \
  /var/spool/postfix/etc/sasl-xoauth2.conf

sudo sasl-xoauth2-tool get-token outlook \
  /var/spool/postfix/etc/tokens/YOUR_OUTLOOK_ACCOUNT@outlook.com \
  --client-id=REPLACE_WITH_APPLICATION_CLIENT_ID \
  --tenant=consumers \
  --use-device-flow
```

The tool displays a website and a device code. Open the displayed Microsoft site on your PC, enter the code, sign in as **YOUR_OUTLOOK_ACCOUNT@outlook.com**, and approve access. Wait for token acquisition to finish in the SSH terminal. Microsoft displayed “All done! You're now signed in to Zentyal SMTP relay” in the successful run.

Then:

```bash
sudo chown postfix:postfix \
  /var/spool/postfix/etc/tokens/YOUR_OUTLOOK_ACCOUNT@outlook.com
sudo chmod 600 \
  /var/spool/postfix/etc/tokens/YOUR_OUTLOOK_ACCOUNT@outlook.com
```

Postfix's outgoing SMTP service was confirmed chrooted:

```bash
sudo postconf -M smtp/unix
# smtp unix - - y - - smtp
```

Therefore `/etc/tokens/YOUR_OUTLOOK_ACCOUNT@outlook.com` as seen by the SMTP process corresponds to the real file `/var/spool/postfix/etc/tokens/YOUR_OUTLOOK_ACCOUNT@outlook.com`. Keep that distinction when troubleshooting. Recheck on a new installation rather than assuming the same chroot setting.

The plugin is designed to refresh tokens automatically, requiring writable token storage. Microsoft revocation/account-policy changes can require repeating device authorization. Extended unattended refresh was not separately tested during this short session.

## 10. Configure Postfix's Outlook relay and sender rewrite

Run on Zentyal:

```bash
sudo cp -a /etc/postfix/main.cf /etc/postfix/main.cf.before-outlook

sudo tee /etc/postfix/outlook-sasl >/dev/null <<'EOF'
[smtp-mail.outlook.com]:587 YOUR_OUTLOOK_ACCOUNT@outlook.com:/etc/tokens/YOUR_OUTLOOK_ACCOUNT@outlook.com
EOF

sudo postmap hash:/etc/postfix/outlook-sasl
sudo chmod 600 /etc/postfix/outlook-sasl /etc/postfix/outlook-sasl.db

sudo tee /etc/postfix/outlook-generic >/dev/null <<'EOF'
alerts@lab.example YOUR_OUTLOOK_ACCOUNT@outlook.com
EOF

sudo postmap hash:/etc/postfix/outlook-generic

sudo postconf -e \
  'relayhost = [smtp-mail.outlook.com]:587' \
  'smtp_sasl_auth_enable = yes' \
  'smtp_sasl_password_maps = hash:/etc/postfix/outlook-sasl' \
  'smtp_sasl_mechanism_filter = xoauth2' \
  'smtp_sasl_security_options = noanonymous' \
  'smtp_sasl_tls_security_options = noanonymous' \
  'smtp_tls_security_level = encrypt' \
  'smtp_tls_CAfile = /etc/ssl/certs/ca-certificates.crt' \
  'smtp_generic_maps = hash:/etc/postfix/outlook-generic'

sudo install -d /var/spool/postfix/etc/ssl/certs
sudo cp /etc/ssl/certs/ca-certificates.crt \
  /var/spool/postfix/etc/ssl/certs/ca-certificates.crt

sudo postfix check
sudo postfix reload
```

`smtp_*` settings configure Postfix's **outgoing client**. `smtpd_*` configure its **incoming server**. Do not accidentally change inbound TLS certificates while configuring the Outlook client.

This sender rewrite is exact: `alerts@lab.example` becomes `YOUR_OUTLOOK_ACCOUNT@outlook.com` on outbound SMTP. It does not rewrite every arbitrary sender. The alert sources here use `alerts@lab.example`. Outlook and syslog-ng authentication are different: syslog-ng submits locally without SMTP authentication in this setup; Postfix authenticates to Outlook using OAuth2.

### Direct delivery test — on Zentyal

```bash
printf 'From: alerts@lab.example\nTo: YOUR_OUTLOOK_ACCOUNT@outlook.com\nSubject: Zentyal Outlook relay test\n\nTesting delivery through Outlook OAuth2.\n' |
  /usr/sbin/sendmail -f alerts@lab.example YOUR_OUTLOOK_ACCOUNT@outlook.com

sudo tail -n 30 /var/log/mail.log
sudo postqueue -p
```

The administrator received this test in Outlook. That established a working OAuth SMTP relay before testing forwarding.

## 11. Forward the local mailbox while retaining a local copy

Earlier we temporarily redirected `YOUR_UI_ACCOUNT@hotmail.com` (the example UI account's address) into the local mailbox to test console notifications. That redirect was explicitly removed at the administrator's request. The historic filename remains but its purpose changed.

On a fresh rebuild create this map; on an existing server back it up and preserve any unrelated aliases:

```bash
sudo tee /etc/postfix/unifi-test-aliases >/dev/null <<'EOF'
alerts@lab.example alerts@lab.example, YOUR_OUTLOOK_ACCOUNT@outlook.com
EOF

sudo postmap hash:/etc/postfix/unifi-test-aliases
```

For the same baseline, the working alias configuration is:

```bash
sudo postconf -e 'virtual_alias_maps = hash:/etc/postfix/unifi-test-aliases, ldap:/etc/postfix/valiases.cf,ldap:/etc/postfix/useraliases.cf,ldap:/etc/postfix/groupaliases.cf'
sudo postfix check
sudo postfix reload
```

For a different install, first read `sudo postconf -h virtual_alias_maps` and prepend the new hash map to the existing Zentyal LDAP maps instead of guessing their paths.

Check:

```bash
sudo postmap -q 'alerts@lab.example' hash:/etc/postfix/unifi-test-aliases
sudo postconf virtual_alias_maps relayhost
```

Expected lookup result:

```text
alerts@lab.example, YOUR_OUTLOOK_ACCOUNT@outlook.com
```

The self-address on the right retains local mailbox delivery; the second address forwards externally. This is **one mailbox's forwarding rule**, not an all-domain catch-all. Because the administrator intends to have only this mailbox, it covers the required incoming alert mail. Nonexistent addresses such as `nonexistent@lab.example` remain invalid unless separately created.

The removed Hotmail redirect is not restored by this guide. Existing console notifications addressed to the UI account's Hotmail address therefore follow their actual recipient routing rather than being redirected into `per`.

## 12. Test the complete path

Inside the syslog container:

```bash
logger --udp --rfc3164 \
  --server 127.0.0.1 --port 514 \
  --tag email-test '|Internet Down| Outlook forwarding test'

grep -F 'Outlook forwarding test' /var/log/messages | tail -n 3
syslog-ng-ctl stats | grep -Ei 'smtp|d_email_test'
```

The pipe characters are required by the final filter. An earlier proposed message without them did not match. This logger command explicitly sends to the **network** source; a plain local `logger` invocation may use a different source than the email log path.

On Zentyal:

```bash
sudo tail -n 30 /var/log/mail.log
sudo postqueue -p
```

Confirm local delivery in the Zentyal mailbox and receipt in Outlook. This end-to-end synthetic test was successful. The synthetic text tests the email/filter pipeline but does not prove UniFi exports an actual WAN outage with exactly that event name.

To prove router export without disconnecting the WAN, temporarily include this clause in the filter and reload:

```conf
or match("|WiFi Client Connected|" value("MESSAGE") type("string") flags("substring"))
```

Reconnect a phone to WiFi and compare its actual CEF event and resulting email. This worked after the duplicate-IP fix. Remove the WiFi clause after testing to avoid unnecessary email volume.

During an actual internet outage, local logging and local Zentyal mailbox delivery can still work while powered LAN components remain reachable. External Outlook delivery cannot complete without a usable internet path; the mail should remain queued for later delivery. We did not test that complete outage/recovery scenario here.

## 13. Make the Postfix settings persistent in Zentyal

Zentyal regenerates `main.cf` from templates. `postconf -e` alone is not sufficient persistence.

The existing custom file was:

```text
/etc/zentyal/stubs/mail/main.cf.mas
```

It already contained previous TLS/certificate customizations and must be preserved. Never blindly copy a stock template over this working custom file.

On a **fresh** installation with no custom template, first create one from that installation's vendor template:

```bash
sudo mkdir -p /etc/zentyal/stubs/mail
sudo cp -a /usr/share/zentyal/stubs/mail/main.cf.mas \
  /etc/zentyal/stubs/mail/main.cf.mas
```

Do not run the copy above if the destination already contains customizations. Then, once the live Postfix settings are working, run the same persistence script used successfully in this session:

```bash
sudo python3 <<'PY'
from pathlib import Path
import re
import shutil
import subprocess
from datetime import datetime

path = Path("/etc/zentyal/stubs/mail/main.cf.mas")
keys = [
    "relayhost",
    "smtp_sasl_auth_enable",
    "smtp_sasl_password_maps",
    "smtp_sasl_mechanism_filter",
    "smtp_sasl_security_options",
    "smtp_sasl_tls_security_options",
    "smtp_tls_security_level",
    "smtp_tls_CAfile",
    "smtp_generic_maps",
    "virtual_alias_maps",
    "smtpd_recipient_restrictions",
]

values = {
    key: subprocess.check_output(
        ["postconf", "-h", key], text=True
    ).strip()
    for key in keys
}

backup = str(path) + ".before-outlook-" + datetime.now().strftime("%Y%m%d-%H%M%S")
shutil.copy2(path, backup)

pattern = re.compile(r"^\s*(" + "|".join(map(re.escape, keys)) + r")\s*=")
lines = [
    line for line in path.read_text().splitlines()
    if not pattern.match(line)
]

lines += ["", "# Custom Outlook relay, forwarding and syslog access"]
lines += [f"{key} = {values[key]}" for key in keys]
path.write_text("\n".join(lines) + "\n")

print("Backup:", backup)
print("Updated:", path)
PY
```

This captures working values, removes the corresponding single-line definitions from the current template, and appends unconditional settings. Unrelated template content, including inbound certificate settings, is retained. The script was tested with this deployment's template; inspect a changed future vendor/custom template for multiline definitions or new logic before applying it unchanged.

The hardcoded custom values now take precedence over the equivalent mail GUI controls. To change the relay/alias/security settings later, update the custom template as well as any live test configuration. After major Zentyal upgrades compare the custom template with the new vendor version so useful upstream changes are not silently missed.

Regenerate and verify:

```bash
sudo zs mail restart
sudo postfix check

sudo postconf relayhost smtp_sasl_mechanism_filter \
  smtp_sasl_password_maps smtp_generic_maps \
  virtual_alias_maps smtpd_recipient_restrictions
```

Repeat the synthetic syslog test from section 12. The administrator confirmed both mailbox deliveries still worked after this restart/regeneration. This is the verified persistent endpoint of the session.

## 14. Troubleshooting by stage

| Symptom | Check next | Known finding from this work |
| --- | --- | --- |
| GUI shows an event, no container packet | Compare router/container packet captures and neighbour MACs | Duplicate `192.168.50.55` sent traffic to the wrong MAC |
| Container sees raw AP logs but no relevant CEF | Inspect router activity export and router capture | Raw device logging and activity CEF are different streams |
| Message is in `/var/log/messages`, email `processed` stays unchanged | Inspect active filter and source | Old WiFi-only filter remained loaded; reload fixed it |
| Configuration file looks right but behavior is old | `syslog-ng-ctl config --preprocessed` | Running config differed from edited file |
| SMTP `queued` rises, `written` does not | Zentyal mail log and container internal errors | HELO `syslog` was rejected |
| Local delivery works, external delivery fails | Postfix queue and outbound SMTP logs | Before OAuth relay, direct MX connections on port 25 timed out |
| OAuth SMTP auth fails | Token exists, chroot path, ownership, configured client ID/authority | Use consumer authority and writable postfix-owned token |
| Works until Zentyal settings are saved | Custom template and generated `postconf` values | Live edits were not enough; custom template fixed persistence |

Useful commands **inside the container**:

```bash
syslog-ng-ctl stats | grep -Ei 'smtp|d_email_test'
syslog-ng-ctl config --preprocessed | grep -A 30 -B 3 'f_email_test'
grep -Ei 'smtp|d_email_test|error|refused|timed out' /var/log/messages | tail -n 30
ss -lntup
```

Counters are relative to the current process lifetime. `written` indicates SMTP handoff, not independent confirmation that Outlook placed the message in the inbox. A historic `dropped=1` remained from earlier failed tests and did not mean each new message was dropped. Compare before/after values for a fresh test.

Useful commands **on Zentyal**:

```bash
sudo tail -n 50 /var/log/mail.log
sudo postqueue -p
sudo postconf -n
sudo postconf -M smtp/unix
sudo postmap -q 'alerts@lab.example' hash:/etc/postfix/unifi-test-aliases
sudo ls -l /var/spool/postfix/etc/tokens/YOUR_OUTLOOK_ACCOUNT@outlook.com
```

For one mail's queue ID:

```bash
sudo grep -F 'QUEUE_ID_HERE' /var/log/mail.log
```

Only inspect token file metadata; do not print credentials. Postfix logs and messages may themselves contain personal addresses and network/device details.

Postfix's “backwards-compatible default settings” messages were informational in this session. They did not block delivery. Do not paste those output lines back into the shell as commands; an earlier accidental paste produced `Command 'postfix:' not found`.

Older messages already queued for Hotmail do not automatically get their recipients rewritten when a new alias map is installed. A new test is preferable to treating an old queue entry as proof of current behavior. `postqueue -f` retries delivery; it does not redo recipient alias expansion. Inspect old messages before deciding whether to requeue or remove individual entries; do not delete the entire queue indiscriminately.

## 15. Related UniFi console SMTP configuration (optional)

This is separate from the final syslog alert route. It was made to work earlier:

- SMTP server: `zentyal.lab.example` or `192.168.50.10`.
- Port 587 with STARTTLS.
- Authenticated username: **`alerts@lab.example`**, not `per`.
- Server certificate: CN `zentyal.lab.example`, SAN DNS `zentyal.lab.example` and IP `192.168.50.10`, issued by the LAN Zentyal CA.
- The CA was installed into the router's system trust store.
- UniFi Core required this systemd environment override:

```ini
# /etc/systemd/system/unifi-core.service.d/90-lab-ca.conf
[Service]
Environment="NODE_EXTRA_CA_CERTS=/etc/ssl/certs/ca-certificates.crt"
```

On the router, after installing the CA as `/usr/local/share/ca-certificates/lab-ca.crt`:

```bash
update-ca-certificates
systemctl daemon-reload
systemctl restart unifi-core
systemctl is-active unifi-core
```

This resolved TLS trust for UniFi Core; the subsequent authentication failure was fixed by using the full mailbox username. There was no running Java process or Java truststore for this native UniFi deployment. Do not repeat the earlier unsuccessful Java/keytool investigation.

Console notifications such as **Backup Created** and **Admin Accessed UniFi Console** were sent through this SMTP path. Alarm Manager's Internet Down event did not produce SMTP mail in the observed tests. That led to the syslog-ng solution. Do not confuse successful console-notification SMTP with proof that Alarm Manager sends every alarm through it.

QNAP's own notification emails also worked using its built-in Outlook browser authorization. That proved the personal Outlook account could send email, but was not used as a LAN SMTP relay by this solution.

## 16. Certificates and earlier webadmin work: boundaries

The Zentyal HTTPS webadmin is on 8443. A manually applied certificate with SAN resolved the browser's domain mismatch. nginx originally referenced a combined certificate/key PEM at `/var/lib/zentyal/conf/ssl/ssl.pem`. Apache on 80/443 was separately configured to redirect the base hostname to `https://zentyal.lab.example:8443/`, and LAN firewall access to 80 was required. These web redirects do not configure SMTP delivery.

Postfix originally used `/etc/postfix/sasl/postfix.pem` for both inbound TLS certificate and key; a suitable SAN certificate was subsequently deployed. The exact final inbound certificate paths and the complete previous certificate-modified template were not exported in the final session, so they are not fabricated here. Before rebuilding/upgrading, back up the actual template and inspect:

```bash
sudo postconf smtpd_tls_cert_file smtpd_tls_key_file smtpd_tls_chain_files
sudo grep -nE 'smtpd_tls_(cert_file|key_file|chain_files)' \
  /etc/zentyal/stubs/mail/main.cf.mas
```

The final syslog-to-Zentyal leg uses plain SMTP inside the LAN, so it does not depend on UniFi trusting Zentyal's private CA. The Outlook leg uses Microsoft's public TLS certificate and the system CA bundle. Preserve inbound TLS configuration for other clients, but keep these trust paths distinct.

## 17. Backups and recovery

Back up these items together, with secrets in protected storage:

- QNAP `syslogdata/conf` and persistent logs as desired; Container Station networking, mounts, image/version and restart settings.
- `/etc/zentyal/stubs/mail/main.cf.mas` and its timestamped backups.
- `/etc/postfix/unifi-test-aliases` and its `.db` map.
- `/etc/postfix/syslog-helo-exception.cidr` and original restriction snapshot.
- `/etc/postfix/outlook-sasl` and `.db`.
- `/etc/postfix/outlook-generic` and `.db`.
- `/etc/sasl-xoauth2.conf` and chroot copy.
- **Sensitive:** `/var/spool/postfix/etc/tokens/YOUR_OUTLOOK_ACCOUNT@outlook.com`; preserve permissions. Reauthorization is preferable if a token backup is unavailable/revoked.
- Actual inbound certificates/private keys and local CA material if the broader mail/webadmin setup is being restored.
- Microsoft registration details and the personal account's ability to sign in/approve authorization.
- Router's private-CA certificate and UniFi Core systemd override if restoring optional console SMTP.

Before changing a working installation, retain generated `main.cf` and the custom template. The recorded backups include `/etc/postfix/main.cf.before-outlook` and custom-template backups named `main.cf.mas.before-outlook-YYYYMMDD-HHMMSS`.

If a template change prevents regeneration, restore the appropriate template backup, then run `sudo zs mail restart`. If only a live test edit is faulty, restore the relevant saved `main.cf` and reload Postfix, but remember that a later Zentyal regeneration uses the template again. Choose the backup corresponding to the state you intend to restore; restoring the pre-Outlook file removes the working relay settings.

After restoring map text files, run `postmap hash:/path/to/map` for each hash map. Do not compile the CIDR table. Restore token ownership to `postfix:postfix` and mode 600, check `postfix check`, and rerun a direct and an end-to-end test.

## 18. Acceptance checklist

- [ ] `192.168.50.55` is unique; router neighbour MAC matches the actual container.
- [ ] Configuration and logs are mounted persistently; no reliance on ephemeral container-only edits.
- [ ] A real router CEF activity event reaches the container and `/var/log/messages`.
- [ ] The running syslog filter contains all six requested event names.
- [ ] Zentyal accepts the syslog SMTP client's HELO without broadening relay authorization.
- [ ] Token acquisition succeeds for the personal Outlook mailbox and token permissions are correct.
- [ ] Direct Zentyal → Outlook test is received.
- [ ] `alerts@lab.example` alias lookup returns both local and Outlook recipients.
- [ ] Synthetic `|Internet Down|` message arrives in both mailboxes.
- [ ] Repeat that test after `sudo zs mail restart`.
- [ ] Later: verify a real WAN outage/recovery and the exact remaining event spellings.
- [ ] Later: establish log rotation, backups and long-term token-refresh observation.

## 19. Sources for future version checks

The configuration above is a record of the tested deployment, not a promise that future UI/package versions remain identical. Consult primary documentation before adapting it:

- [sasl-xoauth2 project — installation, Outlook device flow, token storage and chroot](https://github.com/tarickb/sasl-xoauth2)
- [Microsoft — Outlook.com SMTP settings](https://support.microsoft.com/en-us/outlook/pop-imap-and-smtp-settings-for-outlook-com)
- [Microsoft — SMTP OAuth, delegated scopes and refresh tokens](https://learn.microsoft.com/en-us/exchange/client-developer/legacy-protocols/how-to-authenticate-an-imap-pop-smtp-application-by-using-oauth)
- [Microsoft — application registration and personal-account audience](https://learn.microsoft.com/en-us/entra/identity-platform/quickstart-register-app)
- [Zentyal — custom templates and hooks](https://doc.zentyal.org/en/appendix-c.html)
- [Zentyal — mail module](https://doc.zentyal.org/8.0/en/mail.html)
- [syslog-ng — SMTP destination](https://syslog-ng.github.io/admin-guide/070_Destinations/240_SMTP/README.html)
- [UniFi — System Logs/SIEM integration](https://help.ui.com/hc/en-us/articles/33349041044119-UniFi-System-Logs-SIEM-Integration)

## 20. Suggested prompt for a future chat

> Read the attached UniFi-Syslog-Zentyal-Outlook-Public-Guide.md in full. It describes my verified alert pipeline as of 4 October 2026. I want to rebuild it / diagnose the following issue: [describe current issue]. Use the tested settings and known failure history as your starting point. Ask for current command output where needed, verify current upstream requirements if versions have changed, and do not assume that a synthetic test proves the router's real outage event. I use a personal Outlook account, not M365; the example domain and addresses must be replaced with my own LAN settings. Keep instructions separated by host and do not ask me to disclose tokens or passwords.

