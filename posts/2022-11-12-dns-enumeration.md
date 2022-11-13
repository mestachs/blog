---
title: "DNS/Certificate Enumeration"
date: 2022-11-12T23:31:44+01:00
draft: true
tags: ["TIL", "security"]
---

By design you can't query dns to get "all the records/servers" in that domain or subdomains.
At best you might find some links (ex mail servers) or guess some server names from naming conventions.

You have plenty of [tools](https://github.com/nixawk/pentest-wiki/blob/master/1.Information-Gathering/How-to-gather-dns-information.md) trying to crawl/enumerate them in a _brute_ force approach or using google or other search engines.

But with the rise of automated https certificate, and the [certificate transparency](https://certificate.transparency.dev/), it's easier than ever since most service are over https.
It's now possible to find all hosts that requested a certificate.

There's a site [crt.sh](https://crt.sh) that offer such a data base (as a ui and as a db). It's developped by [robstradling](https://github.com/robstradling) and it's open sourced on [github](https://github.com/crtsh) ! At time of writing, since 2013, 2,563,437,540 certificates have been logged.
This crt.sh site can give you all the certificates issued for a give domain google.com : https://crt.sh/?q=google.com

![image](https://user-images.githubusercontent.com/371692/201496725-9e106477-6b4f-461d-b993-ca15112fab8c.png)

While this information might be useful to be sure you are talking to the good server (web PKI), it might be used by bad actors or pen testers looking for more vulnerable host in your wider infrastructure than your main site.

Note also that crt.sh also offer access to a replicated postgres instance (be gentle with the db).

```bash
psql -h crt.sh -p 5432 -U guest certwatch
```
