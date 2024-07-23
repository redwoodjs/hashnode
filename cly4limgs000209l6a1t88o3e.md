---
title: "Moving our site from Netlify to Fly.io"
seoTitle: "Moving our site from Netlify to Fly.io"
datePublished: Tue Jul 02 2024 16:00:51 GMT+0000 (Coordinated Universal Time)
cuid: cly4limgs000209l6a1t88o3e
slug: moving-our-site-from-netlify-to-flyio
tags: hosting, javascript, javascript-framework, redwoodjs, flyio

---

We recently switched our [main website](https://redwoodjs.com/) from Netlify to [Fly.io](http://Fly.io). This was a pretty smooth process and we're happy with the results. Here's a quick overview of our experience.

### Why we switched

[Netlify](https://www.netlify.com/) is awesome. It's super easy to use and one of the best compliments you can give a deploy provider is that it has been rock solid. We have never had a significant problem and we've been using them for years.

We decided to switch to [Fly.io](http://Fly.io) simply because we want to make use of a persistent server for our deployment. With some of the Redwood features we want to start taking advantage of, e.g. SSR, it makes more sense for us to deploy in a serverful way rather than serverless.

### Deploying with [Fly.io](http://Fly.io)

Fly takes the image of your containerized application and uses that image in a virtual machine to run your application. Fly has support for Redwood when you run `fly launch` so this is exactly how we'll get started.

After running through that command we have our application deployed to [`bighorn-website.fly.dev`](http://bighorn-website.fly.dev) but because we're using the canary version of Redwood we need to make some tweaks before everything works correctly.

I won't go into all the details here and some of our canary setup is subject to change so I would recommend looking at [this](https://github.com/redwoodjs/bighorn-website/commit/741f9652251208544b9de32a7b323d037ba56a35) commit for our current Dockerfile and `./fly/`[`start.sh`](http://start.sh) file setup.

The main changes we chose to make were:

1. Switched away from the Dockerfile Fly generated and used the Redwood Dockerfile [template](https://github.com/redwoodjs/redwood/blob/main/packages/cli/src/commands/experimental/templates/docker/Dockerfile).
    
2. Altered the start script for our Fly machine to start both the web and API server processes. You should follow Fly's advice on multiple processes [here](https://fly.io/docs/app-guides/multiple-processes/).
    

### Networking and DNS

Now that we have our website running on Fly it's time to have the full world use it. This means pointing our [`redwoodjs.com`](http://redwoodjs.com) domain name to the Fly server that runs our application.

Fly has some great [documentation](https://fly.io/docs/networking/custom-domain/) on using a custom domain so that we can avoid having to use [`bighorn-website.fly.dev`](http://bighorn-website.fly.dev) forever.

This could be done in two ways. Either with a `CNAME` or with `A` and `AAAA` records. With `CNAME` we would simply add a record that maps [`redwoodjs.com`](http://redwoodjs.com) to [`bighorn-website.fly.dev`](http://bighorn-website.fly.dev). When using `A` and `AAAA` records we would add those records pointing to our Fly machine's IP addresses and then use the Fly [CLI](https://fly.io/docs/flyctl/certs-add/) to set up a TLS certificate for our domain.

### Automating deployments

Continuous Deployment (CD) doesn't just sound fancy it's also really convenient. Not having to manually redeploy your site when you make a change can make a difference in how much friction you feel.

Using Fly this was super easy! Again, they have some very straightforward [documentation](https://fly.io/docs/app-guides/continuous-deployment-with-github-actions/) that walks you through the full process. For us, we had to take 3 main steps:

1. We create a deploy token using the Fly CLI:
    

```plaintext
fly tokens create deploy -x 999999h
```

2. We add that token to our GitHub repository secrets as `FLY_API_TOKEN`
    
3. We then add a small GitHub action (`.github/workflows/fly.yml`) to redeploy our site when changes happen on our main branch:
    

```yaml
  name: Fly Deploy
  on:
  push:
     branches:
    - main
  jobs:
  deploy:
     name: Deploy application
     runs-on: ubuntu-latest
     concurrency: deploy-group # Only one deploy job runs at a time
     steps:
    - uses: actions/checkout@v4
    - uses: superfly/flyctl-actions/setup-flyctl@master
    - run: flyctl deploy --remote-only
        env:
           FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }}
```

### Result

After all this we have a site deployed on Fly that we can reach with our custom domain: [redwoodjs.com](http://redwoodjs.com)

With having set up some of Redwood's canary features previously, we now get the usual set of features you'd expect from a modern web application. SEO-related metatags and SSR to name two.