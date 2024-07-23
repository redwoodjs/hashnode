---
title: "Middleware in RedwoodJS"
seoTitle: "Middleware in RedwoodJS"
datePublished: Fri May 10 2024 18:42:33 GMT+0000 (Coordinated Universal Time)
cuid: clw10yfp500080al22pxvfa8i
slug: middleware-in-redwoodjs
tags: middleware, redwoodjs

---

# An Introduction to Middleware

**What is middleware?**

It's a function that runs *before* your request is routed and rendered – giving you the ability to:

a) Intercept and modify the response

b) Enrich the request “context” (e.g. add auth details)

c) Enrich the response (add extra headers, cookies, etc.)

```typescript
const myMiddleware = (
  req: MiddlewareRequest,
  res: MiddlewareResponse,
  options: MiddlewareInvokeOptions,
) => {
  // 1. You can "intercept", i.e. skip React rendering
  return new MiddlewareResponse('Thou shall not pass')

  // 2. You can just passthrough the response. (Not shown here, but
  // you're of course still free to do something with the request)
  return res

  // 3. You can also "enrich" the response i.e. perform React
  // render but also add cookies, headers, etc
  res.headers.set('Access-Control-Allow-Origin', '*')
  return res
}
```

Middleware can also be a Class, with the invoke property defining the functionality

```typescript
class MyClassMiddleware implements MiddlewareClass {
   // ...
   async invoke(
    req: MiddlewareRequest,
    res: MiddlewareResponse,
    invokeOptions: MiddlewareInvokeOptions,
  ) {
     // ...
  }
 }
```

Note that this is only available in experimental features in Redwood, where you have streaming-ssr setup, and will be made stable in the Bighorn release.

First upgrade to Redwood canary

```bash
yarn rw upgrade -t canary
```

Then setup streaming-ssr

```bash
yarn rw experimental setup-streaming-ssr -f
```

I go through how this works in this screencast, and break down some of the concepts and how a request will flow through middleware. We'll also do a breakdown of the dbAuth middleware, to see how we can use these concepts in practice.

%[https://youtu.be/PFmJxzBtlfk] 

## Summary of concepts in the video

### **How do I add middleware?**

By registering the middleware in `entry.server.{tsx,jsx}`

```typescript
import {
  createDbAuthMiddleware
} from '@redwoodjs/auth-dbauth-middleware'

// 👇 you return an array of middleware from this function
export const registerMiddleware = () => {
  const dbAuthMiddleware = createDbAuthMiddleware({/*..*/})
  
  return [dbAuthMiddleware]
}

export const ServerEntry ({ css, meta }) => {
  return (
    <Document css={css} meta={meta}>
      <App />
      // ...
```

See examples:

* [Registering dbAuth middleware](https://github.com/redwoodjs/redwood/tree/main/packages/auth-providers/dbAuth/middleware)
    
* [Registering Supabase auth middleware](https://github.com/redwoodjs/redwood/tree/main/packages/auth-providers/supabase/middleware)
    

### MiddlewareRequest

The middleware request object builds on top of the [Fetch API Request](https://developer.mozilla.org/en-US/docs/Web/API/Request) – giving you access to all the HTTP level details in the request.

A few examples:

```typescript
const myMiddleware = async (
  req: MiddlewareRequest,
  res: MiddlewareResponse,
) => {
  // Same as Fetch API Request
  console.log(req.url)
  console.log(await req.json())
  console.log(req.headers.get('Content-Type'))

  // Extra helpers
  console.log(req.cookies.get('session'))
  req.serverAuthContext.set(res)
}
```

### MiddlewareResponse

Similarly, the MiddlewareResponse object builds on top of the [Fetch API Response](https://developer.mozilla.org/en-US/docs/Web/API/Response). Note that in a Fetch API Response, properties such as headers and body are read-only, but not in a `MiddlewareResponse`!

```typescript
const myMiddleware = async (
  req: MiddlewareRequest,
  res: MiddlewareResponse,
) => {
  res.headers.set('Handled-By-Middleware', 'true')

  // MiddlewareResponse has a cookie jar!
  res.cookies.unset('session')
  return res
}
```

### Options

The third parameter passed to middleware when they’re invoked is appropriately named `MiddlewareInvokeOptions` . While most middleware may not use this parameter, it can certainly come in useful when you need more details. See the comments below for what the options are

```typescript
export type MiddlewareInvokeOptions = {
  // if the request matched a Route defined in Routes.tsx
  route?: RWRouteManifestItem
  // any css files that'll be included
  cssPaths?: Array<string> 
  // parameters e.g. /blogPost/:id would give you the value of id
  params?: Record<string, unknown>
  // For dev only, access to the vite instance
  viteDevServer?: ViteDevServer 
}
```

### Chaining and Middleware Routing

Middleware can be chained i.e. executed one after another.

```typescript
export const registerMiddleware = async () => {
  return [
    globalMiddleware, // equivalent to [globalMiddleware, '*']
    [aboutPageMiddleware, '/about'],
    [productPageMw, '/product/:id'],
  ]
}
```

The second parameter of the tuple uses [find-my-way](https://github.com/delvedor/find-my-way) Route patterns. If you've used Fastify you'll be familiar with the syntax.

## Gotchas!

Some interesting things to point out, as you familiarize yourself with these concepts

#### **Intercepts**

Sometimes you will want to intercept a request, and avoid rendering the page in React.

If you set a body, it counts as an intercept – so it’ll skip React rendering (whose job is to set the body!)

This is best illustrated in [OG image generation middleware](https://github.com/redwoodjs/redwood/blob/main/packages/ogimage-gen/src/OgImageMiddleware.ts), where we intercept the request, and instead of rendering HTML, we render an image

```typescript
async invoke(
  req: MiddlewareRequest,
  mwResponse: MiddlewareResponse,
  invokeOptions: MiddlewareInvokeOptions,
) {
  // ....
  mwResponse.headers.append(
    'Content-Type',
    mime.lookup(extension)
  )
    
  mwResponse.body = image

  return mwResponse
}
```

**Cookie delete vs unset**

When setting or unsetting a cookie on a middleware response, remember that this code runs on the *server,* and *before* rendering in React.

```typescript
// Will expire the session cookie. This is what you want 99% of
// the time!
res.cookies.unset('session')

// Removes the cookie from the cookieJar.
// Only useful if you want to prevent a cookie being set/expired
res.cookies.clear('session')
```

**Middleware Routing order of precedence**

```typescript
export const registerMiddleware = async () => {
  return [
    [globalMiddleware, '*'],
    [aboutPageMiddleware, '/about'],
  ]
}
```

In the above example, it would be natural to assume that `globalMiddleware` would run in *addition* to `aboutPageMiddleware` for the `/about` route – but in reality only the aboutPage middleware will be executed when visiting `/about` .

This has to do with routing precedence. The middleware will be matched in the following order:

1. static
    
2. parametric node with static ending
    
3. parametric(regex)/multi-parametric
    
4. parametric
    
5. wildcard
    

In order to always run the `globalMiddleware` no matter what route they visit you will need to specify it twice. Both for `*`, which will make sure it's ran for all routes *not* matched by any other middleware, and then also for `/about` so that it's run for that route too. (And if you add middleware specific to any other route in the future, you'll need to remember to also explicitly add `globalMiddleware` for that route.)

```typescript
export const registerMiddleware = async () => {
 return [
  [globalMiddleware, '*'],
  [globalMiddleware, '/about'], 
  [aboutPageMiddleware, '/about'],
 ]
}
```

This is something we are considering changing, so if you have suggestions on what you would like to see in these APIs, we're all ears!

## The dbAuth Middleware

I also have a practical example of how these middleware concepts are applied. Check out the breakdown of the dbAuth middleware and how authentication works with server rendering in this video:

%[https://youtu.be/DrBe_uc31No]