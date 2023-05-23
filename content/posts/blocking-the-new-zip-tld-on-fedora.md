+++
title = "Blocking the new .zip TLD on Fedora"
date = "2023-05-20T19:12:32+02:00"
author = "nleanba"
tags = ["linux"]
description = "An attempt to block the new TLD"
showFullContent = false
readingTime = true
hideComments = false
color = "" #color from the theme settings
draft = false
math = false
+++

# What:

Because it seems like it might provide some (idk) security benefits, and because it seemed like an intersting exercise, I wanted to figure out how to block any requests to a .zip url from my laptop.

# How:

This is the hard part that I’m not sure I fully figured out...

This is documenting what I did so far:

1. Installing bind9:

    ```shell
    dnf install bind
    ```

2. Updating/Creating various Config files:

    {{< code language="shell" lang="FILE" title="/etc/hosts" line-numbers="true" line="9-11" start="8" >}}
# ...
# Does not work
127.0.0.1  *.zip
::1        *.zip{{< /code >}}
    {{< code language="shell" lang="FILE" title="/etc/bindresvport.blacklist" line-numbers="true" line="12" >}}
#
# This file contains a list of port numbers between 600 and 1024,
# which should not be used by bindresvport. bindresvport is mostly
# called by RPC services. This mostly solves the problem, that a
# RPC service uses a well known port of another service.
#
623     # ASF, used by IPMI on some cards
631     # cups
636     # ldaps
664     # Secure ASF, used by IPMI on some cards
749     # Kerberos V kadmin
774     # rpasswd
873     # rsyncd
921     # lwresd
992     # SSL-enabled telnet
993     # imaps
994     # irc
995     # pops
    {{< /code >}}
    {{< code language="c" title="/etc/named.conf" >}}
//
// named.conf
//
// Provided by Red Hat bind package to configure the ISC BIND named(8) DNS
// server as a caching only nameserver (as a localhost DNS resolver only).
//
// See /usr/share/doc/bind*/sample/ for example named configuration files.
//

options {
    listen-on port 53 { 127.0.0.1; };
    listen-on-v6 port 53 { ::1; };
    directory 	"/var/named";
    dump-file 	"/var/named/data/cache_dump.db";
    statistics-file "/var/named/data/named_stats.txt";
    memstatistics-file "/var/named/data/named_mem_stats.txt";
    secroots-file	"/var/named/data/named.secroots";
    recursing-file	"/var/named/data/named.recursing";
    allow-query     { localhost; };

    /* 
    - If you are building an AUTHORITATIVE DNS server, do NOT enable recursion.
    - If you are building a RECURSIVE (caching) DNS server, you need to enable 
    recursion. 
    - If your recursive DNS server has a public IP address, you MUST enable access 
    control to limit queries to your legitimate users. Failing to do so will
    cause your server to become part of large scale DNS amplification 
    attacks. Implementing BCP38 within your network would greatly
    reduce such attack surface 
    */
    recursion yes;

    dnssec-validation yes;

    managed-keys-directory "/var/named/dynamic";
    geoip-directory "/usr/share/GeoIP";

    pid-file "/run/named/named.pid";
    session-keyfile "/run/named/session.key";

    /* https://fedoraproject.org/wiki/Changes/CryptoPolicy */
    include "/etc/crypto-policies/back-ends/bind.config";

    /* (noam) attempt at rpz blocking of .zip TLD */
    response-policy { zone "zip"; };
};

logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

zone "." IN {
    type hint;
    file "named.ca";
};

zone "zip" {
    type master;
    file "zip-rpz";
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
    {{< /code >}}
    {{< code language="dns-zone" title="/var/named/named.localhost" >}}
$TTL 1D
@	IN SOA	@ rname.invalid. (
        0	; serial
        1D	; refresh
        1H	; retry
        1W	; expire
        3H )	; minimum
NS	@
A	127.0.0.1
AAAA	::1
    {{< /code >}}
    {{< code language="dns-zone" title="/var/named/zip-rpz" >}}
; zone file for rpz blocking *.zip
$TTL    604800
@        SOA nonexistent.nodomain.none. dummy.nodomain.none. 1 12h 15m 3w 2h
        NS  nonexistant.nodomain.none.


zip CNAME .
veryspecific.example.com CNAME .
    {{< /code >}}

3. Applying changes

    ```shell
    sudo service bind9 restart
    sudo service bind restart
    systemctl status named.service
    journalctl -xeu named.service

    nslookup zip
    nslookup url.zip

    sudo service network restart
    sudo service named restart
    ```
4. Cry because it has had no apparent effect
5. Reboot