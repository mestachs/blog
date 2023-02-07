---
title: "Rules of engagement before touching a server"
date: 2022-12-28T07:58:44+01:00
draft: true
tags: ["devops"]
---

# Rules of engagement before touching a server or its data

Strangely "ops" work, migration/fixes of "production" data,... is often stuff you have to learn the hard way (alone, during a crisis) and/or in binome/team (with some more senior colleague) or following procedures/runbooks (for more mature organisation).

It's generally

- not tought,
- discouraged (only a few people have access to the production )
- acquired with experience (your own, your senior colleague or as company through defined procedures/runbooks )

That's why I tried to document a bit some principles to follow for any planned (I have a jira) or unplanned (the monitoring or people complains) intervention on server (upgrade, restart, decommissioning,... ) or their data (sql, api, mass import,...).

It might look like common sense but when you are unprepared this really helps. If you ended up in a position where you are accidentaly the "ops/dba" person this might be a good read ;)

## Keep calm and focus

- no one is dying, it's probably not your fault even if you think it is (perhaps the company fault to let you do risky stuff) but not your fault
- don't be afraid to ask for advice/help/disambiguation
- in case unplanned incident, depending on the severity call/slack your colleague(s), project manager, 'boss', management team
- do one thing at time, it might be tempting to do something else during "waiting time" (think restoring a huge dump) in procedure but don't get distracted by running another task at the same time. As pilots says [Lose sight, lose fight](https://en.wikipedia.org/wiki/Fighter_pilot#:~:text=A%20common%20saying%20for%20dogfighting,revealed%20as%20the%20fight%20progresses.). This avoid you to miss a step or to run a command on the wrong server.

## Document what you do and do what you document

- keep a log file of the commands and their outputs (eg number of records updated/deleted, error messages, warnings) (try to persist them as you go, your laptop might crash or the ssh connection might get disconnected : comment it in a jira, slack it as text snippet or put it as a secret/private gist,... )
- put that file in jira or in the slack thread so that others can "react" or "learn" or have it in case they need to "redo" it.
- if a colleague need to step in (you are stuck, you have unexpected error message, you have to leave and pass the torch), this log is of great value
- if it needs to be redone, it will be a good draft for a futur procedure
- if the procedure says something, and you know that this can be done in another way, still stick on the procedure (don't improvise ! you can discuss that later out of the crisis)

## Safety first : don't start destructive operations without proper backup

- if you are removing a db server in aws => last rds snapshot
- if you delete/mass update records => a backup/snapshot + store the content before modification if possible + log of the queries
- if you drop a table => a backup and log the ddl
- if you run a mass update through api/repl/script, make sure the script has a dry run mode where you can export the data "before modification"
- even if it looks like a test, development, demo server => do as if it was a production one, take a backup
  - a mis-labelled server or something that was tagged as staging might have become a production server because in a hurry
  - you deleted the "test" and someone wanted to delete the "staging" (cfr It's better late than sorry rule)
  - time has been invested to configure that db, perhaps someone will change its mind about the deletion of the db,
  - being consistent how you handle your servers will avoid problems, forgetting about this 'safety first' rule when you need it the most!

## Over communicate and communicate publicly

- since touching servers implies you but also your colleagues, clients : communicate either via jira or a slack/mail/... thread
- since you are probably remote, it's the only way people can intervene :
  - "hey wait stop we are doing a demo right now"
  - "don't do that before doing this and this and this"
  - people might notice that the server is up and think the upgrade is done, make your communication clear the operation is still running, give regular progress notification and tell people when you are done and service is back to normal.
- avoid private channels/conversations, use a slack thread in a public channel (#devops, or a dedicated one to the project)
- if you do a zoom call, note the action decided in the channel, share the link so that everybody can take the temperature or share their findings
- announce planned and unplanned operation in the slack channel #devops and idealy in the "business" channel so if the monitoring detects something, someone else won't jump in to restart things while in fact it's expected to stay down. (you might pause the monitoring alerts, don't forget to re enable it)

## In case of doubt : don't move. It's better late than sorry

- you are not sure which db is linked to the server => ask or wait
- the person didn't gave you a target version or the url of the server to upgrade
- if it's something you have never done : ask a colleague to watch you while doing it, a good opportunity to pair/learn/ask questions
- if it's an app/infrastructure you don't know well : ask for help
- remember 5xP : `Proper Preparation Prevents Poor Performance`
  - re read the procedure, disambiguate with your colleague, establish in which case you are
  - eg
    - verify current version before starting an upgrade,
    - I'm moving this server from heroku to aws so need a last backup from heroku,
    - upgrading from hobby postgres to standard plan on heroku is not the same as upgrading from standard to bigger plan.
    - the dns is in ghandi, ovh or aws or handled by client (the later might need coordination with them),
    - prepare/collect/verify the info before upgrading (identifier/url/current version/ products connecting to it that might need to be retested)
    - ...
  - mass import/update ? may be you should increase the db storage first

## Watch first, don't blindly change things

- launch diagnose before restarting a server
  - at work we have a script call `diagnose` that collects last lines in the logs, currently running sql queries, thread stacks, summarize http access/error logs, graphs from monitoring,...
  - this script can be triggered by chatops keeping you away from the server
- a server misbehave ? check everything : storage, is the process running, which logs, does the log match your assumptions ? do you find similar errors message in slack/jira/stackoverflow

## If you plan to change stuff, one change at a time

- change one thing at a time, check the results (undo if the change was not helping and not hard or safe to undo)
- only one person touch, the others watch, keep them posted

## Don't ssh for pleasure

- ssh in last resort and motivated by an incident or jira ticket
  getting insightful info
- try to stay in "readonly"

## Learn from the incident

- most of the team on holliday ? make a document to share with your team mates.
- make a small postmortem and find how to prevent that to happen again
  - monitoring
  - settings changes
  - ...
- create a jira with traces collected so a developper can look at it and perhaps identify improvement/fixes for the root cause.
- take your notes and turn them into a procedure/runbook/script

## TLDR

- Keep calm and focus
- Document what you do and do what you document
- Safety first : don't start destructive operations without proper backup
- Over communicate and communicate publicly
- In case of doubt : don't move. It's better late than sorry.
- Watch first, don't blindly change things
- If you plan to change stuff, one change at a time
- Don't ssh for pleasure (try to automate evidence collection)
- Learn from the incident
