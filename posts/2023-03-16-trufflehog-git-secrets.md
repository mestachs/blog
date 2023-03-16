---
title: "Trufflehog, git and secrets "
date: 2023-03-16T07:58:44+01:00
draft: true
tags: ["development", "secrets", "trufflehog"]
---

# The problem

With trends like putting more and more code bases as open source, developping in the *open* can become a security risk.
Open sourcing an old code base that wasn't in the state of the art, you know some junior dev worked on it or shortcuts where made in the early beginning of the project might a source of credentials leak, perhaps not obviously in the `main` branch hidden more deeply in the repository.

# The solution

There are a few tools to allows to locate such sensitive data in the repository. 
In this article I'll mainly focus on [trufflehog](https://github.com/trufflesecurity/trufflehog) and their vesion 3, a golang rewrite of their old python library.


## Finding credentials in your code base

With [trufflehog](https://github.com/trufflesecurity/trufflehog), you can be locate credentials such as usernames, passwords, certificates/private keys or API keys that are used to authenticate access to certain resources. These credentials may be necessary for accessing sensitive data or performing certain operations, and it's important to keep them secure and not expose them publicly. 

## Improving a bit trufflehog

So you think you or a colleague leaked some credentials in the code and want to give trufflehog a try.

Trufflehog is great at searching the whole git repo (all commits) and is looking for a lot of know patterns of API keys (for various SAAS/PAAS/.. products)

Sadly something that was omitted in the current version is searching for hardcoded secret, which is strangely the thing happening most of the time (at least in my entourage). It's probably because this kind of scanning is source of a lot of false positive, but franckly is a necessary work.

So let's add a small config to trufflehog with regexp that would match some of the patterns I found generally leaking these credentials.

Store this in a file (`hog`)

```
detectors:                
  - name: hard coded password detector ultimate
    keywords:
      - password
      - pass
      - PASSWORD
    regex:
      adjective: '(.*)?(password|PASSWORD|pass)['',"]?[\]]?(\s+)?(=>|:|=|==)(\s+)?[''|"](\S+)[''|"]'                 
```
Note I'm not a regexp expert but it tries to match some "constant" idioms I found in js, ruby, python, php, `.env`, bash.
This should perhaps be improved to look for other keywords too (token, secret, encryption_key, access_key, ...)

like the following snippets

```
$db['production']['password'] = "supersecretpassword_in_php";

connection = { password:'mysecretisnomoresecretenowindictsymbol' }
connection = { 'password' => 'mysecretisnomoresecretenowindict'}
password='mysecretisnomoresecretenowwithquotes'
NASA_PASSWORD= "mysecretisnomoresecretenowwithquotes"
```

then you can reference that file while running trufflehog

```
docker run --rm -it -v "$PWD:/pwd" -v "$PWD/../:/config" trufflesecurity/trufflehog:latest git file:///pwd --config=/config/hog
```

## Running the tool against your code

Trufflehog is quite easy to run against local directory, all the github repo of a user/organisation,...
This take sometime but you might find some of the results interesting.

## What you might find

I did the exercice on some of the repo, and found a lot of stuff, too much stuff in fact.

Some of the findings were
 
### False positive
  
 * either user password to create a user for unit testing purpose (by the way it would be great to fix a kind of standard for that)
 
 ``` 
 Raw result: "password": "unittest_password"
 Raw result: "dhis_password": "test_password"
 Raw result: user = User.objects.create_user(username='testuser', password='12345'
 ```
 
 * "publicly" known credentials. Ex https://play.dhis2.org/ propose demo instance with an admin user with password 'district' 
 
 ```
 Raw result: client = Dhis2::Client.new(user: "admin", password: "district"
 Raw result: https://admin:district@play.dhis2.org
 ```
 
In my experience this is source of most false positive, a first manual review is necessary.
Sadly trufflehog doesn't seem to offer such exclusion mechanim. 
So I redirected the json output to a python script "hiding" the known false-positive.
Let's hope this will move in the tool it self at some point.
 
### Some leaked but outdated

 - the password was already rotated
 - the user was deleted
 - the api keys invalidated
 
### Some "not well handled" leak

The code has been fixed but the commit is still in the repo, referencing the hardcoded crendentials
and the credentials are still valid... oups

### Some leak you didn't know at all

In this family I found 
 - some "nodejs crash report" with a json embedding all the env variables that were accidently commited
 - some config file (ex: database.yml .env) that were committed instead of `.gitignore`d
 - some commit at early stage of the project where infra wasn't yet "well thought" (you know the kind of POC that goes to production)
 - some test fixture like json api results recorded with something like [vcr](https://github.com/vcr/vcr)
 - some seeds command that were "bundling" some production data (wait what... or the seed became the prod I don't know)

Note that for some detectors trufflehog can try to verify if the token found is a valid one (ex an aws key)

```
Found verified result 🐷🔑
Detector Type: AWS
Decoder Type: PLAIN
Raw result: AKIASP2TPHJSZD4CPS4E
File: conf/secrets.yml
Line: 2
Timestamp: 2023-02-18 12:26:47 +0100 +0100
```

Don't worry this one was on purpose, generated by [canarytoken](https://canarytokens.org/generate)

 
# The conclusion 

It's a great tool, but only shows problems, not what to do with these problems.

## What to do when you find out about a leak

The only true way of handling things, is this 5 step process

1. inform your colleagues/boss, that will decide how to communicate to security officer, clients,... 
2. find where these credentials might have been used, get someone investigate if these credentials might have been used outside your organisation.
3. implement a fix in your code to use a secured way of handling crendentials.
4. rotate/invalidate these credentials.
5. put everything in production.

things to avoid : 

 - just commit a fix using env variables : git never forgets, trufflehog will find the "old" credentials that are probably still usable => rotate/invalidate everything you found
 - asking to delete the problematic data to github/bitbucket/gitlab : this will be hard, slow and the correct answer is rotate/invalidate that's it. 
     - Scripts are running all day long scanning repositories to find such API keys (ex aws keys for attack, extorsion, vandalism, to spin off ec2 instances for larger attacks, bitcoin mining,...)
     - someone might find a old copy of the repo during a target attack, and use that to pivot to the other system 
         - think a continuous integration service got hacked, or someone found an laptop from your company

## How to prevent such a leak 

The general solution for these secrets is to externalize them but solution varies from basic to more paranoid mode

- environnement variables as would [12factor apps](https://12factor.net/config)
   - ruby [dotenv](https://www.rubydoc.info/gems/dotenv/2.1.1) or
   - python [dotenv](https://pypi.org/project/python-dotenv/)
   - js [dotenv](https://www.npmjs.com/package/dotenv)
- a config file not stored in the git repository 
- an encrypted config file (secrets.yml)
- an external system that will allow to inject the secret at runtime (like [Vault](https://www.vaultproject.io/),  [sops](https://github.com/mozilla/sops) or k8s [secrets](https://kubernetes.io/docs/concepts/configuration/secret/)

And with this in place, put your secrets/credentials in a password manager (like 1password) and make your sure people have only access to the entries to be able to do their job (don't give the aws root account to the marketing team ;))

A second safety net is to setup git pre commit [hook](https://github.com/trufflesecurity/trufflehog#precommit-hook) to do the same scanning when commiting on your laptop.

## What to do to minimize problems

To minimize the impact of such leak
 -  **The principle of least privilege** : limits users' access rights to only what are strictly required to do their jobs. So keep the access rights in the target system to the minimum necessary. For example you need AWS API keys to access a bucket, make sure to use a dedciated AWS Iam user and a dedicated access policy : that grant only access to that bucket, perhaps only list, write but not delete operations, not all buckets, not the AWS web console, not the ec2/rds,...
 - we should **avoid reusing the credentials** in other apps/context, this will complexify the process of rotating the credentials.
 - **use unique generated secrets** and don't reuse them in other projects (I remember a team always using the same password accross serveral products/saas... because they probably didn't had a password manager)
 
## Continuously run these tools

Here it's a one time check, but this should be done continuously ;)

There now, I'll need to handle "leaked but resolved" secrets as false positive ;)
