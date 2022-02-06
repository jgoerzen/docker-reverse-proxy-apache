# Docker web proxy help

**NOTE: This package has moved from Github.  See its [new home on Salsa](https://salsa.debian.org/jgoerzen/docker-apache-proxy).**

Part of the [docker-apache-proxy](https://salsa.debian.org/jgoerzen/docker-apache-proxy) collection.

Docker users frequently have a reverse proxy (nginx, haproxy, apache,
etc) listen for incoming requests on ports 80 and 443, and the
dispatch them to various workers.

This collection helps streamline this process.  It uses Apache for
both the reverse proxy and the proxy client, and takes the annoying
parts out of setting this up.  It features optional full integration
with letsencrypt for free and easy SSL/TLS certificates.

# Feature List

 - Based on my
   [Debian Apache base](https://salsa.debian.org/jgoerzen/docker-debian-base),
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

# The proxied application (jgoerzen/proxied-app-apache)

Let's talk about the proxied application first.  This is where you run
your web applications -- blogs, wikis, whatever.  This is a base image
for you to build upon.

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

Finally, make sure to end your Dockerfile with `CMD ["/usr/local/bin/boot-debian-base"]`.

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

It is not necessary to expose ports using `-p` or `-P` from this
container, since the reverse proxy server does so.

Please see the comments below under Recommended Volumes.

# Reverse proxy server (jgoerzen/reverse-proxy-apache)

This server receives connections and dispatches them to your other
Docker containers.  It also is fully integrated with the letsencrypt
project, automatically requesting and renewing your SSL certificates
if you'd like.

You can build upon this image, but it should need very little
tweaking.

There are three core things that you can do with this image as-is:

 1. Proxy (almost) all requests to a site to the Docker container
    hosting it.  "Almost" because letsencrypt verification requests
    are intercepted and handled here.
 2. Perform simple redirects (eg, example.com -> www.example.com)
 3. Proxy letsencrypt (ACME) requests ONLY (for when you are running
    letsencrypt in your target container)

## Site setup

In your Dockerfile, you'll use RUN to call `docker-setupsites` to
provision configurations.  I'll cover each of the above three cases
here.

### Site setup, case 1: proxying requests to a Docker container

There are three `docker-setupsites` subcommands here:
`proxysite_ssl`, `proxysite_nossl`, and `proxysite_both`.  They all
take the same parameters and differ only in how they handle SSL.  The
first parameter is the target (which should almost always be port 80,
since we do SSL termination here), the second is the name of the
configuration, and the third and following are a list of one or more
domains.  Examples:

    docker-setupsites proxysite_both "wordpress.proxynet:80" wordpress-sites  \
          blog.example.com news.example.com
    
    docker-setupsites proxysite_nossl "mainweb.proxynet:80" mainweb \
          www.example.com

This will cause configurations to be created for blog.example.com and
news.example.com, in both SSL and non-SSL versions, directing traffic
to wordpress.proxynet:80 (saved, incidentally, in configuration files
named wordpress-sites.80.conf and wordpress-sites.443.conf).  If
letsencrypt generation is used, SSL certificates for blog.example.com
and news.example.com will be automatically handled.  Also,
www.example.com will have its traffic sent to mainweb.proxynet:80.

### Site setup, case 2: redirect sites

This is very similar - the subcommands are `redirectside_ssl`,
`redirectsite_nossl`, and `redirectsite_both`.  They take exactly two
parameters - the source for redirection as a site, and a target URL,
neither of which should have a trailing slash.

For instance:

    docker-setupsites redirectsite_both example.com https://www.example.com
    docker-setupsites redirectsite_nossl happenings.example.com http://news.example.com
    docker-setupsites redirectsite_ssl happenings.example.com https://news.example.com
   
In this case, a request for either `http://example.com` or
`https://example.com` will be sent to `https://www.example.com`.  Note
that this will tend to push people to SSL.  Becuase we redirect an
entire site, `http://example.com/linux` will be sent to
`https://www.example.com/linux` as well.

When you use `redirectsite_ssl` or `redirectsite_both`, your target
should always be an `https` URL, so you can avoid the user getting
warnings about an insecure redirect.  Sometimes you do not wish to
push people into SSL.  The second and third lines in the example above
demonstrate that situation, where the non-SSL and SSL redirects go to
`http` or `https` URLs, respectively.

### Site setup, case 3: ACME redirects

Sometimes, you need only to proxy ACME to a destination.  Perhaps, for
instance, you're running an IMAP server and have a local certbot
there.  With all of the instances in cases 1 and 2, ACME verification
requests are intercepted and handled locally.  This inverts the sense;
ONLY ACME verification requests are sent.

Here's an example:

    docker-setupsites proxy_acme "imap.proxynet:81" \
         imap.example.com smtp.example.com
    
In this case, inbound requests on port 80 (these are always non-SSL
requests) for imap.example.com and smtp.example.com will be sent to
port 81 on imap.proxynet, where you have presumably set up certbot to
listen.

## Letsencrypt handling

By default, letsencrypt handling is not enabled.  If you wish to
handle SSL on your own, you will need to `a2enmod ssl` and make some
modifications to the SSL config files.  However, if you want
letsencrypt to handle it, do *NOT* `a2enmod ssl` but rather set the
`LETSENCRYPT_EMAIL` environment variable to your container.

When `LETSENCRYPT_EMAIL` is set, then When your container first
starts, a pre-init script will do this:

 - First, it will start Apache on the non-SSL ports only.  (The SSL
   configurations generated by docker-setupsites are all wrapped in
   `IfModule` for SSL, and SSL isn't enabled yet, which is good,
   because we don't have a valid configuration yet.)  This is to
   answer the certbot validation requests.
 - Then, it sends off an automated certbot request and waits for the
   answers.
 - It lets certbot install its certs and enable SSL in Apache as
   appropriate.
 - The pre-init script then deletes itself and proceeds with the boot.
 
A cron job in the container will handle updates and revalidation of
your certs.

## Final note

Finally, make sure to end your Dockerfile with `CMD ["/usr/local/bin/boot-debian-base"]`.


# Recommended Parameters - Running Container

I recommend you to run your containers with (older systemd, cgroupd v1):

    docker run -td --stop-signal=SIGRTMIN+3 \
      --tmpfs /run:size=100M --tmpfs /run/lock:size=100M \
      -v /sys/fs/cgroup:/sys/fs/cgroup:ro \
      --name=name -t -d --net=whatever
      
Or with a newer systemd, as in Debian bullseye on the host:

    docker run -td --stop-signal=SIGRTMIN+3 \
      --tmpfs /run:size=100M --tmpfs /run/lock:size=100M \
      -v /sys/fs/cgroup:/sys/fs/cgroup:rw --cgroupns=host \
      --name=name --net=whatever

# Recommended Volumes

I recommend that you add `VOLUME ["/var/log/apache2"]` to your
Dockerfile for both containers, and `VOLUME ["/etc/letsencrypt"]` to
your reverse proxy container.  When rebuilding and restarting your
containers, use a sequence such as:

    docker stop web
    docker rename web web.old
    docker run <<parameters>> --volumes-from=web.old  --name-web ....
    docker rm web.old
   
This will let your logs persist, and will avoid unnecessary calls to
letsencrypt to obtain new certs.  The latter is important to avoid
false expiration emails and hitting their rate limiting.

# Copyright

Docker scripts, etc. are
Copyright (c) 2018-2022 John Goerzen
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

# See Also

 - [Salsa page](https://salsa.debian.org/jgoerzen/docker-apache-proxy)
 - Docker hub packages:
   [jgoerzen/proxied-app-apache](https://hub.docker.com/r/jgoerzen/proxied-app-apache/),
   [jgoerzen/reverse-proxy-apache](https://hub.docker.com/r/jgoerzen/reverse-proxy-apache/)
