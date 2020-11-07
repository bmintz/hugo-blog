+++
title = "Let's Encrypt With nginx Using Certbot's Standalone Mode"
date = "2019-11-11T14:12:07-06:00"
lastmod = "2020-11-07T15:11:46-06:00"
draft = false
+++

Most guides recommend using Cerbot's [nginx plugin](https://certbot.eff.org/lets-encrypt/debianbuster-nginx),
but when I asked about it in the nginx IRC channel, I was told off. I was unable to find a guide for this so I figured it out myself. I now have certbot and nginx working together to provide automatically renewed TLS certificates that are trusted by all clients I care about. Here's how I set that up.

Let's Encrypt's validation servers work by requesting that you respond to an HTTP request at your domain,
for a specific URL, with a specific value. Certbot has a mode called "standalone" which responds to these requests
directly, which has a few advantages over the nginx plugin and the "webroot" mode:

- Certbot doesn't need write access to your web root, or to your config files
- You don't need to *have* a web root. Some configs use nginx strictly as a reverse proxy with no static content.

**All of the following steps require root access and have only been tested on Debian, although may work on other OSes.**

## Installing certbot and nginx

On Debian: `apt install certbot nginx`

## Configuring Certbot

Tell certbot to listen on some arbitrary port when it's time to renew.
It doesn't matter what port, just so long as it's not already in use.

**/etc/letsencrypt/cli.ini**<br>
```
standalone
http-01-port = 8888
```

**/etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh**<br>
```
#!/usr/bin/env sh
systemctl reload nginx
```

`chmod +x /etc/letsencrypt/renewal-hooks/deloy/reload-nginx.sh`

This makes sure that when a certificate is issued or renewed, nginx is made aware of it. Otherwise, after renewal,
nginx will still be using outdated certificates.

## Configuring nginx

Tell nginx to proxy *all* requests for /.well-known/acme-challenge over port 80 to certbot.

```
# /etc/nginx/conf.d/upstream-letsencrypt
upstream letsencrypt {
	server 127.0.0.1:8888;
}
```

We place that in a separate file so that we can reuse that `upstream` configuration across multiple websites.

```
# /etc/nginx/sites-available/mysite.example
server {
	server_name mysite.example www.mysite.example;
	listen 80;
	listen [::]:80;
	location /.well-known/acme-challenge/ {
		proxy_pass http://letsencrypt;
	}
	# if you want to redirect all http:// traffic to https://, use this too:
	location / {
		return 308 https://$host$request_uri;
	}
}
```

- Enable the config: `ln -s /etc/nginx/sites-{available,enabled}/mysite.example`
- Reload nginx: `systemctl reload nginx`

To test that nginx is proxying to certbot properly, run:

```
curl -I http://www.mysite.example/.well-known/acme-challenge/foo
```

If you do not get back `502 Bad Gateway` in the response, double check your configs, `/var/log/nginx/error.log`, `/var/log/nginx/access.log`, and `systemctl status nginx`. `502` is the correct response because the Certbot standalone server only runs when it needs to.

## Getting a certificate

```
certbot certonly -d mysite.example,www.mysite.example --dry-run
```

The `--dry-run` flag tells certbot to do two things differently:

1. Do not actually issue certificates, just test to make sure that Let's Encrypt can properly verify that you own the domain.
2. Use [Let's Encrypt's staging servers], which have much more forgiving rate limits in case you mess up.

If that works, run it again without the `--dry-run` flag. If *that* works, congrats!

[Let's Encrypt's staging servers](https://letsencrypt.org/docs/staging-environment/)

## Configuring nginx to use the certificate

We'll add a new server block to the same nginx config as before that tells nginx to use the new certificates that certbot just deployed. Put this anywhere in `/etc/nginx/sites-available/mysite.example`:

```
server {
	server_name mysite.example www.mysite.example;

	listen 443 ssl http2;
	listen [::]:443 ssl http2;
	ssl_certificate /etc/letsencrypt/live/mysite.example/fullchain.pem;
	ssl_certificate_key /etc/letsencrypt/live/mysite.example/privkey.pem;
	# this may be located elsewhere, such as /usr/lib/python3/dist-packages/certbot/ssl-dhparams.pem
	ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

	# actual content configuration goes here
	# for you this may mean `location` blocks, or `root` directives, `proxy_pass` directives, or some combination
	# of these
}
```

Now reload nginx again: `systemctl reload nginx`

If all goes well, running `curl https://www.mysite.example/` should give you a 404 Not Found and zero certificate errors. Now go configure nginx to serve your actual website and have fun hosting it yourself =)
