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

Now it's time to discover the schema of the db

The easier is to just append a `&showSQL=Y` to the query

https://crt.sh/?q=google.com&showSQL=Y

```sql
WITH ci AS (
    SELECT min(sub.CERTIFICATE_ID) ID,
           min(sub.ISSUER_CA_ID) ISSUER_CA_ID,
           array_agg(DISTINCT sub.NAME_VALUE) NAME_VALUES,
           x509_commonName(sub.CERTIFICATE) COMMON_NAME,
           x509_notBefore(sub.CERTIFICATE) NOT_BEFORE,
           x509_notAfter(sub.CERTIFICATE) NOT_AFTER,
           encode(x509_serialNumber(sub.CERTIFICATE), 'hex') SERIAL_NUMBER
        FROM (SELECT *
                  FROM certificate_and_identities cai
                  WHERE plainto_tsquery('certwatch', $1) @@ identities(cai.CERTIFICATE)
                      AND cai.NAME_VALUE ILIKE ('%' || $1 || '%')
                  LIMIT 10000
             ) sub
        GROUP BY sub.CERTIFICATE
)
SELECT ci.ISSUER_CA_ID,
        ca.NAME ISSUER_NAME,
        ci.COMMON_NAME,
        array_to_string(ci.NAME_VALUES, chr(10)) NAME_VALUE,
        ci.ID ID,
        le.ENTRY_TIMESTAMP,
        ci.NOT_BEFORE,
        ci.NOT_AFTER,
        ci.SERIAL_NUMBER
    FROM ci
            LEFT JOIN LATERAL (
                SELECT min(ctle.ENTRY_TIMESTAMP) ENTRY_TIMESTAMP
                    FROM ct_log_entry ctle
                    WHERE ctle.CERTIFICATE_ID = ci.ID
            ) le ON TRUE,
         ca
    WHERE ci.ISSUER_CA_ID = ca.ID
    ORDER BY le.ENTRY_TIMESTAMP DESC NULLS LAST;
```

replace the `$1` by `'google.com'` that's it. You can issue this query in psql prompt.

You can probably turn this into a shell script and watch your own domains.

Have fun and stay safe !
