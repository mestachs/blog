---
title: "DNS/Certificate Enumeration"
date: 2022-11-12T23:31:44+01:00
draft: true
tags: ["TIL", "security"]
---

By design you can't query dns to get "all the records/servers" in that domain or subdomains.
At best you might find some links (ex mail servers) or guess some server names from naming conventions.

You have plenty of [tools](https://github.com/nixawk/pentest-wiki/blob/master/1.Information-Gathering/How-to-gather-dns-information.md) trying to crawl/enumerate them in a _brute_ force approach or using google or other search engines.

But with the rise of automatic https certificate it's easier than ever since most service are over https.
It's now possible to find all hosts that requested a certificate.

This crt.sh site can give you all the hosts from google.com : https://crt.sh/?q=google.com

![image](https://user-images.githubusercontent.com/371692/201496725-9e106477-6b4f-461d-b993-ca15112fab8c.png)
