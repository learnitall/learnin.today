---
title: "Redirecting to gpg key..."
excerpt: "Redirecting to gpg key..."
permalink: "/gpg"
layout: "single"
---

{% assign redirect_url = "/assets/documents/ryan_drew_public.gpg" | prepend: site.baseurl | prepend: site.url %}
<meta http-equiv="refresh" content="0; url={{ redirect_url }}">

<a href="{{ redirect_url }}">Click here if you are not redirected.</a>
