---
title: "Self-hosting DNS for no fun, but a little profit!"
date: "2025-11-18T23:00:00+02:00"
tags:
- english
- linux
- dns
---

After [Gandi](https://montefiore.eu/en/montefiore-investment-announces-the-sale-of-its-stake-in-gandi-to-total-webhosting-solutions/) was bought up and started taking extortion level prices for their
domains I've been looking for an excuse to migrate registrar. Last week I
decided to bite the bullet and move to [Porkbun](https://porkbun.com/) as I have another domain renewal
coming up. However after setting up an account and paying for the transfer for
4 domains, I realized their DNS services are provided by [Cloudflare](https://www.cloudflare.com/)!

I personally do not use Cloudflare, and stay far away from all of their products
for various reasons. And with this weeks outage I was quite happy I stick with
that decision 😅.

I was planning on writing up a bit of the things I've learned while working on a
setup up to self-host authoritative DNS servers for my domains, now seems like a
good time! I hope it gives people a bit of motivation to self-host DNS.

The intention here is not to list all available options, but list the decisions
I made. The goal here is not to self-host a complete redundant DNS server setup.
I personally don't have time for that, but I would like to not be tied to the
DNS services of the registrar I'm using, and also have agency over how my
domains are being run.

# DNS servers

DNS servers consist of effectively two parts. One primary server, and preferably
multiple secondary servers. The job of the primary server is to tell the
secondary servers about your records, and which gives you redundancy to ensure
there is always something serving your DNS records.

The goal is to host our own primary DNS server, and then we can either self-host
secondary servers or delegate this job to one of more free solutions. This
simplifies the setup on our end as we don't have to self-host a redundant
network of DNS server, but still retain agency over our domain records.

There are multiple ways to structure your DNS servers, the one I think makes the
most sense for self hosting is what is called a "hidden master". This means that
the primary DNS server is never part of your announced nameservers with your
domains, but they are never publicly announced and only notifies secondary
servers about new records. The secondary servers are the ones we keep as
published nameserver records.

It's practical to not have people send DNS requests into my small cluster and
home network. With a "hidden master" setup we only need to care about the
secondary servers getting updates, and if our home server setup disappears for
an hour we do not have any visible downtime.

If you don't have the ability to host something that is publicly reachable
through your local hackerspace, or other means. You could look at hosting this
on a cheap VPS somewhere. [Oracle Cloud](https://signup.oraclecloud.com/) offers a free tier VPS that I think
should be perfectly capable of hosting a primary DNS server.

If you would rather not do the primary server yourself, you could look into
[servfail.network](https://beta.servfail.network/) which is a small community
run authoritative DNS provider currently getting funding from NLNet.

For hosting secondaries we have two options. We could host one ourself, or we
could use several free options for hosting secondaries. I think utilizing free
secondary DNS servers makes sense! They are free after all.

A couple of options:

[ns-global.zone](https://ns-global.zone/) is a free DNS secondary anycast
network. You just host a primary server somewhere, sign-up to this service and
they will provide you with free secondaries!

[Hurricane Electric](https://dns.he.net/) is an old ISP that has offered several
free services for decades. They allow you to use them as secondary DNS servers.

[Hetzner](https://www.hetzner.com/dns/) is a fairly well known server provider that offers free secondary servers.

Using a combination of these three, in whatever you think is a fun combination,
gives us quite a bit of diversity to easily self-host a redundant DNS setup for
our domains.

# knot-dns 

I plan on writing a longer post about my personal home server setup, but it is
essentially an Incus cluster with 3 nodes. 2 NUC-sized computers at home, and a
tiny VPS at my local hackerspace that works as a proxy for my local services I
want to publicly expose. This is done through [Wireguard](https://www.wireguard.com/) tunnels.

For my primary DNS server I decided to use [knot-dns](https://www.knot-dns.cz/)
which is a fairly well known server project. It sits reachable and publicly exposed 
to serve the secondary servers. I host it using the [Docker OCI image](https://hub.docker.com/r/cznic/knot). Which is 
hosted using the OCI support in incus. Terraform config is [here](https://github.com/Foxboron/ansible/blob/master/terraform/incus/modules/dns/main.tf).

Here is a (slightly edited and small) copy of my current knot config file that serves `bloat.dev`.

```
server:
    automatic-acl: on
    listen: [ 0.0.0.0@53, ::@53 ]

log:
  - target: syslog
    any: info

key:
  # TSIG key to authenticate zone transfers
  - id: hetzner-key
    algorithm: hmac-sha256
    secret: <SNIP>

remote:
  # https://ns-global.zone/
  - id: ns-global
    address: [204.87.183.53, 2607:7c80:54:6::53]

  # Hetzner secondary
  - id: hetzner
    address: [213.239.242.238, 2a01:4f8:0:a101::a:1, 213.133.100.103, 2a01:4f8:0:1::5ddc:2, 193.47.99.3, 2001:67c:192c::add:a3]
    key: hetzner-key

template:
  # Default options for our zones
  - id: default
    storage: /config/zones
    notify: [hetzner, ns-global]

zone:
  - domain: bloat.dev
    notify: nameservers
```

I don't have time to explain zone files, the post would be too long, but I edit 
the above configuration and my zone files locally under `/srv/hackeriet.linderud.dev/coredns01` 
and use [`syncthing`](https://syncthing.net/) to synchronize the files.
I really like having files locally to edit for my servers.

The result of this is that `bloat.dev` has two secondary DNS servers replying to DNS records I announce!

```
λ ~ » dig ns bloat.dev +short
ns1.first-ns.de.
ns-global.kjsl.com.
```

Moving all my important domains to this setup is a work in progress, I have
plans to at least self-host one public secondary. But I hope laying all of this
out here gives people some inspiration to try and self-host more things.
