# DNS ACME Challenges with BIND 9

For my homelab I always used [Caddy](https://caddyserver.com/) as my reverse Proxy and because its not directly accessible from the internet, I decided to use ACME DNS Challenges with [Cloudflare](https://www.cloudflare.com) for my [Let's Encrypt](https://letsencrypt.org) certificates.
But since lately, I decided to try to rely less on big tech companies and do more stuff on my own.
I tried a couple things like for example [acme-dns](https://github.com/joohoi/acme-dns), but it always felt a little strange to me that all those solution invent new APIs for this and everybody has to implement them for Cloudflare, acme-dns and all the other APIs out there.
At that point I found out that there is [RFC2136](https://datatracker.ietf.org/doc/html/rfc2136) which describes dynamic DNS Updates directly via an DNS Request and also found out that there is an [Caddy Plugin](https://github.com/caddy-dns/rfc2136) for this.

So I decided to say goodbye to Cloudflare and manage my own DNS server and use dynamic DNS Update for certificates.
This setup, I want to describe here.
Maybe you also want to use less big tech.

## Prerequisites

First of all, since others want to connect to your DNS server or at least Let's Encrypt wants to for the challenges, you need to have a server which is accessable from the Internet directly via IP.
I decided to rent a pretty small and cheap VM for this by [netcup](https://www.netcup.com) and installed [Debian](https://www.debian.org) Trixie, but you do you.

Also I decided to use [BIND 9](https://www.isc.org/bind/) as my DNS server.
As far as I am aware, there are multiple different DNS server which support dynamic DNS Updates, like [Knot DNS](https://www.knot-dns.cz/) or [PowerDNS](https://www.powerdns.com/), but honestly, the main reason for me to use BIND 9 was that I already know about it and did not really care after the setup was done.

Also I needed to compile Caddy on my own which is pretty easy with [xcaddy](https://github.com/caddyserver/xcaddy).
Since I wanted to deploy Caddy in a Container anyway, I just created my own image with, like this:

```Dockerfile
FROM caddy:builder AS builder

RUN xcaddy build \
    --with github.com/caddy-dns/rfc2136

FROM caddy:latest
COPY --from=builder /usr/bin/caddy /usr/bin/caddy
```

## Configuring BIND 9

Now we need to configure our own DNS server.
I will do this as briefly as possible and not bother you with details of my own BIND 9 configuration like DNSSec or IPv6.
Also I will do this exemplary with my own domain `haardiek.org`.

For the overview, there are two relevant directories for BIND 9 in Debian Trixie `/etc/bind` for static configuration and `/var/lib/bind` for dynamic files.

First of all we need a way to make sure that we are the only ones able to update DNS entries in our zone, this is done by cryptographically signing the DNS messages we are going to send for updates.
For this we need a [TSIG Key](https://bind9.readthedocs.io/en/v9.18.2/advanced.html#tsig), which can be generated and stored like this:

```bash
tsig-keygen haardiek.org > /etc/bind/key.haardiek.org
```

After that, we can add our generated key and our zone to `/etc/bind/named.conf.local`, which is meant to be for your own configuration in Debian.
It should look something like this:

```
# /etc/bind/named.conf.local

# include the update key
include "/etc/bind/key.haardiek.org";

# define the zone
zone "haardiek.org" {
    type master;
    # dynamic zone file
    file "/var/lib/bind/db.haardiek.org";
    allow-transfer { none; };
    # allow updates with the key
    allow-update { key "haardiek.org"; };
};
```

The most complicated part for me was now, that I wanted to have static DNS entries for all of my server I can manage via a configuration file (I deploy them with [Ansible](https://www.ansible.com/)) and also dynamic entries managed from Caddy for the DNS challenges in the same zone `haardiek.org`.
This is something which is not supported by BIND 9 DNS.
With dynamic updates, BIND 9 stores all changes in a journal files `/var/lib/bind/*.jnl` and plays them back to the zones files.
If you change the zone file manually, the journal does not fit anymore and an error will occur, when reloading BIND 9.
You can work around this by with the included tooling, see [here](https://bind9.readthedocs.io/en/v9.18.2/advanced.html#the-journal-file), but this also means to block new dynamic updates for some times and seems clumsy for me to implement in any automation.

The easiest way for me to work around this, was to not handle my own DNS entries in a zone file, but also use the dynamic update mechanism, I already provide for Caddy.
There are some amazing tools out there to do this like [octoDNS](https://github.com/octodns/octodns) where you can define all our DNS entries via YAML and sync it idempotently to your zone and maybe in the future I will do that, but currently I do not have that many entries and decided to simply use [nsupdate](https://linux.die.net/man/8/nsupdate).
`nsupdate` lets you submit your dynamic DNS requests directly from a small file, but it is not as comfortable, because we also need to handle the idempotency ourselves.

So I created a small zone file `/var/lib/bind/db.haardiek.org` as a starting point for the zone with:

```
$TTL 300
@ IN SOA ns1.haardiek.org. admin.haardiek.org. (
        1          ; Serial
        3600       ; Refresh
        1800       ; Retry
        604800     ; Expire
        300        ; Minimum TTL
)

;; NS Records
@        IN NS ns1.haardiek.org.

;; A Records
ns1      IN A 188.68.52.102

;; AAAA Records
ns1      IN AAAA 2a03:4000:6:ecfd:74d8:fdff:feea:c71c
```

And created my my small `nsupdate` script  `/etc/bind/nsupdate.haardiek.org` (I shortened it a bit for this):

```bash
server 127.0.0.1
zone haardiek.org

update delete ns1.haardiek.org A
update add ns1.haardiek.org 300 A 188.68.52.102

update delete ns1.haardiek.org AAAA
update add ns1.haardiek.org 300 AAAA 2a03:4000:6:ecfd:74d8:fdff:feea:c71c

update delete blog.haardiek.org CNAME
update add blog.haardiek.org 300 CNAME shaardie.github.io.

update delete haardiek.org MX
update add haardiek.org 300 MX 10 mxext1.mailbox.org.
update add haardiek.org 300 MX 10 mxext2.mailbox.org.
update add haardiek.org 300 MX 20 mxext3.mailbox.org.

show

send
```

And after reloading the server with `systemctl reload bind9` to make BIND 9 aware of the new zone, I was able to add my own static entries with:

```bash
nsupdate -k "/etc/bind/key.haardiek.org" "/etc/bind/nsupdate.haardiek.org"
```

As said, I do all this with Ansible, so the regular steps for installing or updating for me are:

* Copying the static configuration to `/etc/bind/`.
* Initializing the dynamic zone files in `/var/lib/bind`, if they are not already present.
* Reloading BIND 9, if something changed to make it aware of new zones.
* Executing the `nsupdate` script, if some static entries changed.

And that's it.

## Configuring Caddy

The Caddy configuration is even more simple.

Like described in [The RFC2136 DNS Provider Plugin](https://github.com/caddy-dns/rfc2136), I created a configuration file with the information about the dns server and the TSIG key in `/etc/caddy/nsupdate.caddy`:


```
{
    acme_dns rfc2136 {
        key_name "haardiek.org"
        key_alg "hmac-sha256"
        key "XXX"
        server "ns1.haardiek.org:53"
    }
}
```

and referenced it in my `Caddyfile` with `import /etc/caddy/nsupdate.caddy`.
After that DNS Challenges against my own server were automatically used if no other configuration was present.

So creating a reverse proxy entry with proper TLS is now as easy as:

```
kuma.internal.haardiek.org {
  reverse_proxy kuma-uptime-kuma-1:3001
}
```

## Conclusion

So that's my setup and I am pretty happy about because it reduces my reliance on big tech companies at least a bit.

Setting is up was a bit fiddly at the beginning and the big DNS servers are not the most user friendly, but I think this solution should be quite solid for now.
