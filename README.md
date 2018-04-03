---
title: DNS-Over-TLS Client Setup
date: 2018-04-02 11:22:47
tags:
  - Tech
---

Link: https://blog.kmahyyg.xyz/2018/DNS-Over-TLS/

# Preface

Read this to know what you should know before you skim my article:

- [DNS-Over-TLS vs DNSCrypt](https://tenta.com/blog/post/2017/12/dns-over-tls-vs-dnscrypt)
- [Technical details in RFC7858](https://tools.ietf.org/html/rfc7858)
- [Quad9 DNS](https://www.quad9.net) offered by IBM.
- [Cloudflare DNS](https://1.1.1.1) This DNS is the fastest ever DNS, but working much more unstable in China Mainland. This DNS doesn't provide any EDNS support.
- [DNS-Over-TLS test via Knot-DIG](https://developers.cloudflare.com/1.1.1.1/dns-over-tls/)

# Preparation

1. From your DNS Provider, do a cross-check whether it support DNS-Over-TLS or not.

2. Copy the SSL certificate PIN and hostname (PIN must encoded in SHA256 with Base64 Encoded) to let resolver confirm DNS certificate.You can get it from ```kdig``` result in a reliable network or provider's website.

   For example:

   ```bash
    [kmahyyg@PatrickY  /etc] $ kdig -d @1.1.1.1 +tls-ca +tls-host=cloudflare-dns.com baidu.com
   ;; DEBUG: Querying for owner(baidu.com.), class(1), type(1), server(1.1.1.1), port(853), protocol(TCP)
   ;; DEBUG: TLS, imported 166 system certificates
   ;; DEBUG: TLS, received certificate hierarchy:
   ;; DEBUG:  #1, C=US,ST=CA,L=San Francisco,O=Cloudflare\, Inc.,CN=*.cloudflare-dns.com
   ;; DEBUG:      SHA-256 PIN: yioEpqeR4WtDwE9YxNVnCEkTxIjx6EEIwFSQW+lJsbc=
   ;; DEBUG:  #2, C=US,O=DigiCert Inc,CN=DigiCert ECC Secure Server CA
   ;; DEBUG:      SHA-256 PIN: PZXN3lRAy+8tBKk2Ox6F7jIlnzr2Yzmwqc3JnyfXoCw=
   ;; DEBUG: TLS, skipping certificate PIN check
   ;; DEBUG: TLS, The certificate is trusted. 
   ;; TLS session (TLS1.2)-(ECDHE-ECDSA-SECP256R1)-(AES-256-GCM)
   ;; ->>HEADER<<- opcode: QUERY; status: NOERROR; id: 58591
   ;; Flags: qr rd ra; QUERY: 1; ANSWER: 2; AUTHORITY: 0; ADDITIONAL: 1

   ;; EDNS PSEUDOSECTION:
   ;; Version: 0; flags: ; UDP size: 1536 B; ext-rcode: NOERROR
   ;; PADDING: 394 B

   ;; QUESTION SECTION:
   ;; baidu.com.          		IN	A

   ;; ANSWER SECTION:
   baidu.com.          	466	IN	A	111.13.101.208
   baidu.com.          	466	IN	A	220.181.57.216

   ;; Received 468 B
   ;; Time 2018-04-02 21:48:03 CST
   ;; From 1.1.1.1@853(TCP) in 193.7 ms
   ```

3. According to popularity, we have two methods to use DNS-Over-TLS as a client. More details, see [DNS Privacy](https://dnsprivacy.org/wiki/display/DP/DNS+Privacy+Clients)

   (1) Unbound + Stubby.

   The Stubby is a TCP request forwarder working behind Unbound. Configuration are distributed all the internet.

   Unbound is a famous DNS server.

   (2) Knot Resolver

   Knot Resolver is created by CZ.NIC Labs, which has Knot DNS Server and Knot Dig.

   Knot Resolver is written in C & Golang & Lua, with a highly extensible framework, working as a full-featured DNS relay.

   Today, I'm going to introduce Knot Resolver to you.


# Installation and Deployment

All examples is tested to be working properly under Deepin Linux, which is Debian Sid based.

If you have a Debian-based *nix System, this tutorial should work.

Check whether a software is listening 53,853 ports first.


## (1)Unbound with Stubby: All simple

### Install Unbound first

```bash
# apt update -y && apt install unbound -y
```

### Install Stubby

Cuz Stubby only exists in sid, and its dependencies is lacked in the current repository. So, I recommend you to compile from source directly.

Stubby relies on GetDNS. But the GetDNS in Debian is so old, so you still need to compile it yourself at the very beginning.

### Dependencies and compile

```bash
# apt install build-essential libunbound-dev libidn2-dev libssl-dev libtool m4 autoconf libyaml-dev #Note: libidn2-0-dev maybe used here.
```

#### Compile GetDNS with --with-stubby

Here, we chose to install at /usr/local/getdns

```bash
# git clone https://github.com/getdnsapi/getdns.git
# git submodule update --init
# libtoolize -ci # (use glibtoolize for OS X, libtool is installed as glibtool to avoid name conflict on OS X)
# autoreconf -fi
# mkdir /usr/local/getdns
# ./configure --prefix=/usr/local/getdns --without-libidn --enable-stub-only --with-stubby
# make
# make install
# ldconfig
# ln -s /usr/local/getdns/bin/stubby /usr/bin/stubby
```

After you successfully compiled, try ```sudo stubby -lv``` to check.

### Configuration in Unbound and Stubby

```bash
# unbound-anchor
# touch /var/run/unbound.pid
# touch /var/log/unbound.log && chmod 666 /var/log/unbound.log
```

Refresh root hints first.

Directly Clone the repo listed below and put the configure file in the proper location.

For example:

- Unbound: /etc/unbound
- Stubby: /usr/local/getdns/etc/stubby

```bash
$ git clone https://github.com/kmahyyg/DoT_1111.git
```

#### Performance

So poor. Average time cost is 1.+s ~ 3.+s. It depends on the network environment and how much GFW involved in, cause GFW usually give network data packet which transferred to outside China mainland a TCP_CONN_RESET.

FUCK U, CCP! 

FUCK U, GFW! 

## (2)Knot-Resolver: Simple config with complicated installer

### Compile from source

See [Github](https://github.com/CZ-NIC/knot-resolver) for more details including Docker image.

### Configuration

```bash
root@hostname # vi /etc/knot-resolver/kresd.conf
root@hostname # mkdir /tmp/kresd-cache.mdb
root@hostname # cat /etc/hosts > /etc/hosts.y.kresd
```

```
-- Default empty Knot DNS Resolver configuration in -*- lua -*-

-- Bind ports as privileged user (root) --
net = { '127.0.0.1', '::1' }

-- Unprivileged user --
user('knot-resolver','knot-resolver')

-- Cache related --
cache.size = 50*MB
event.recurrent(60 * minute, function()
        cache.prune()
end)

cache.storage = 'lmdb:///tmp/kresd-cache.mdb'

-- Load modules --
modules = {
    'policy < hints',
    'hints > iterate',
    'daf'
}

-- Add custom hosts file --
hint.add_hosts('/etc/hosts.y.kresd')

-- DNS Firewall --
daf.add 'qname ~ (api|ps|sv|offnavi|newvector|ulog.imap|newloc)(.map|).(baidu|n.shifen).com deny'
daf.add 'qname ~ (.+.|^)(360|so).(cn|com) deny'
daf.add 'qname ~ (.*.||)(dafahao|minghui|dongtaiwang|epochtimes|ntdtv|falundafa|wujieliulan).(org|com|net) deny'
daf.add 'qname ~ (.*.||)(guerrillamail|guerrillamailblock|sharklasers|grr|pokemail|spam4|bccto|chacuo|027168).(info|biz|com|de|net|org|me|la) deny'

-- At this time, no DNS64 support --
-- Now supported: DNSSEC --

-- BEGIN: DNS over TLS --
policy.add(policy.all(policy.TLS_FORWARD({{'1.1.1.1', pin_sha256={'yioEpqeR4WtDwE9YxNVnCEkTxIjx6EEIwFSQW+lJsbc=','PZXN3lRAy+8tBKk2Ox6F7jIlnzr2Yzmwqc3JnyfXoCw='}}})))
-- END: DNS over TLS --
```

#### Test for performance

Unable to do that. Strongly recommend you **NOT** to use this method because this package doesn't have any modules built-in except ```iterator,cache,validator``` , you'll need to compile all modules inneed by yourself. I didn't go through this method.

# Conclusion



NOT RECOMMEND TO USE THIS AS A PRIMARY DNS METHOD BECAUSE OF POOR CONNECTIVITY IN CHINA.

# More questions

Feel free to contact me via comments at the bottom of the text or use TG to get in touch with me.
