---
title: "Solving Android CA Trust Issues for Local Homelab Services with Route 53, Let's Encrypt, and BIND9"
date: 2026-07-25T13:25:00-07:00
draft: false
categories: ["Computers"]
tags:
  - homelab
  - android
  - letsencrypt
  - certbot
  - route53
  - bind9
  - nginx
  - docker
  - openproject
  - nextcloud
  - dns
  - sysadmin
description: "How to get native Android apps for Nextcloud and OpenProject working over trusted HTTPS without rooting your phone, exposing internal IPs, or opening router ports."
---

I run OpenProject and Nextcloud on my home network, both behind an internal
certificate authority I run with PicoCA. Browsers on my laptop are happy to trust
that root cert once I install it, but the
official Android apps for both services flatly refuse to talk to a server whose
certificate they don't recognize. No amount of "install anyway" in Android's
certificate settings fixes it, and I didn't want to root my phone just to sync a
project board and a file share.

Working through this turned into a real rabbit hole — DNS, IAM policy, Nginx syntax
deprecations, a Docker symlink trap, and eventually an OpenProject version upgrade I
wasn't planning on doing that day. I used an AI assistant (Gemini) as a sounding
board while I worked through it, and this post is the writeup of what I actually did
and why, with the domain names swapped for `example.com` and `example.local`.

## Why Android won't just trust my CA

Since Android 7 (API level 24), apps are split into two trust worlds. There's the
**system** trust store — the CAs baked into the OS — and the **user** trust store,
where anything you install yourself through Settings lands. By default, apps built
against a modern `targetSdkVersion` only trust the system store. Installing my
PicoCA root as a user certificate makes it show up correctly in a browser, but
OpenProject's and Nextcloud's native apps use Android's standard network stack with
its default network security config, which ignores user certificates entirely.

The paths around this all had problems:

- **Root the phone** and use a Magisk module to promote the CA into the system
  store. Every app would trust it, but I didn't want to root my daily driver phone
  for this.
- **Decompile and re-sign the APKs** with a patched `network_security_config.xml`
  that explicitly trusts user CAs. This works, but it means maintaining a
  hand-patched build of someone else's app forever, through every update.
- **Get a real publicly-trusted certificate.** Let's Encrypt won't touch `.local` —
  it's not a valid public TLD, and DNS-01 validation requires a domain a public CA
  can actually verify ownership of. My internal domain, `nextcloud.example.local`,
  was a non-starter from the get-go.

The way out was to stop trying to make Android trust a private CA at all, and
instead get a real Let's Encrypt certificate for services that never touch the
public internet.

## Split-horizon DNS with a wildcard cert

I already own a real domain on Route 53. The trick is that owning a domain and
*hosting a public website on it* are two different things — nothing requires me to
publish A records for `openproject.example.com` or `nextcloud.example.com` at all.

The plan:

- **Route 53 stays empty.** No A or CNAME records for either service. The only
  thing that ever touches Route 53 is a temporary `_acme-challenge` TXT record that
  Certbot creates and deletes on every renewal, via the DNS-01 challenge. Anyone
  running `dig openproject.example.com` from the outside world gets `NXDOMAIN`.
- **My local BIND9 resolver overrides those two names** to the internal LAN IP
  running Nginx.
- **One wildcard certificate**, `*.example.com`, covers every internal service I
  ever add. No more juggling per-service PicoCA leaf certs.

My first instinct was actually simpler and wrong: create public CNAME records
pointing `openproject.example.com` at `openproject.example.local`, so a `dig` from
the outside would resolve to something meaningless without exposing an IP. That
doesn't work, and not just because it leaks the fact that a service called
"openproject" exists. A CNAME is an alias, not a shortcut — the resolver a client
is actually asking (Google's `8.8.8.8`, a carrier's DNS, whatever) has to be able to
resolve the *target* too. `.local` only means anything to my own BIND9 server, so
any public resolver hits the CNAME, tries to chase it to `example.local`, and comes
back with `NXDOMAIN`. Recursive resolution happens server-side, on whichever
resolver the client asked — not on the client. If I want split-horizon DNS to
actually route traffic, the override has to happen on a resolver my devices are
actually using, which means my own BIND9 server.

## Getting BIND9 to override selectively

I already run BIND9 as my network's resolver, using views to apply different rules
to different clients (I have a `blocked` view that null-routes certain domains for
specific devices). I wanted `openproject.example.com` and `nextcloud.example.com`
to resolve to my Nginx box's LAN IP, while every other name — including anything
else under `example.com` — passed through to public DNS normally.

That selective override is what a BIND **Response Policy Zone (RPZ)** is for: a
small zone of override rules that BIND consults before falling back to normal
resolution, rather than making BIND authoritative for the whole `example.com` zone
(which would silently `NXDOMAIN` every subdomain I hadn't explicitly listed).

My first attempt at wiring it up threw:

```
rpz 'rpz.local' is not a primary or a secondary zone
```

The cause was that I'd declared `response-policy { zone "rpz.local"; };` inside the
global `options {}` block, but BIND was already using named `view`s for split
client policy (`blocked` and `default`). Response policy has to be declared inside
each view where you want it applied — a global declaration can't reach into a zone
that's only defined inside a view. The fix was to move `response-policy` into each
view block and define `rpz.local` as a zone inside those same views:

```
view "default" {
    match-clients { any; };

    response-policy {
        zone "rpz.local";
    };

    zone "rpz.local" {
        type master;
        file "/var/lib/bind/db.rpz.local";
        allow-query { localhost; 192.168.0.0/16; };
    };
};
```

And the RPZ zone file itself just lists the two overrides:

```
$TTL 60
@       IN      SOA     localhost. root.localhost. ( 2026072401 1h 15m 1w 1d )
        IN      NS      localhost.

openproject.example.com  IN  A   192.168.x.x
nextcloud.example.com    IN  A   192.168.x.x
```

With that in place, a device using my BIND9 server gets the LAN IP for those two
names and normal public resolution for everything else — `dig` for any other
`example.com` subdomain sails straight through.

## Locking down the IAM user for DNS-01

Certbot's `certbot-dns-route53` plugin needs API access to create and delete the
`_acme-challenge` TXT record on every renewal. I wanted an IAM policy scoped as
tightly as possible: this key should not be able to touch anything except TXT
records in my one hosted zone.

Route 53's `ChangeResourceRecordSets` doesn't have a record-type parameter by
default, but it does expose a condition key,
`route53:ChangeResourceRecordSetsRecordTypes`, that lets you restrict *which record
types* an API call is allowed to modify. Because a single `ChangeResourceRecordSets`
call can bundle multiple changes, that key is multi-valued, and IAM requires an
explicit set qualifier (`ForAllValues` or `ForAnyValue`) before it'll accept the
condition — leaving it off produces a "missing qualifier" policy validation error.
`ForAllValues:StringEquals` says *every* record type touched by the call must be
`TXT`, which is exactly what I want:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "CertbotListAndGetHostedZone",
            "Effect": "Allow",
            "Action": [
                "route53:GetHostedZone",
                "route53:ListResourceRecordSets"
            ],
            "Resource": "arn:aws:route53:::hostedzone/Z0123456789EXAMPLE"
        },
        {
            "Sid": "CertbotManageTXTRecordsOnly",
            "Effect": "Allow",
            "Action": "route53:ChangeResourceRecordSets",
            "Resource": "arn:aws:route53:::hostedzone/Z0123456789EXAMPLE",
            "Condition": {
                "ForAllValues:StringEquals": {
                    "route53:ChangeResourceRecordSetsRecordTypes": ["TXT"]
                }
            }
        },
        {
            "Sid": "CertbotGlobalActions",
            "Effect": "Allow",
            "Action": [
                "route53:ListHostedZones",
                "route53:GetChange"
            ],
            "Resource": "*"
        }
    ]
}
```

Scoping the resource ARN to my one hosted zone means this key can never touch any
other zone in the account, and the TXT-only condition means that even if it leaked,
whoever had it could only ever create or delete challenge TXT records — not
repoint an A record or hijack anything that actually resolves.

## Certbot on Debian trixie, credentials kept out of `~/.aws`

Debian 13 enforces PEP 668, so a bare `pip install certbot` is a non-starter.
`pipx` handles that cleanly:

```bash
apt install -y pipx
pipx ensurepath
pipx install certbot
pipx inject certbot certbot-dns-route53
```

I didn't want the IAM credentials sitting in `~/.aws/credentials`, so they live in
a root-only file instead, and a small wrapper script loads them into the
environment before invoking Certbot:

```bash
mkdir -p /etc/letsencrypt/aws
chmod 700 /etc/letsencrypt/aws
chmod 600 /etc/letsencrypt/aws/route53.env   # AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY
```

```bash
#!/bin/bash
set -euo pipefail
ENV_FILE="/etc/letsencrypt/aws/route53.env"
set -a
source "$ENV_FILE"
set +a
/root/.local/bin/certbot "$@"
```

The initial wildcard cert:

```bash
certbot-route53.sh certonly \
  --dns-route53 \
  -d "example.com" \
  -d "*.example.com" \
  --agree-tos \
  -m "admin@example.com"
```

And a cron entry that renews twice daily (only actually renewing within 30 days of
expiry) and reloads Nginx on success:

```
27 3,15 * * * /usr/local/bin/certbot-route53.sh renew --quiet --post-hook "docker exec docker-nginx-1 nginx -s reload"
```

`nginx -s reload` picks up new cert files without dropping active connections,
which matters for anything holding a long-lived sync connection.

## The Docker symlink trap

Nginx runs in `docker-compose`, so the certificate needs to get into that
container. My first pass mounted just the live directory:

```yaml
volumes:
  - /etc/letsencrypt/live/example.com:/etc/letsencrypt/live/example.com:ro
```

This looks reasonable and will bite you the moment Nginx tries to actually read
the files. Everything under `/etc/letsencrypt/live/<domain>/` — `fullchain.pem`,
`privkey.pem` — is a **symlink**, and the targets are relative paths pointing into
`/etc/letsencrypt/archive/<domain>/`. Mount only the `live` directory into a
container and those symlinks point at a path that doesn't exist inside it; Nginx
fails to start with a plain "no such file or directory," which is a confusing error
for what's really a missing bind mount. The fix is to mount the whole
`/etc/letsencrypt` tree so `live` and `archive` are both present and the relative
symlinks resolve correctly:

```yaml
volumes:
  - /etc/letsencrypt:/etc/letsencrypt:ro
```

## Nginx: cert.pem vs. fullchain.pem, and a deprecated directive

My first server block used `cert.pem` instead of `fullchain.pem`. Browsers often
paper over this by fetching missing intermediates themselves, but Android's native
TLS stack does not — it strictly validates the chain it's handed and rejects
anything incomplete with `SSL_ERROR_UNKNOWN_CA_SAME_ISSUER`. `fullchain.pem`
includes Let's Encrypt's intermediate, and that error went away immediately once I
switched to it.

Separately, current Nginx (1.25.1+) deprecated putting `http2` directly on the
`listen` line:

```nginx
# old, produces warnings
listen 443 ssl http2;

# current
listen 443 ssl;
http2 on;
```

Leaving the old form in place across multiple server blocks sharing port 443 also
throws a second warning — `protocol options redefined for 0.0.0.0:443` — because
Nginx binds socket-level options from the first server block it evaluates for that
port, and conflicting `listen ... http2` lines in later blocks look like a
redefinition. Switching every `server{}` block to the standalone `http2 on;` form
cleared both warnings.

## Getting the apps themselves to accept the new domain

With DNS, the cert, and Nginx all sorted, hitting the new URLs returned application-level
errors, not TLS errors — which was actually reassuring, since it meant everything
below the app layer was working:

- **Nextcloud** returned "Access through untrusted domain." Nextcloud keeps its own
  allowlist of hostnames independent of TLS, in `config.php`:

  ```php
  'trusted_domains' => array(
      0 => '192.168.x.x',
      1 => 'nextcloud.example.local',
      2 => 'nextcloud.example.com',
  ),
  ```

- **OpenProject** returned "Invalid host_name configuration," fixed under
  **Administration → System settings → Display** by setting the host name to
  `openproject.example.com` and the protocol to `https`.

Once both apps loaded fine in a browser, the Nextcloud Android app got through
login but hung forever spinning on the final "Grant Access" screen. The cause
turned out to be that Nextcloud didn't know it was sitting behind a reverse proxy
terminating TLS, so it was generating its OAuth-style callback links as `http://`
instead of `https://`, and the phone app was refusing to follow them. Nextcloud's
fix for exactly this is a small set of proxy-awareness settings:

```php
'overwritehost' => 'nextcloud.example.com',
'overwriteprotocol' => 'https',
'overwrite.cli.url' => 'https://nextcloud.example.com',
'trusted_proxies' => array(
    0 => '192.168.x.x',
),
```

`overwriteprotocol` forces every internally generated link — including the grant
callback — to use `https`, and `trusted_proxies` tells Nextcloud it can trust the
`X-Forwarded-Proto` header Nginx is already sending. Force-closing and reopening
the app, the grant screen redirected instantly.

## The OpenProject OAuth rabbit hole

OpenProject's app got further but hit a wall logging in, with a mostly blank
authorization screen showing only the tail end of an error message — "...this
method." — and this in the server log:

```
method=GET path=/oauth/authorize controller=Doorkeeper::AuthorizationsController action=new status=401
```

The full error is OpenProject's standard Doorkeeper OAuth rejection: *"The client
is not authorized to perform this request using this method."* The usual fix is
enabling built-in OAuth applications under **Administration → Authentication →
OAuth applications**. Except there was no such setting to enable — no checkbox, no
default applications listed at all. That absence turned out to be the real clue:
built-in OAuth applications for the mobile app are a feature of OpenProject 17,
and I was running version 15.

Upgrading turned into its own small saga. OpenProject's database migrations only
support moving one major version at a time, so jumping straight to 17 refused to
start. Going to 16 first worked and ran its schema migration — but 16 also wanted
PostgreSQL upgraded from 13 to 17, so that came first: dump, restore into a
PostgreSQL 17 container, confirm it, *then* move OpenProject from 16 to 17. Once
that chain was done, the built-in OAuth toggle appeared, and the app connected on
the first try.

## Wrapping up

None of the individual pieces here were exotic — Route 53, Certbot, BIND9, Nginx,
and two apps' proxy-awareness settings are all well-documented on their own. What
made this a genuine multi-day project was that Android's CA trust model forces
every one of those pieces to actually work end to end before you see anything
other than an error: split-horizon DNS has to route the name correctly, the cert
has to be the full chain, Nginx has to speak modern HTTP/2 without warnings, the
app has to have the new hostname on its allowlist, and it has to know it's
sitting behind HTTPS. Skip any one link and you get a different, unrelated-looking
failure instead of a hint about what's actually wrong.

The result is worth it, though: one wildcard certificate, zero public DNS records
for anything that resolves internally, an IAM key that can only ever create or
delete a TXT record, and both apps working natively on an un-rooted phone without
ever touching Android's certificate settings again.
