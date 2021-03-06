sftdyn is a minimalistic dynamic DNS server that accepts update requests via HTTP(S) and forwards them to a locally running DNS server via `nsupdate -l`.

It lets you easily create a dyndns.org-like service, using your own DNS server, and can (probably) be used with your router.

## Quick Guide

This guide assumes that you're using bind, your zone is dyn.sft.mx, and your server's IP is 12.345.678.901. You'll want to adjust that.

#### Nameserver
Somewhere in `named.conf`, add

    zone "dyn.sft.mx" IN {
        type master;
        file "/etc/bind/dyn.sft.mx.zone";
        journal "/var/cache/bind/dyn.sft.mx.zone.jnl";
        update-policy local;
    };

`/var/cache/bind` must be writable for *bind*.

Create the empty zone file

    cp /etc/bind/db.empty /etc/bind/dyn.sft.mx.zone

If you want to use `dyn.sft.mx` as the hostname for your update requests, add a record to the zone file:

    IN A 12.345.678.901

#### sftdyn
To install *sftdyn*, use `pip install sftdyn` or `./setup.py install`. Launch it with `sftdyn [command-line options]`.

Configuration is by command-line parameters and conf file. A sample conf file is provided in `sample.conf`. If no conf file name is provided, `/etc/sftdyn/conf` is used. hostnames/update keys are specified in the conf file.

You _can_ use sftdyn in plain HTTP mode. Your average commercial dynamic DNS provider provides a HTTP interface, so most routers only support that.

However, for security reasons, you _should_ use HTTPS. For that, your server needs a X.509 key and certificate. You can buy a cert from your DNS provider, or create a self-signed one; both have their benefits.

To generate `server.key` and a self-signed `server.crt` valid for 1337 days:

    openssl genrsa -out server.key.org 4096
    openssl req -new -key server.key -out server.csr
    openssl x509 -req -days 1337 -in server.csr -signkey server.key -out server.crt
    rm server.csr

Make sure you enter your server's domain name for _Common Name_.

sftdyn _should_ run under the same user as your DNS server, or it _might_ not be able to update it properly.

#### Client
To use your router as client, select _user-defined provider_, enter http://dyn.sft.mx:8080/yourupdatekey as the update URL, and random stuff as domain name/user name/password. (tested with my AVM Fritz!Box. YMMV). Most routers don't support HTTPS update requests, so you'll probably need HTTP.

To use your GNU+Linux box as client, just set up a cronjob that talks to sftdyn every few minutes:

    */10 * * * * curl https://dyn.sft.mx:4443/mysecretupdatekey

If you use HTTPS with a self-signed certificate, curl will refuse to talk to the server.
 - Use `curl -k` to ignore the error (Warning: see the security considerations below).
 - Copy `server.crt` to the client, and use `curl --cacert server.crt`.

| HTTP code     | Text          | Response interpretation             |
| ------------- | ------------- | ----------------------------------- |
| 200           | OK            | Update successful                   |
| 200           | UPTODATE      | Update unneccesary                  |
| 403           | BADKEY        | Unknown update key                  |
| 500           | FAIL          | Internal error (see the server log) |
| 200           | _your ip_     | Returned if no key is provided      |

## About
I wrote this script after the free dyndns.org service was shut down. After a week or so of using plain nsupdate, I was annoyed enough to decide to write this.

It is the main goal to stay as minimal as possible; for example, I deliberately didn't implement a way to specify the hostname or IP that you want to update; just a simple secret update key is perfectly good for the intended purpose. If you feel like it, you can make the update key look like a more complex request; every character is allowed. Example: `?host=test.sft.mx&key=90bbd8698198ea76`.

The conf file is interpreted as python code, so you can do arbitrarily complex stuff there. You can also update server.clients at runtime using the interactive console.

## Security considerations

- When using HTTP, or if your `server.key` has been stolen or broken, an eavesdropper can steal your update key, and use that to steal your domain name.
- When using HTTPS with `curl -k`, a man-in-the-middle can steal your update key.
- When using HTTPS with a paid certificate, a man-in-the-middle with access to a CA can steal your update key (no problem for government agencies, but this is pretty unlikely to happen).
- When using HTTPS with a self-signed certificate and `curl --cacert server.crt`, no man-in-the-middle can steal your update key.

sftdyn is pretty minimalistic, and written in python, so it's unlikely to contain any security vulnerabilities. The python ssl and http modules are used widely, and open-source, so there _should_ be no security vulnerabilities there.

Somebody who knows a valid udpate key could semi-effectively DOS your server by spamming update requests from two different IPs. For each request, nsupdate would be launched and your zone file updated.

## Development
IMHO, the project is feature-complete; it has everything that **I** currently want.

Features that _might_ be useful, which I _might_ implement if someone asked nicely:
 - Support to run this inside an Apache web server (WSGI?)
 - Initscripts for _insert your distro here_
 - I'm sure there are more

If you have any requests, ideas, feedback or bug reports, are simply filled with pure hatred, or just need help getting the damn thing to run, join `irc.freenode.net/#sfttech` (I'm mic_e).

If you actually _did_ implement a useful feature in a sufficiently non-bloaty way, please send a pull request; I'd be happy to merge it.

The license is GNU GPLv3 or higher.
