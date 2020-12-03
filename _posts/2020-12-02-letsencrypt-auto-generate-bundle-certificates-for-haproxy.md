---
layout: post
title:  "Letsencrypt: auto generate bundle certificates for Haproxy"
tags: [ letsencrypt, ssl, tls, haproxy ]
---

I was surprised to not get instant answer googling "letsencrypt automatically generate bundle certificates for haproxy".

So here it is, to save 5 minutes of your time.

Generates bundle(fullchain + privkey) certificates for everything in `/etc/letserncrypt/live/*`, does not update generated files if not necessary.

<script src="https://gist.github.com/selivan/a65b2c8dfe1a2563b50d822727fa7d0f.js"></script>

<noscript><a href="https://gist.github.com/selivan/a65b2c8dfe1a2563b50d822727fa7d0f">gist generate-bundle-certs.sh </a></noscript>
