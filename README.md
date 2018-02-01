# Docker web proxy help

Docker users frequently have a reverse proxy (nginx, haproxy, apache,
etc) listen for incoming requests on ports 80 and 443, and the
dispatch them to various workers.

This collection helps streamline this process.  It uses Apache for
both the reverse proxy and the proxy client, and takes the annoying
parts out of setting this up.  It features optional full integration
with letsencrypt for free and easy SSL/TLS certificates.

# Feature List

 - Based on my
   [Debian Apache base](https://github.com/jgoerzen/docker-debian-base),
   inheriting its features:
   - automated security patches for the OS, openssl, and Apache
   - Real init with zombie process reaping
   - Clean shutdown support
   - See the above URL for details.
 - Support for automating the process of requesting and updating your
   SSL certificates from letsencrypt, making the process completely
   transparent and automatic - should you wish to use it.
 - Low memory requirements and efficient.
 - Based on Apache, so it's what you (probably) already know.

# Assumptions

You have set up a Docker network of some sort that these systems can
use.  One easy way is to use `docker net create proxynet` and then
make sure to say `--net=proxynet` and set a `--name` on your calls to
`docker run`.

# The proxied application

Let's talk about the proxied application first.  This is where you run
your web applications -- blogs, wikis, whatever.

## Use

To act as a proper proxied application, your Dockerfile can start with
`FROM jgoerzen/proxied-app-apache`.  Then, you only need to do two
things:

First, drop a file in `/etc/apache2/sites-available` with a
`<VirtualHost *:80>` line.  It should include an
`Include sites-avaialable/common-sites` line to bring in needed
configuration.  Don't forget to call `RUN a2ensite sitename` in your
Dockerfile for this.  (Of course, you can add as many of these files
as you like.)

Secondly, you need to define what IPs to authorize as your reverse
proxy.  You can do this by either setting the `PROXYCLIENT_AUTHORIZED`
environment variable to a single IP address or address plus netmask,
or replacing the file `/etc/apache2/authorized-proxies.txt` with one
or more such entries, one per line.  These are sent to the Apache
[RemoteIPInternalProxyList](https://httpd.apache.org/docs/2.4/mod/mod_remoteip.html#remoteipinternalproxylist)
directive.  If you are using Docker's default networking, and wish to
authorize *any* internal host as your source, a common way would be
`172.16.0.0/12`.  However, it would be more secure to put your systems
on a separate Docker network and only authorize it.  Even better, give
your reverse proxy an `--ip` and authorize only that.

## Internal details

There are a couple of interesting issues here.  First, the IP address
that the request appears to come from is going to be the IP of the
reverse proxy or load balancer, not the IP of the browser.  This can
mess with logging, security, etc.  This uses the `X-Forwarded-For` and
`X-Forwarded-Proto` headers to propagate the proper remote IP, and set
the HTTPS variable if relevant.  This lets most web programs properly
understand what the real remote is, and whether they used SSL to
access the site.  Note that while you could proxy port 443 over to
your proxied application with these scripts, this setup assumes that
you terminate SSL at the proxy and use basic HTTP on over to the
client.

The reverse-proxy-apache setup included here will set both of these
headers appropriately.



# Copyright

Docker scripts, etc. are
Copyright (c) 2018 John Goerzen
All rights reserved.

Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions
are met:
1. Redistributions of source code must retain the above copyright
   notice, this list of conditions and the following disclaimer.
2. Redistributions in binary form must reproduce the above copyright
   notice, this list of conditions and the following disclaimer in the
   documentation and/or other materials provided with the distribution.
3. Neither the name of the University nor the names of its contributors
   may be used to endorse or promote products derived from this software
   without specific prior written permission.

THIS SOFTWARE IS PROVIDED BY THE AUTHORS AND CONTRIBUTORS ``AS IS'' AND
ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
ARE DISCLAIMED.  IN NO EVENT SHALL THE REGENTS OR CONTRIBUTORS BE LIABLE
FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS
OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION)
HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT
LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY
OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF
SUCH DAMAGE.

Additional software copyrights as noted.

