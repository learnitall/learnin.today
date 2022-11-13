---
title: "Redirecting to resume..."
except: "Redirecting to resume..."
permalink: "/resume"
layout: "single"
---

{% assign redirect_url = "/assets/documents/ryan_drew_resume_11132022.pdf" | prepend: site.baseurl | prepend: site.url %}
<meta http-equiv="refresh" content="0; url={{ redirect_url }}">

<a href="{{ redirect_url }}">Click here if you are not redirected.</a>
