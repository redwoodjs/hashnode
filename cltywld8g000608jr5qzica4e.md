---
title: "React Server Components Now in RedwoodJS"
seoTitle: "RedwoodJS latest canary includes React Server Components support"
seoDescription: "Redwood's preview of React Server Component support is now available! Follow this walkthrough to find out what's new and how to covert an app from GraphQL."
datePublished: Sun Mar 24 2024 07:00:00 GMT+0000 (Coordinated Universal Time)
cuid: cltywld8g000608jr5qzica4e
slug: rsc-now-in-redwoodjs
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1711397796556/49d05754-6e7c-4f2e-99cf-07ada6d73544.png
tags: reactjs, rsc, bighorn, canary

---

Welcome to the preview of RedwoodJS Bighorn! You may not have realized it, but you’ve been living in the Arapahoe epoch since Redwood v1.0. Bighorn is the next epoch, and will bring a massive change to how apps are built with Redwood: [React Server Components](https://www.joshwcomeau.com/react/server-components/) (RSC) and [Server-side Rendering](https://community.redwoodjs.com/t/react-streaming-and-server-side-rendering-ssr/5052) (SSR)!

Core team lead Tobbe [announced](https://community.redwoodjs.com/t/react-server-components-rsc/5081) the first RSC build back in July of 2023 and the team has been hard at work on it ever since. We’re ready to show everyone what we’ve been working on and invite you to try it out yourself and give us feedback! This document walks you through what’s different about a Redwood RSC app versus a “classic” Redwood app built with GraphQL. We’re going to review an example app that we’ll convert from GraphQL to RSC, demonstrating the surprisingly small number of steps required to do so.

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text">Redwood’s epochs are named after <a target="_blank" rel="noopener noreferrer nofollow" href="https://en.wikipedia.org/wiki/List_of_national_forests_of_the_United_States" style="pointer-events: none">National Forests</a> in the United States!</div>
</div>

You can also watch a video covering how RSC works in Redwood and how to convert an existing GraphQL app:

%[https://youtu.be/5IZv3khsx0o] 

## Why RSC?

Redwood has been all-in on GraphQL from the start—why the change in direction? One of Redwood’s prime directives has been to go from idea to startup as quickly as possible. With that in mind, we've come to find that starting from the ground up with GraphQL incurs a lot of mental and logistical overhead from day one, sometimes before you even really know what you’re building. GraphQL may not be the best choice for starting a greenfield application when you’re trying to stay nimble and make changes to large parts of your app quickly.

Other concerns that we’ve experienced ourselves, and received feedback about, when it comes to GraphQL:

* Developer productivity: GraphQL is yet another technology that a developer needs to learn to become a productive contributor to your codebase. While GraphQL is earning more marketshare every day, it’s far from ubiquitous at this point in time. Requiring the full GraphQL stack from the start also adds a lot of overhead for even the smallest change to your data, requiring changes to the schema, GraphQL SDL, resolvers, types, and maybe more depending on the structure of your components.
    
* Security: you need to seriously think about how to secure access to your data when you expose a GraphQL API. Not everyone is willing to take on that responsibility, especially if they don’t plan on providing a public-facing API. Redwood is as secure as possible by default, but we can’t stop you from exposing potentially secret data if you’re really determined to do so, or if you simply overlook the fact that fields will be available to the outside world if you create a simple SDL that wraps an entire model (something that’s very convenient to do in development, but easy to forget about and becomes a liability in production).
    
* Performance: it is extremely easy to fall into the trap of the [N+1 problem](https://hygraph.com/blog/graphql-n-1-problem), [cyclic queries and depth limits](https://escape.tech/blog/cyclic-queries-and-depth-limit/) on even the most trivial of GraphQL implementations. If you don’t explicitly require a public API for your application then why incur the performance penalty (and mental overhead) for an entire class of problems that can instead be avoided entirely by removing GraphQL from the stack?
    

We took a long, hard look at our original requirement that all apps be all-in on GraphQL. After careful consideration, we decided that for the reasons stated above we now think that the benefits of switching to RSC-by-default were just too good *not* to make it the default in the framework. One of the goals of Redwood is to maximize developer productivity and we feel RSC is yet another huge step in that direction.

## What’s the future for GraphQL in Redwood?

We still love GraphQL, and it will always be fully supported in Redwood. Going forward we’ll provide a simple setup command if you’d prefer your app to use GraphQL from the start, or if you’ve reached a point in development where you want to start providing an API to others (you can have both RSC and GraphQL working together in the same app). That setup command will be as simple as:

```bash
yarn rw setup graphql
```

That’s subject to change, but you get the idea. That will result in the necessary modifications to your codebase to enable GraphQL and everything along with it (directives, service-to-resolver mappings, etc.).

## How React Server Components Work

RSC presents a new way to think about React-based web applications. First, let’s take a look at how requests and data flow through a traditional Redwood app:

1. A browser makes a request to your server, `https://example.com/photos/123/edit`
    
2. The server returns the JS files necessary to get React started.
    
3. The browser starts your app running, rendering any pages and components it can. When it comes across a Cell, which needs data from the server, React shows the `<Loading>` component(s) from the Cell and makes one or more GraphQL calls (defined in `QUERY`) to the server to get that data.
    
4. The server receives the GraphQL request, invokes a resolver (Service), usually getting data from your database, but could also be from a third-party API, the file system, etc.
    
5. The data is formatted for a GraphQL response, and returned to the browser.
    
6. React replaces the `<Loading>` component with `<Success>` component, passing the data returned from GraphQL as props.
    

Here’s a sequence diagram showing the whole flow:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1710882408347/e3e66e98-e771-4878-8271-2cf17c2b1f02.png align="center")

Here’s the definition a typical Cell in a classic Redwood app that demonstrates the full flow above, where server data is needed before the result can be displayed:

```jsx
import { Link, routes } from '@redwoodjs/web'

// GraphQL query to get Product data
export const QUERY = gql`
  query GetProducts {
    products {
      id
      name
    }
  }
`

// Displayed instantly and until we receive a GraphQL response or erro
export const Loading = () => {
  return <div>Loading products...</div>
}

// Replaces <Loading> if there's a GraphQL error
export const Failure = (error) => {
  return <div>Something went wrong: {error.message}</div>
}

// Replaces <Loading> if `products` is an empty array
export const Empty = () => {
  return <div>No products found</div>
}

// Replaces <Loading> if everything goes right and there's at least 1 product
export const Success = ({ products }) => {
  return (
    <ul>
      {products.map((product) => (
        <li key={product.id}>
          <Link to={routes.product({id: product.id})}>
            {product.name}
          </Link>
        </li>
      )}
    </ul>
  )
}
```

Now let’s look at rendering the same Cell in RSC. The first major change is that there is no “api” server any longer, there’s only a single “web” server. It will return the initial JS needed for React, as well as handle the requests for RSC components *and* data:

1. A browser makes a request to your server, `https://example.com/photos/123/edit`
    
2. The server returns the JS files necessary to get React started, including any wrapping Layouts.
    
3. The browser starts your app running, rendering up to the beginning of the first Page. It then makes a request to the server for the page content and anything inside (including Cells and their data).
    
4. The server receives the RSC request and starts rendering the page, gathering any data required in any Server Cell(s) in order for the `<Success>` component to be rendered completely.
    
5. The result is JSX + Data and formatted into React’s **Flight** format for returning to the browser.
    
6. The browser receives the Flight info and renders it into the DOM.
    

Here’s the diagram showing the new data flow:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1710882506793/15a9a07f-6136-4c8a-b14a-51a618ea6868.png align="center")

We’ve removed a column from this diagram by eliminating an entire server! There’s also no longer a need to marshal data to and from GraphQL, and we get to write our server code right beside our client code, continuing Redwood’s embrace of [single file components](https://www.swyx.io/react-sfcs-here).

Here is the same cell as the GraphQL implementation, but transformed into a **Server Cell** that can be rendered by RSC:

```jsx
import { Link, routes } from '@redwoodjs/web'
import { db } from 'web/src/lib/db'

// Get Product data
export const data = () => {
  return { products: db.products.findMany() }
}

// Displayed instantly and until we receive a RSC response
export const Loading = () => {
  return <div>Loading products...</div>
}

// Replaces <Loading> if there's an error
export const Failure = (error) => {
  return <div>Something went wrong: {error.message}</div>
}

// Replaces <Loading> if `products` is an empty array
export const Empty = () => {
  return <div>No products found</div>
}

// Replaces <Loading> if everything goes right and there's at least 1 product
export const Success = ({ products }) => {
  return (
    <ul>
      {products.map((product) => (
        <li key={product.id}>
          <Link to={routes.product({ id: product.id })}>
            {product.name}
          </Link>
        </li>
      )}
    </ul>
  )
}
```

The only thing different here is that `QUERY` has been replaced by `data()` . Now you can do things in `data()` that only work in a server context: file system access, database calls, and more. In addition, the result of `data()` is automatically `await`ed so that you don’t need to care if it returns a Promise or not (like the database call in the code sample above).

## What is the RSC “Flight” Format?

The most common description you'll come across when reading about RSC is that it “renders and returns HTML to the browser,” but that’s not entirely accurate. A new type of document is returned in the body of the response: plain text that React calls **Flight**. It’s sort of JSX turned into JSON, which React (on the client) knows how to then turn back into HTML and update the DOM. Here’s an example chunk (this is the actual body in the response of a request to the RSC endpoint in the latest canary of Redwood):

```jsx
1:"$Sreact.suspense"
0:["$","$1",null,{"fallback":["$","div",null,{"className":"flex h-screen items-center justify-center","children":["$","div",null,{"className":"text-center","children":[["$","h2",null,{"className":"mb-4 text-lg text-neutral-500","children":"Loading photos..."}],"$undefined"]}]}],"children":"$L2"}]
3:I["/assets/rsc-Slide.jsx-0-BJd1X8TY.mjs",["/assets/rsc-Slide.jsx-0-BJd1X8TY.mjs"],"default"]
2:["$","div",null,{"className":"","children":["$","ul",null,{"className":"flex flex-wrap justify-center","children":[["$","$L3","1",{"photo":{"id":1,"filename":"20150225-DSF4486-thumb.webp"}}],["$","$L3","2",{"photo":{"id":2,"filename":"20151231-DSF0293-thumb.webp"}}],["$","$L3","3",{"photo":{"id":3,"filename":"20151231-DSF0401-thumb.webp"}}],["$","$L3","4",{"photo":{"id":4,"filename":"20160603-DSF2047-thumb.webp"}}],["$","$L3","5",{"photo":{"id":5,"filename":"20160723-DSF2210-thumb.webp"}}],["$","$L3","6",{"photo":{"id":6,"filename":"20161009-DSF8597-thumb.webp"}}],["$","$L3","7",{"photo":{"id":7,"filename":"20170721-DSCF2449-thumb.webp"}}],["$","$L3","8",{"photo":{"id":8,"filename":"20170721-DSCF2492-thumb.webp"}}],["$","$L3","9",{"photo":{"id":9,"filename":"20220328-DSCF1003-thumb.webp"}}],["$","$L3","10",{"photo":{"id":10,"filename":"20220328-IMG_5414-thumb.webp"}}],["$","$L3","11",{"photo":{"id":11,"filename":"20220330-DSCF1369-thumb.webp"}}],["$","$L3","12",{"photo":{"id":12,"filename":"20220330-IMG_5512-thumb.webp"}}],["$","$L3","13",{"photo":{"id":13,"filename":"20220331-DSCF1484-thumb.webp"}}],["$","$L3","14",{"photo":{"id":14,"filename":"20240224-DSCF2814-thumb.webp"}}]]}]}]
```

You can see there are references to a couple of JS files in there (the line starting with `3`), letting the client knows it will need to request a couple of additional component packs, and then the actual HTML that will be rendered, along with the data that’s passed as props (blocks labeled `0` and `2`). Here’s a formatted breakdown of the `0` block:

```jsx
[
  "$",
  "$1",
  null,
  {
    "fallback":[
      "$",
      "div",
      null,
      {
        "className":"flex h-screen items-center justify-center",
        "children":[
          "$",
          "div",
          null,
          {
            "className":"text-center",
            "children":[
              [
                "$",
                "h2",
                null,
                {  
                  "className":"mb-4 text-lg text-neutral-500",
                  "children":"Loading photo..."
                }
              ],
              "$undefined"
            ]
          }
        ]
      }
    ],
    "children":"$L2"
  }
]
```

That's the content of `<PhotosCell>` which starts out rendering the `<Loading>` component as you can see in the deepest section of the tree.

Here is the formatted contents of block `2` :

```javascript
[
  "$",
  "div",
  null,
  {
    "className":"",
    "children":[
      "$",
      "ul",
      null,
      {
        "className":"flex flex-wrap justify-center",
        "children":[
          [
            "$",
            "$L3",
            "1",
            {
              "photo":{
                "id":1,
                "filename":"20150225-DSF4486-thumb.webp"
              }
            }
          ],
          [
            "$",
            "$L3",
            "2",
            {
              "photo":{
                "id":2,
                "filename":"20151231-DSF0293-thumb.webp"
              }
            }
          ],
          // ...remaining photos...
        ]
      }
    ]
  }
]
```

It’s similar to the [React.createElement()](https://react.dev/reference/react/createElement) syntax, but not exactly the same. This is the render tree for the `<Success>` component of the cell, denoting that another child component is needed `$L3` which is the `<Slide>` component (the source of which is denoted in block `3` of the full flight response). All of this is then placed in the `$L2` placeholder in block `0`.

## Using RSC in Redwood

What does a Redwood app built on RSC look like? Let’s find out!

Below we’ll be discussing the changes that were needed to go from [this app](https://github.com/cannikin/cambium-classic) built with classic RedwoodJS (using GraphQL) to [this app](https://github.com/cannikin/cambium-rsc) which is visually/functionally the same, but is now built with RSC. Feel free to clone one or both and get them running locally. The classic app you can run with `yarn rw dev` but the RSC version will need to be run with `yarn rw build && yarn rw serve` (more details in the “RSC Gotchas” section below).

A couple of requirements before we get started with RSC:

* Node v20 or greater is required (can be installed via [nvm](https://github.com/nvm-sh/nvm))
    
* Yarn 4 is required (can be enabled via [corepack](https://yarnpkg.com/corepack))
    

Assuming you have those you can clone the RSC repo, install dependencies, and start the server:

```bash
git clone https://github.com/cannikin/cambium-rsc.git
cd cambium-rsc
yarn install
yarn rw build
yarn rw serve
```

### The Example App

**Cambium** is a simple light-table/photo-editing app. You choose a photo, tweak its brightness, contrast, etc. and then share a URL. Anyone following that URL can see your photos with the edits you made, along side some nerdy photographer data beneath it (shutter speed, aperture, etc):

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1710882919503/cd3a3fcf-8346-4499-8123-b785c61339f9.png align="center")

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1710882938524/c0b924c4-e6bb-4161-9377-3af71f506a6d.png align="center")

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1710882951911/da5c2765-fbae-4c30-a904-e41882e9ada1.png align="center")

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text">The “cambium” is the part of a tree just beneath the bark where new cell growth occurs! We love trees.</div>
</div>

The following sections detail the few changes that were made to the GraphQL version of the app to get it running on Redwood RSC. This entire process took about two hours, and only because it was our first attempt at doing it! You’ll be pleasantly surprised how little code actually requires a change. *Your app absolutely does not need a complete re-write to move from GraphQL to RSC.*

### Cells

In order to turn a Cell into a Server Cell you only need to change one function. You can remove the `QUERY` completely, and export a function named `data()` instead. Whatever is returned from this function will be given to the `<Success>` component.

```jsx
import { photos } from 'src/services/photos'

export const data = () => {
  return { photos: photos() }
}
```

To keep things simple, Cambium doesn’t read image data from a database, it gets it from the files directly: we read the images from disk (something you can only do on a server, remember) and then parse out the EXIF details (camera make and model, shutter speed, etc.) and return them to the client.

The `<Success>` component is identical to the GraphQL implementation: it destructures a property `photos` which it got as a result of the `data()` call and then uses that in the component implementation inside.

You’ll see that `data()` is using a `photos()` function exported from a plain Redwood Service module. We’re carrying forward the concept of Services from classic Redwood and encapsulating our logic for what it means to return a set of photos to one place. Notice that in this app the services are co-located in the web side along with the components, and in fact there's no `api` directory at all!

```bash
└── web
    └── src
        ├── components
        ├── layouts
        ├── pages
        └── services
            └── photos
                └── photos.js
```

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text">We’re still deciding where to put things like <code>functions</code>, <code>services</code> and your <code>prisma.schema</code> file in the final Bighorn release.</div>
</div>

The service itself just loops through all of the photos in `web/public/photos` and returns them as an array, giving each a fake `id` so we can view and edit a single photo later:

```jsx
const getFiles = ({ thumb }) => {
  return fs
    .readdirSync(photosPath)
    .filter((file) => fs.statSync(path.join(photosPath, file)).isFile())
    .filter((file) =>
      thumb ? file.includes('thumb') : !file.includes('thumb')
    )
}

// returns the collection of all available photos
export const photos = () => {
  return getFiles({ thumb: true }).map((filename, index) => {
    return {
      id: index + 1,
      filename,
    }
  })
}
```

### `'use client'`

For a very thorough explanation of the concepts behind RSC you should spend a few minutes reading [this article](https://www.joshwcomeau.com/react/server-components/), but here’s a great explanation of RSC and state:

> Server Components never re-render. They run once on the server to generate the UI. The rendered value is sent to the client and locked in place. As far as React is concerned, this output is immutable, and will never change (at least, not until something happens at the router level, like navigating to a new page).
> 
> This means that a big chunk of React's API is incompatible with Server Components. For example, we can't use state, because state can change, but Server Components can't re-render. And we can't use effects because effects only run after the render, on the client, and Server Components never make it to the client.

In the classic implementation of Cambium we use state to track what the values of your adjustments (brightness, contrast) are. How do we do this if the Cell is now rendered on the server?

The trick is that even if a component is rendered on the server, it can still have *children* which are rendered on the client. We let React know that a component should be rendered on the client by adding an explicit `'use client'` literal expression (also known as a Directive) at the top of the file. Note the quotes, meaning this is just a plain string as far as JS is concerned.

What this means for Cells is that if what is rendered in `<Success>` needs state, move that code to its own component, pass in any data as props, and include `'use client'` at the top. We’ve nicknamed this pattern the **Cell Success Child Component**. Here’s what the Cell looks like with this pattern:

```jsx
// web/src/pages/EditPhotoPage/EditPhotoCell/EditPhotoCell.jsx

import { photo } from 'src/services/photos'
import EditPhotoSuccess from './EditPhotoSuccess'

export const data = ({ id }) => {
  return { photo: photo({ id }) }
}

export const Success = ({ photo }) => {
  return <EditPhotoSuccess photo={photo} />
}
```

And the Cell Success Child Component (the real component is more complex than this, but you’ll get the idea):

```jsx
// web/src/EditPhotoPage/EditPhotoCell/EditPhotoSuccess/EditPhotoSuccess.jsx

'use client'

import { useState } from 'react'

const EditPhotoSuccess = ({ photo }) => {
   const [adjustments, setAdjustments] = useState(ADJUSTMENT_DEFAULTS)
  
  return (
    <>
      <img src={photo.filename} className={adjustmentsToCSS(adjustments)} />
      <Controls adjustments={adjustments} />
    </>
  )
}
```

<div data-node-type="callout">
<div data-node-type="callout-emoji">💡</div>
<div data-node-type="callout-text">We’re using a new file organization scheme here we’ve developed over the years since Redwood v1.0: if a component is <em>only</em> used by a single parent, it goes into its parent’s directory. So the <code>EditPhotoCell</code> lives inside the <code>EditPhotoPage</code> directory, and <code>EditPhotoSuccess</code> lives inside <code>EditPhotoPage/EditPhotoCell</code>. We’re debating whether to make this scheme the default recommendation in Redwood apps going forward. <a target="_blank" rel="noopener noreferrer" class="notion-link-token notion-focusable-token notion-enable-hover" href="https://github.com/redwoodjs/redwood/issues/10263" style="pointer-events: none">What do you think?</a></div>
</div>

What about all of the child components to this one, like `<Controls>`, do they need `'use client'` at the top as well? No, as long as the component itself doesn’t use state or other client-only functionality internally. React is smart and knows that when you `'use client'` that anything imported in that component *also* needs to be rendered on the client and does so automatically. (See the **Rendering Server Components Inside Client Components** section for more info.)

A side effect of this is that using those same child components in a server-rendered component will allow them to still be rendered on the server (remember, the children don’t contain `'use client'`). If a server component somewhere included `<Controls>` as a child, as long as the `adjustments` prop is still passed in then `<Controls>` will happily render on the server.

Here’s a super helpful graphic Tobbe created explaining how an example page flips back and forth between client and server rendering:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1710883821754/c3f16abf-6701-44ca-bfe1-53e5f3581879.png align="center")

The original drawing can be accessed [here](https://excalidraw.com/#json=r79hKtU7A9EiN2DndkSuM,7VkPX6EYJvcBqkVsjtG8pQ).

### Moving Services

There’s no more api side, so services need to live *somewhere*! For this example app we moved them to `web/src/services` but that’s not their final location. We’re currently thinking they should live at the same level as `web` itself, maybe in a `db` or `data` directory.

## RSC Gotchas

There are a couple of temporary workarounds that were required to convert Cambium to RSC, and you’ll need to keep them in mind if you start building with the Bighorn canary. These workarounds will not be required once Bighorn is official.

### No dev server

The dev server isn’t currently compatible with RSC so you’ll need to build and serve your app from scratch any time you make a code change:

```bash
yarn rw build
yarn rw serve

# one-liner
yarn rw build && yarn rw serve
```

### The routing `entries.ts` file

The Router itself remains unchanged, but we need to manually add entries to the `/web/src/entries.ts` file. Any route defined in the Router needs its own `case` in the `switch` statement:

```jsx
import { defineEntries } from '@redwoodjs/vite/entries'

export default defineEntries(
  // getEntry
  async (id) => {
    switch (id) {
      case 'EditPhotoPage':
        return import('./pages/EditPhotoPage/EditPhotoPage')
      case 'HomePage':
        return import('./pages/HomePage/HomePage')
      case 'PhotoPage':
        return import('./pages/PhotoPage/PhotoPage')
      default:
        return null
    }
  }
)
```

A future release will manage this file for you within the framework and it will be gone from the app's codebase completely.

### Replacing the current URL

In a classic Redwood app you can update the current URL by passing a `{ replace: true }` option as the last argument to `navigate()`. The app will *not* make any new requests to the server. It simply replaces the URL in the navigation history:

```jsx
navigate(routes.editPhoto({ id: 123, brightness: 1.1 }, { replace: true }))
```

In the current canary, making this call *will* trigger a call to the server, which will result in your page being redrawn. In Cambium RSC this would have resulted in the entire page going blank and then fading in again (transitions are re-triggered) every time you move an adjustment slider. To prevent this, the adjustments being stored in the URL use the hash instead of the query string:

```plaintext
// classic
http://localhost:8910/photos/123/edit?brightness=0.15&contrast=2

// rsc
http://localhost:8910/photos/123/edit#brightness=0.15&contrast=2
```

### **No**`__dirname` or `__filename`

Although components and services are rendered on the server, you do not currently have access to core node vars like `__dirname` and `__filename`. Node 20 provides [equivalent properties](https://nodejs.org/api/esm.html#importmeta)`import.meta.dirname` and `import.meta.filename` but only for ESM modules. Redwood is still using CJS (we’re in the process of converting over). In the meantime you can use `path.resolve()` which will resolve a full file path relative to where the code is executed (not the original file path of the file itself). You may need to experiment with the path, but eventually you’ll get it!

### No &lt;Metadata&gt; in Server Components

The `<Metadata>` component lets you easily add SEO-friendly `<meta>` tags to your pages' `<head>`. However, `<Metadata>` uses React Context internally, and context uses state, and state can only be used in in client components. So when adding `<Metadata>` make sure its in a component with `'use client'` at the top.

### No mutations, aka Server Actions

The current canary of Redwood Bighorn is effectively “read only” as we haven’t finished implementing [Server Actions](https://vercel.com/blog/understanding-react-server-components#server-actions-react%E2%80%99s-first-steps-into-mutability) yet, which allow for POSTing data to a server to signal you want to make a change. Coming very soon!

### No Server Cells in Layouts

You may come across some unexpected behavior if you try to reference a Server Cell as a child component in your Layout. In one instance, it would render correctly as soon as the initial page loaded, but when the RSC response returned from the server it would be wiped out and replaced with an error message. Depending on what you're doing in the `data()` function you may be okay, but things like file system access via `fs` or `path` will error. We're working on this one!

### Rendering Server Components inside Client Components

Remember when we said that once you `'use client'` that child components are rendered on the client as well? That's not really the complete picture: there is a way to have server-rendered components as children of client-rendered one.

Imagine a theme picker where you can choose if a certain page of your app should display in light or dark mode. The state that's stored needs to be pretty close to the root of the page since it affects all children. In our case, maybe we'll be setting a CSS class in a wrapper around the other components (in this example pretend the actual switcher is implemented somewhere else, this is just displaying the result of your theme choice):

```javascript
import { useState } from 'react'
import Header from './Header'
import MainContent from './MainContent'

const StyledPage = () => {
  const [theme, setTheme] = useState('light')

  return (
    <main className={theme}>
      <Header />
      <MainContent />
    </main>
  )
}

export default StyledPage
```

Based on what we've just learned about RSC, in order to get this page to render going forward we'd need to add `'use client'` to the top and now both `<Header>` and `<MainContent>` are going to end up rendered in the browser as well. That kind of defeats the purpose of RSC, if even components that don't use state and now forced to render on the client again.

If we move the theme-related content into its own component we can pass `<Header>` and `<MainContent>` as *children* which will allow them to still be rendered on the server.

First, the new `<Theme>` component (it's now the only thing that needs `'use client'` at the top):

```javascript
'use client'

import { useState } from 'react'

const Theme = ({ children }) => {
  const [theme, setTheme] = useState('light')

  return (
    <main className={theme}>
      {children}
    </main>
  )
}

export default Theme
```

And now `<StyledPage>` can use that, setting the others as children:

```javascript
import { useState } from 'react'
import Theme from './Theme'
import Header from './Header'
import MainContent from './MainContent'

const StyledPage = () => {
  return (
    <Theme>
      <Header />
      <MainContent />
    </Theme>
  )
}

export default StyledPage
```

And we're back to the majority of our page being server-rendered! Why does this work? To quote Josh Comeau and [his explanation](https://www.joshwcomeau.com/react/server-components/#workarounds-7):

> `<Theme>`, a Client Component, is a parent to `<Header>` and `<MainContent>`. Either way, it's still higher in the tree, right?
> 
> When it comes to client boundaries, though, the parent/child relationship doesn't matter. `<StyledPage>` is the one importing and rendering `<Header>` and `<MainContent>`. This means that `<StyledPage>` decides what the props are for these components.
> 
> Remember, the problem we're trying to solve is that Server Components can't re-render, and so they can't be given new values for any of their props. With this new setup, `<StyledPage>` decides what the props are for `<Header>` and `<MainContent>`, and since `<StyledPage>` is a Server Component, there's no problem.

Dan Abramov explains:

> We shouldn’t think of “use client” as marking the child tree as client. It’s really about making child *imports* client.
> 
> The mental model about “use client” is that it’s the starting point of bundling. Like an entry point. And then anything *imported* from that will also get into the bundle and run on the client side.

So as long as the client component (`<Theme>` in this case) isn't importing those other components, they're free to still render on the server.

## Creating Your Own App

You can also watch a video covering creating your own app from scratch:

%[https://youtu.be/PW0ulmCpWn4] 

### Getting Started

You can start playing with RSC using the latest canary build of Redwood

```bash
npx -y create-redwood-app@canary -y ~/rsc_app
cd ~/rsc_app
```

You’ll then need to enable a couple of experimental features:

```bash
yarn rw experimental setup-streaming-ssr -f
yarn rw experimental setup-rsc
```

Finally, build and serve:

```bash
yarn rw build
yarn rw serve
```

As part of the `setup-rsc` command a barebones RSC app is created for you, demonstrating a client component rendering inside of a server component:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1710884294596/86881d65-a9b6-4899-b60b-fb124f991081.png align="center")

You can start your own app from this codebase by:

* Replacing the couple of pre-created pages with your own
    
* Editing `web/src/Routes.ts` and `web/src/entries.ts`
    

Note that the generators have not been updated for RSC so anything generated with them may require manual changes to conform to RSC style.

## Need Help?

You can ask questions in [this community thread](https://community.redwoodjs.com/t/react-server-components-rsc/5081) and we’ll try to answer them the best we can. If you think you’ve found a bug please post it here before opening a GitHub Issue—it could just be something we haven’t documented yet!

## What’s Next?

We’ve almost got **SSR** working with RSC and once that’s ready the initial page load from the server will contain pre-rendered HTML, without the need to have React start up and render before you see anything (this improves the speed to [First Contentful Paint](https://web.dev/articles/fcp)). If the server detects that the request was made by a search engine bot the response will wait for the entire page to be rendered, cells and all, to be sure that everything is present for the bot to digest as quickly as possible.

**Server Actions** are also in-progress and are the mechanism that your app will to use to save data (like forms) to the server/database.

**Auth** has to be modified to work in an RSC world, and it’s almost ready. The state of the user’s session used to be communicated via GraphQL but obviously that has to change now.

**Generators** still need to be updated to detect whether your app is RSC or GraphQL and generate the proper template.

As mentioned above, we have some proposed [**file directory structure**](https://github.com/redwoodjs/redwood/issues/10263) changes which, when finalized, will require changes to the generators and documentation.

And finally, we need to make a pass through most of our **docs** to update them to cover any new/modified behavior thanks to RSC.

If any of this sounds like something *you’d* like to work on, [get in touch](https://discord.com/invite/redwoodjs) and let us know! We always love expanding the community and getting more folks involved in Redwood development, and everyone has a path to becoming a core team member!

## Finally

React Server Components are looking like they’re going to be the future of React, and as always Redwood is right on the bleeding edge. We’re all very excited to bring this next epoch of Redwood to you, and we know it’ll be worth the wait!