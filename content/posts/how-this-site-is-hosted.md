---
title: "How this site is hosted"
date: 2020-02-08T11:41:06-05:00
draft: false
---

This site was designed to be as simple and fast as possible.

* No client-side javascript
* No server management
* HTTPS only and no certificate management
* Source files for blog posts should be markdown
* Costs should be fixed

My previous attempt at this was based on [The One Cent Blog](https://hugotunius.se/2016/01/10/the-one-cent-blog.html). This used [S3](https://aws.amazon.com/s3/), [Jekyll](https://jekyllrb.com/), [Cloudflare](https://www.cloudflare.com/), and [Travis-CI](https://travis-ci.org/) (for triggering a bucket update when a new commit hits Github). It has the basic parts I wanted - a static site generator, and automated/free tools in front to manage certificates and do caching/DDOS protection, etc. S3 is fantastic, and serving static sites is an officially supported use case, however it's a bit overkill for a basic site. The One Cent Blog does a good job of using the correct tools like AWS Cloudformation to build up a stack and give credentials to Travis-CI for writing to the bucket, but that's an entire account, credentials, and billing system that I wanted to avoid for this use case.

## Serving at the edge.

Ideally I want all the bits served from this domain to come from a CDN. Initially I was looking into [Fastly](https://www.fastly.com)'s [Edge compute platform](https://www.fastly.com/products/edge-compute) with the idea that I could embed a simple Rust web server compiled to Wasm in Fastly's edge, and serve my site directly from the CDN layer. This was pretty experimental and would have cost $50/month at Fastly, so I was looking for another solution.

Cloudflare (which I was already planning to use for CDN, analytics, WAF, etc...) recently introduced [Workers](https://workers.cloudflare.com), which compete in the same space as Fastly's Edge programming. You can run Javascript or Wasm code at the edge, on every request. Finally, they have a [KV product for Workers](https://www.cloudflare.com/products/workers-kv/), which provides a key-value data store that can easily be accessed from Worker code. This is exactly what I was looking to implement in a custom Wasm server: look up a request URL in a hash map and return some HTML.

## The stack:

- [Hugo](https://gohugo.io/)
- [Cloudflare Workers](https://workers.cloudflare.com)

That's it. Hugo is a bit heavyweight for my needs, but it's been very easy to use and it's written in Go, so it's a single binary. The theme I'm using with Hugo is is [Notepadium](https://themes.gohugo.io/hugo-notepadium).

I use Cloudflare's [Wrangler](https://developers.cloudflare.com/workers/tooling/wrangler/) CLI tool to publish the site to my Cloudflare Worker via the Cloudflare API. It uses a single namespace in the KV store, and maps each URL to a html file that is returned when the URL is requested.

```bash
λ wrangler publish
 Using namespace for Workers Site "__kerby_website-workers_sites_assets"
 Uploading...
 Success
⬇️ Installing wranglerjs...
⬇️ Installing wasm-pack...
 Built successfully, built project size is 10 KiB.
 Successfully published your script to kerby.website/*
 ```

## Results

It's fast enough. The root url scores a perfect 100 in Chrome's lighthouse audit for performance. There is definitely a bit of a delay when hitting the site cold, where Cloudflare has to spin up a Worker. Overall, I'll take that tradeoff for the simplicity of this setup.

Recurring costs are $5 per month.