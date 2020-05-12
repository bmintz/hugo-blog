---
title: "Let's Encrypt With nginx Using Certbot's Standalone Mode"
date: 2019-11-11T14:12:07-06:00
draft: false
---

Most guides recommend using Cerbot's [nginx plugin](https://certbot.eff.org/lets-encrypt/debianbuster-nginx),
but when I asked about it in the nginx IRC channel, I was told this:

> Let's Encrypt is a neat service, but certbot's nginx mode is an ugly pile of garbage
> that modifies nginx configs in potentially confusing if not broken ways.
> Use it in standalone mode (HTTP-01 or DNS-01) instead; you can proxy to it with nginx.

I was unable to find a guide for this so I figured it out myself. Here's how I set it up:

```
# /etc/nginx/conf.d/upstream-letsencrypt.conf
upstream letsencrypt {
	server 127.0.0.1:8888;
}
```

```
# /etc/nginx/sites-available/mysite.example
server {
	server_name mysite.example www.mysite.example;
	
	listen 443 ssl http2;
	listen [::]:443 ssl http2;
	ssl_certificate /etc/letsencrypt/live/mysite.example/fullchain.pem;
	ssl_certificate_key /etc/letsencrypt/live/mysite.example/privkey.pem;
	# this may be located elsewhere, such as /usr/lib/python3/dist-packages/certbot/ssl-dhparams.pem
	ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
}

server {
	server_name mysite.example www.mysite.example;
	listen 80;
	listen [::]:80;
	location / {
		return 301 https://$host$request_uri;
	}
	location /.well-known/acme-challenge/ {
		proxy_pass http://letsencrypt;
	}
}
```

To test that nginx is proxying to certbot properly, run:

```
certbot certonly \
	--standalone \
	--http-01-port=8888 \
	-d mysite.example,www.mysite.example \
	--non-interactive \
	--register-unsafely-without-email \
	--agree-tos \
	--dry-run
```

(Read the man page on `--register-unsafely-without-email`.
If you want to specify an email address instead use the `--email` flag instead.)

If that works, run it again without the `--dry-run` flag. 

Now certbot will install a renewal script either as a cron job or as a systemd timer. Change it so that it executes
`/usr/bin/certbot -q renew --http-01-port=8888` instead.
If you're using systemd, run `systemctl edit certbot.service`, and put this in the temporary file it creates:

```
[Service]
ExecStart=/usr/bin/certbot -q renew --http-01-port=8888
```
