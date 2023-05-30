+++
title = "Blocking the new .zip TLD on Fedora"
date = "2023-05-20T22:53:14+02:00"
author = "nleanba"
tags = ["linux"]
keywords = []
description = "An attempt to block the new TLD"
showFullContent = false
readingTime = true
hideComments = false
color = "" #color from the theme settings
draft = false
math = false
+++

# What:

Because it seems like it might provide some (idk) security benefits, and because
it seemed like an intersting exercise, I wanted to figure out how to block any
requests to a .zip url from my laptop.

# How:

Trying to do so using the bind-DNS server

1. Installing bind:

    ```shell
    dnf install bind
    ```

2. Updating/Creating various Config files:

    {{< code language="c" title="/etc/named.conf" lang="updated" line-numbers="true" line="44-48" >}}
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

	recursion yes;

	dnssec-validation yes;
	managed-keys-directory "/var/named/dynamic";
	geoip-directory "/usr/share/GeoIP";

	pid-file "/run/named/named.pid";
	session-keyfile "/run/named/session.key";

	/* https://fedoraproject.org/wiki/Changes/CryptoPolicy */
	include "/etc/crypto-policies/back-ends/bind.config";

	/* nope -- pointless */
	// response-policy { zone "zip"; };
};

logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

zone "zip" IN {
	type master;
	file "zip-rpz";
	allow-update { none; };
};

zone "." IN {
	type hint;
	file "named.ca";
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";

{{< /code >}}

    {{< code language="dns-zone" title="/var/named/zip-rpz" lang="added" line-numbers="true" >}}
$TTL 1D ; default expiration time of all RRs without their own TTL value
@       IN  SOA     ns.zip. postmaster.ns.zip. ( 2020091025 7200 3600 1209600 3600 )
@       IN  NS      ns                     ; nameserver
*       IN  A       127.0.0.1              ; localhost
        IN  AAAA    ::                     ; localhost
{{< /code >}}

3. Apply temporarily

    ```shell
    sudo systemctl enable named
    sudo service named restart
    resolvectl dns wlp0s20f3 127.0.0.1
    ```

    _Note: this applies it_ very _temporarily (ca 2 mins, idk why)_

    Various other commands, some useful:

    ```shell
    journalctl -xeu named.service

    dig url.zip
    dig example.com

    # ??
    sudo firewall-cmd --add-service=dns --perm
    sudo firewall-cmd --reload

    # ??
    sudo chgrp named -R /var/named
    sudo chown -v root:named /etc/named.conf
    sudo restorecon -rv /var/named
    sudo restorecon /etc/named.conf
    ```


4. Apply persistently

    {{< code language="toml" title="/etc/systemd/resolved.conf" lang="updated" line-numbers="true" line="8" start="5" >}}
# ...
[Resolve]
# ...
DNS=127.0.0.1
# ...{{< /code >}}

    After rebooting, it now “fails” to resolve any .zip url.\
   They are redirected to `127.0.0.1` (or `::`) where nobody is listening...

5. ~~Profit~~
