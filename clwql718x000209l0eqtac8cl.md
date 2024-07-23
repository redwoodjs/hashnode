---
title: "Using Middleware: RSS and Sitemap"
seoTitle: "Using RedwoodJS Middleware: RSS and Sitemap"
datePublished: Tue May 21 2024 05:00:00 GMT+0000 (Coordinated Universal Time)
cuid: clwql718x000209l0eqtac8cl
slug: using-middleware-rss-and-sitemap
tags: javascript, middleware, redwoodjs

---

We recently redesigned our website for the new "Bighorn" epoch of Redwood. This included starting this series of blog posts which had a fun side effect of giving us an exercise to solve. How do we implement an RSS feed when our blog is built into a Redwood app?

Let's walk through how we used [Redwood's new middleware functionality](https://redwoodjs.com/blog/middleware-in-redwoodjs) to implement this feature.

**Requirements**

First, we list out our two basic requirements:

1. We need a `/rss.xml` which provides an RSS feed based on our blog posts.
    
2. We also wanted to add a `/sitemap.xml` which lists both some static pages we have but includes the dynamic blog post pages too.
    

**Approaches**

Confronted with this problem there isn't an immediate easy answer with traditional Redwood tools. We can't rely on standard pages as we're not looking for a rendered React page as our response. We also can't rely on API functions because they're nested under some specific path - `/.redwood/functions/` by default.

**Using Middleware**

Middleware lets us intercept requests, perform some logic, and either return a response or let the request flow through to other code that would handle it further. This is exactly what we want in this case. With a basic middleware, we could:

1. Intercept any request to `/rss.xml`
    
2. Fetch all our blog post data, transforming it into an RSS feed
    
3. Respond with that feed, correctly encoded as XML
    

The sitemap would essentially be the same where we intercept requests to `/sitemap.xml` and respond with our sitemap content.

Okay, so how does this look in practice? Well, the first thing we must do is have the correct prerequisites. Right now we have to use the canary version of Redwood to get access to middleware but that is the only requirement here.

*Note: The style here isn't too important. For example, where I've used dynamic imports could easily be a static import. Where I used an async function could have instead been a middleware class.*

We then start by registering our middleware which we do in our `entry.server.tsx` file.

```ts
export  async  function  registerMiddleware() {
  const { middleware: sitemapMw } = await import('./middleware/sitemap')
  const { middleware: rssMw } = await import('./middleware/rss')

  return [sitemapMw, rssMw]
}
```

Then we have to actually implement our middleware logic. I chose to use a simple async function like so:

```ts
export async function middleware(
  req: MiddlewareRequest,
  mwResponse: MiddlewareResponse
) {
  // 1. Intercept if `rss.xml`
  // 2. Grab all blog posts from the API
  // 3. Make an RSS feed from the posts data
  // 4. Respond with an XML encoded response
  return mwResponse
}
```

Let's build out those 4 actions. First, we check whether we should intercept this request or not by checking the request URL:

```ts
export async function middleware(
  req: MiddlewareRequest,
  mwResponse: MiddlewareResponse
) {
  // 1. Intercept if `rss.xml`
  const url = new URL(req.url)
  if (url.pathname !== '/rss.xml') {
    return mwResponse
  }

  // 2. Grab all blog posts from the API
  // 3. Make an RSS feed from the posts data
  // 4. Respond with an XML encoded response
  return mwResponse
}
```

Second, we have to grab the blog post data from our API:

```ts
import { getAllPosts } from './util'

export async function middleware(
  req: MiddlewareRequest,
  mwResponse: MiddlewareResponse
) {
  // 1. Intercept if `rss.xml`
  const url = new URL(req.url)
  if (url.pathname !== '/rss.xml') {
    return mwResponse
  }

  // 2. Grab all blog posts from the API
  const posts = await getAllPosts()
  const latestPost = posts.sort(
    (a, b) =>
      new Date(b.publishedAt).getTime() - new Date(a.publishedAt).getTime()
  )[0]

  // 3. Make an RSS feed from the posts data
  // 4. Respond with an XML encoded response
  return mwResponse
}
```

An RSS feed is fairly simple in its specification so we could have built the content ourselves but this is Javascript so we'll find an npm package that does this for us:

```ts
import { Feed } from 'feed'
import { getAllPosts } from './util'

export async function middleware(
  req: MiddlewareRequest,
  mwResponse: MiddlewareResponse
) {
  // 1. Intercept if `rss.xml`
  const url = new URL(req.url)
  if (url.pathname !== '/rss.xml') {
    return mwResponse
  }

  // 2. Grab all blog posts from the API
  const posts = await getAllPosts()
  const latestPost = posts.sort(
    (a, b) =>
      new Date(b.publishedAt).getTime() - new Date(a.publishedAt).getTime()
  )[0]

  // 3. Make an RSS feed from the posts data
  const feed = new Feed({
    title: 'RedwoodJS',
    description: 'Redwood is the full-stack JavaScript application framework.',
    id: process.env.DEPLOY_URL,
    link: process.env.DEPLOY_URL,
    language: 'en',
    favicon: `${process.env.DEPLOY_URL}/favicon.png`,
    image: `${process.env.DEPLOY_URL}/favicon.png`,
    copyright: 'All rights reserved 2024',
    updated: new Date(latestPost.publishedAt),
    generator: 'RedwoodJS: RSS Middleware',
  })
  for (const post of posts) {
    const title = post.seo?.title || post.title
    const description = post.seo?.description || post.brief
    const image = post.ogMetaData?.image || post.coverImage?.url
    const link = process.env.DEPLOY_URL + '/blog/' + post.slug

    feed.addItem({
      title,
      link,
      date: new Date(post.publishedAt),
      description,
      published: new Date(post.publishedAt),
      id: link,
      image,
    })
  }

  // 4. Respond with an XML encoded response
  return mwResponse
}
```

Finally, we just set the body of our response and return:

```ts
import { Feed } from 'feed'
import { getAllPosts } from './util'

export async function middleware(
  req: MiddlewareRequest,
  mwResponse: MiddlewareResponse
) {
  // 1. Intercept if `rss.xml`
  const url = new URL(req.url)
  if (url.pathname !== '/rss.xml') {
    return mwResponse
  }

  // 2. Grab all blog posts from the API
  const posts = await getAllPosts()
  const latestPost = posts.sort(
    (a, b) =>
      new Date(b.publishedAt).getTime() - new Date(a.publishedAt).getTime()
  )[0]

  // 3. Make an RSS feed from the posts data
  const feed = new Feed({
    title: 'RedwoodJS',
    description: 'Redwood is the full-stack JavaScript application framework.',
    id: process.env.DEPLOY_URL,
    link: process.env.DEPLOY_URL,
    language: 'en',
    favicon: `${process.env.DEPLOY_URL}/favicon.png`,
    image: `${process.env.DEPLOY_URL}/favicon.png`,
    copyright: 'All rights reserved 2024',
    updated: new Date(latestPost.publishedAt),
    generator: 'RedwoodJS: RSS Middleware',
  })
  for (const post of posts) {
    const title = post.seo?.title || post.title
    const description = post.seo?.description || post.brief
    const image = post.ogMetaData?.image || post.coverImage?.url
    const link = process.env.DEPLOY_URL + '/blog/' + post.slug

    feed.addItem({
      title,
      link,
      date: new Date(post.publishedAt),
      description,
      published: new Date(post.publishedAt),
      id: link,
      image,
    })
  }

  // 4. Respond with an XML encoded response
  mwResponse.headers.set('Content-Type', 'application/xml')
  mwResponse.body = feed.rss2()

  return mwResponse
}
```

Voila! In about 100 lines of code, we intercepted the request and replied with a custom RSS feed. For those wondering the `getAllPosts` function that isn't shown here performs a simple fetch to the standard Redwood GraphQL endpoint to get the blog data as JSON.

The implementation of the sitemap is very similar so but all the code for our website is of course publically available on [GitHub](https://github.com/redwoodjs/bighorn-website) if you ever want to take a look further.

**What's next?**

We enjoy finding ways to use our own features. We find it's the best way to learn what needs improving. We'll likely realize there are more existing problems that middleware can help us solve more elegantly. We also hope Redwood users can come up with fun and exciting middlewares to share with everyone on our [forum](https://community.redwoodjs.com/) or discord!