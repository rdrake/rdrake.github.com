---
title:  RYO Netflix DNS Proxy
layout:  post
---
## Introduction ##

Netflix makes use of IP geolocation in order to determine what, if any, content to serve to the user.  If you wish to access US-only Netflix content, you must have a US IP.  You no longer require a US Netflix account, displayed content is determined based on your IP only.

The easiest way to accomplish this would be with a non-transparent proxy.  With this method it will appear as though you are accessing Netflix from inside of the US.  Of course you need a US IP in order to do this.

This introduces a problem:  all traffic is routed through the proxy.  In order to avoid this, we must make use of DNS to route select domains through our proxy and leave others untouched.

My OS of choice is [FreeBSD](http://www.freebsd.org/), but other unix-like OSes should work just fine.

## Attempt 1:  SOCKS Proxy ##

My first attempt utilized a SOCKS proxy.  I chose [Nylon](http://monkey.org/~marius/pages/?page=nylon) as it was simple to setup and had a low memory footprint.

It's available in the ports tree.  You can either `pkg_add -r nylons` or build from ports.

    # cd /usr/ports/net/nylon
    # make install clean

Next I modified the `nylon.conf` file to allow my IP to access the SOCKS proxy.

    [General]
    [Server]
    Allow-IP=127.0.0.1/32 w.x.y.z

The defaults for the rest of the options were fine for my needs.  Next enable Nylon and start it up.

    # echo 'nylon_enable="YES"' >> /etc/rc.conf
    # /usr/local/etc/rc.d/nylon start

If all goes well you should see Nylon running

    # ps -A | grep nylon

You should now be able to set your browser's SOCKS proxy to Nylon's IP and port 1080.  If you visit netflix.com, you should be greeted with US-only content.

### Negotiation Failed ###

If you cannot connect to the proxy via your browser, run Nylon in foreground mode.

    nylon -c /usr/local/etc/nylon.conf -f

If you get an error message `negotiation failed`, ensure your browser is set to use a SOCKS proxy and *not* an HTTP proxy.

### Selective Proxying ###

A problem with this method is that all traffic will go through the proxy.  There are two ways to avoid this.

1.  Use a browser extension [FoxyProxy](http://getfoxyproxy.org/) or similar.
2.  Make use of a [PAC](http://en.wikipedia.org/wiki/Proxy_auto-config) file.

I used a PAC file.  Simply put, a PAC file tells your browser which sites to send through a proxy and which to leave untouched.  Here's the one I used.

    function FindProxyForURL(url, host) {
        if (shExpMatch(url, "http://*netflix.com*"))
            return "SOCKS w.x.y.z:port";
	
        return "DIRECT";
    }

For Firefox, open up preferences and go to **Advanced**, **Network**, and click on **Settings...** under **Connection**.  Point the automatic proxy configuration URL to your PAC file.

For OSX, you can do the same in **System Preferences** under **Network**, **Advanced...**, **Proxies**.

Other browsers should be similar.

**Note:**  You can also use SSH tunnels to create a SOCKS proxy instead of running Nylon.

    $ ssh -D <port> w.x.y.z

## Attempt 2:  Nginx &amp; Dnsmasq ##

While the first solution worked, it was far from ideal.  It required configuring each computer.  My second attempt involved using [Nginx](http://nginx.org/) as a reverse proxy and [Dnsmasq](http://www.thekelleys.org.uk/dnsmasq/doc.html) to redirect select domains through Nginx.

I close them for their simplicity and because I already run my site on Nginx.

Netflix makes use of the following three domains to determine which location you are in and whether or not you are able to play the selected content:

 * `movies.netflix.com`
 * `moviecontrol.netflix.com`
 * `cbp-us.nccp.netflix.com`

### Configuring Nginx ###

Nginx is simple to configure for the first two domains, but more difficult for the last one.  The first two use simple HTTP while the last one makes use of SSL.

If you don't already have Nginx installed, it's available via `pkg_add -r nginx` or by the ports collection.

    # cd /usr/ports/www/nginx
    # make install clean

Now add the first two domains to your configuration.

    server {
        listen 80;
        server_name movies.netflix.com;

        location / {
            proxy_pass http://movies.netflix.com:80;
        }
    }

    server {
        listen 80;
        server_name moviecontrol.netflix.com;

        location / {
            proxy_pass http://moviecontrol.netflix.com:80;
        }
    }

The last one requires a certificate.

    # cd /usr/local/etc/nginx/
    # openssl req -new -x509 -nodes -out server.crt -keyout server.key
    # chmod 600 server.key

And a different configuration block.

    server {
        listen 443;
        ssl on;
        ssl_certificate server.crt;
        ssl_certificate_key server.key;

        server_name cbp-us.nccp.netflix.com;

        location / {
            proxy_pass https://cbp-us.nccp.netflix.com;
            proxy_redirect default;
        }
    }

### Setting Up Dnsmasq ###

Dnsmasq is extremely simple to setup.  Simply install it from the ports.

    # cd /usr/ports/dns/dnsmasq
    # make install clean

Then add the following to the configuration file.

    address=/movies.netflix.com/w.x.y.z
    address=/cbp-us.nccp.netflix.com/w.x.y.z
    address=/moviecontrol.netflix.com/w.x.y.z

Once you start Dnsmasq, you should be able to query it using `dig`.

    $ dig @w.x.y.z movies.netflix.com
    
    ; <<>> DiG 9.8.1-P1 <<>> @w.x.y.z movies.netflix.com
    ; (1 server found)
    ;; global options: +cmd
    ;; Got answer:
    ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 43492
    ;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0
    
    ;; QUESTION SECTION:
    ;movies.netflix.com.            IN      A
    
    ;; ANSWER SECTION:
    movies.netflix.com.     0       IN      A       w.x.y.z
    
    ;; Query time: 0 msec
    ;; SERVER: w.x.y.z#53(w.x.y.z)
    ;; WHEN: Sun Mar 18 23:16:30 2012
    ;; MSG SIZE  rcvd: 52

Finally, change your DNS to the Dnsmasq server and try accessing [Netflix](http://netflix.com/)!  You should be able to browse the content, but when you try to play content, you will receive an error.  The problem is we generated our own self-signed SSL certificate and no browsers will trust it.

You must access [https://cbp-us.nccp.netflix.com/](https://cbp-us.nccp.netflix.com/) and accept the certificate.  Now Netflix streams should work.

## Attempt 3:  Squid and BIND ##

Coming soon!

If you have any comments, please leave them below.
