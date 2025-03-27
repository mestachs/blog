---
title: "Single page app everywhere"
date: 2025-03-07T09:58:44+01:00
draft: false
tags: ["development", "github", "spa", "single page app"]
---

## The inventory

Small inventory of my side projects deployed as single page app

- [PDF Toolbox](https://mestachs.github.io/pdf-toolbox/) : merge, compress, turn jpg/png into pdf.
- [Taskr](https://mestachs.github.io/taskr/#/gh/g/mestachs/c0fd9058cf5b7a02eae11e1d77ca4d09) : small notebook environment in js easily create recipe taking xlsx, geopackage, geojson and visualize results as table or maps.
- [CSV viewer](https://mestachs.github.io/csv-viewer/) : you can share csv from private gist rendered/filterable/... ex running queries on a postgres, faulty data
- [Checkl](https://mestachs.github.io/checkl/?gist=https://gist.github.com/mestachs/068e3fd98e205db9e78ef3b1c63f4adc) : my markdown based engine used to render runbook, ops checklist, blog,... 
- [Shary](https://mestachs.github.io/shary/) : qr code generator (allow to share something between my pc and my phone like a ngrok complex url)
- [Mastodon thread unroll](https://mestachs.github.io/mastodon-unroll/) : unroll mastodon thread like a boss to ease reading it.
- [beSolitaire](https://mestachs.github.io/besolitair/#/) : card playing
- [belgium](https://mestachs.github.io/belgium/#/europe/ ) : written for my kids to learn provinces, chef-lieu, european countries/capitals.

## The stack

The stack evolved with time, initally with create-react-app, npm or yarn.

It seem I'm stabilizing it to 
 - [bun.sh](https://bun.sh/guides/ecosystem/vite) and vite : to install dependencies, run, hot reload, production build
 - [gh-pages](https://www.npmjs.com/package/gh-pages) to deploy on github pages

## Just a browser

From the projects above, it's crazy what you can do with _just a browser_.

You have plenty of librairies 
- to access geopackage, sqlite, xlsx, 
- to manipulate geo data (turfjs, spatialite, duckdb)
- based on wasm to do more crazy stuff (compress pdf by porting a c program)
- note you also can have access to a local file system through vanilla js

The main limitations are probably CORS to access external data/api which are great for security but way more annoying for such "browser" only applications.


## But why I'm doing that ?

It's **cheap** and for the moment everything is opensource so free to host via github.
It's **fun**.

Note it's a pattern that I'd like to promote at work : currently we have plenty of servers doing nothing, waiting for someone to came in, I don't need lambda crazyness (and price risk), it feels this way of developping scales much better (why doing heavy queries in the backend for stuf that are nearly static or could work offline). You might argue that it's moving the burden on the client and not more environnemental friendly (data transferts, asking more resources on the client too).

You can _collect or pre bake the data_ via github actions
 - think scrap data from an api (clean, normalize, pre-calculate things on this data)
 - commit that data to keep a version of it over time
 - deploy a new static site with this data that will be snappy like hell

Ex a friend of mine did something like that to collect daily the covid data and commit it, I modified my belgium app to display this data as choropleth from it's extracts.
  
This pattern has apparently a name [baked-data](https://simonwillison.net/2021/Jul/28/baked-data/) and I shared some research on [mastodon](https://mestachs.github.io/mastodon-unroll/?q=https://mastodon.green/@mestachs/112034104506584614)

I think duckdb and parquet files hosted on s3 and a nice js app can beat a lot the current apps.
