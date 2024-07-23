---
title: "Building a new docs site with RSC"
datePublished: Tue Apr 23 2024 23:22:05 GMT+0000 (Coordinated Universal Time)
cuid: clvd0gg0u000808mlg5vu664p
slug: building-a-new-docs-site-with-rsc
tags: documentation, rsc, example-app

---

As big fans of the [dog fooding principle](https://en.wikipedia.org/wiki/Eating_your_own_dog_food), we want to put RSC through its paces by using it to build more than just a demo app. As Redwood's support for RSC becomes more mature we are going to need to document it with the same attention we pay to the existing Redwood docs. The [current docs site](https://redwoodjs.com/docs/index) is powered by [Docusaurus](https://docusaurus.io/). What if we built our own docs site to document RSC, using RSC? Docu-ception and docu-dog fooding rolled into one!

Enter [Docwood](https://github.com/redwoodjs/docwood), the codename for our RSC-powered documentation app.

## Requirements

The list of our requirements we came up the project:

1. Keep the docs in the existing [redwoodjs/redwood](https://github.com/redwoodjs/redwood) repo, right alongside the code. This makes it super simple to write docs at the same time as code and a single PR merges everything at once.
    
2. All docs are written in [Markdown](https://www.markdownguide.org/) or [MDX](https://mdxjs.com/).
    
3. The navigation structure and names of the docs should depend on the directory structure and content of the docs themselves (no separate config file to denote parent/child relationships or doc names).
    
4. The app should feature real-world usage of RSC.
    
5. Creating or modifying docs should not involve making any changes to the Docwood codebase itself.
    
6. Creating or modifying docs should not require a re-deploy of the Docwood app.
    

#4 was the most nebulous requirement. What does "real-world usage" mean? After some discussion about what we wanted to highlight with RSC, we decided on the following:

1. The docs site would be a real React app running in the browser (not just a statically generated site).
    
2. Viewing a doc meant invoking the RSC flow to make a request to a server, where the Markdown would be complied and returned on fly.
    
3. No pre-converted Markdown-&gt;HTML stored anywhere, no content stored in a database, no pre-processing of docs files at build time. However, Redwood's [service caching](https://redwoodjs.com/docs/services#caching) *could* be used, since we would still be performing a full RSC request and converting Markdown to HTML on the fly to populate the cache. (Service caching will be our recommended RSC performance-enhancement workflow—more delicious dog food!).
    

With those requirements in place, we got to work.

## create-redwood-app

If you start a new Redwood app from scratch today, doing so with the canary release will make RSC available via an experimental setup command (follow our guide at the end of the [RSC walkthrough blog post](https://redwoodjs.com/blog/rsc-now-in-redwoodjs) to learn more):

```bash
npx -y create-redwood-app@canary -y ~/docwood
cd ~/docwood
yarn rw experimental setup-streaming-ssr -f
yarn rw experimental setup-rsc
```

This starts you with a barebones RSC app to demonstrate usage, but a couple of minor edits gets us back to a blank canvas.

## Routing

Since all docs would be rendered on the fly we really only needed a single "container" page to display any particular doc. We used a [glob-type route parameter](https://redwoodjs.com/docs/router#glob-type) which would take any parts of the URL and put them into a single prop which we would then parse on the backend to determine what doc to actually render:

```javascript
<Route path="/docs/{path...}" page={DocsRendererPage} name="docs" />
```

Thanks to this route, everything is sent to the `<DocsRendererPage>` and it accepts a single prop `path` containing the full path for the doc (styling removed for clarity):

```javascript
const DocsRendererPage = ({ path }) => {
  return (
    <div>
      <aside>
        <DocsNavigationCell docPath={path} />
      </aside>
      <main>
        <DocsRendererCell docPath={path} />
      </main>
    </div>
  )
}
```

Some examples of how a URL maps to the `path` prop:

* `/docs/index` → `"index"`
    
* `/docs/quick-start` → `"quick-start"`
    
* `/docs/reference/deploy/baremetal` → `"reference/deploy/baremetal"`
    

We also needed one more route to catch the case of the URL looking like `/docs` where there were no additional path parts:

```javascript
<Route path="/docs" page={DocsRendererPage} name="docsHome" />
```

This means a request for `/docs` still renders the `<DocsRenderPage>` but `path` will be `undefined`. The `<DocsRendererCell>` will take care of figuring our what to do in that case!

This results in the following basic, mostly unstyled layout of a typical doc page:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1713207114582/f75d758a-eb91-4391-86c2-14cb4a41cb04.png align="center")

For now we've surrounded each component with a dashed box to help keep track of what is rendering where.

## Navigation

One of our requirements was to have the navigation built on the fly based on the directory structure of the underlying Markdown docs. This is handled by the `<DocsNavigationCell>` Server Cell which contains the new `data()` function (executed on the server) and the resulting `<Success>` component to render once `data()` is done:

```javascript
import { getDocumentTree } from 'src/lib/docs'

const DocLink = (link) => {
  return (
    <li key={link.link}>
      <a href={link.link}>
        {link.title}
      </a>
      {link.children.length > 0 && (
        <ul
          key={`${link.link}-sub`}
        >
          {link.children.map((child) => (
            <DocLink link={child.link} />
          ))}
        </ul>
      )}
    </li>
  )
}

export const data = async ({ docPath }) => {
  const subTree = await getDocumentTree(docPath)
  return { links: subTree }
}

export const Loading = () => {
  return <div>Loading nav...</div>
}

export const Failure = ({ error }) => {
  return <div>Error: {JSON.stringify(error.message)}</div>
}

export const Success = ({ links }) => {
  return (
    <nav>
      <ul>
        {links.map((link) => (
          <DocLink link={link} />
        })}
      </ul>
    </nav>
  )
}
```

`getDocumentTree()` is a little verbose to get into here, but you can see the [full function in the repo](https://github.com/redwoodjs/docwood/blob/main/web/src/lib/docs.ts). It looks through the docs on disk and builds a simple nested object containing the link to a doc, or a directory if there is one, containing any child docs inside.

The name of the doc is pulled from any [frontmatter](https://www.npmjs.com/package/front-matter) inside, but falls back to just the name of the file, converted from `dashed-case-name.md` to `Dashed Case Name`. Sorting is alphabetical, so if you want to force a different order to your files you can prefix the name with a number. For example, at the root of the docs directory, we have:

```bash
/docs
    ├── 01_quick-start.md
    ├── 02_tutorial
    ├── 03_reference
    ├── 04_how-to
    └── index.md
```

Which mirrors the structure of the existing docs site. By default we're only showing 2 levels of depth in the nav, but the component can display down to any depth.

## Displaying Documentation

For the docs page itself, we turn to the `<DocsRendererCell>` which contains a `data()` function that parses the `path` prop to determine what needs to be rendered. There are four possibilities:

1. Markdown file
    
2. MDX file
    
3. A specially named `index.md` file
    
4. A virtual index page
    

#1 and #3 are both Markdown files, but treated slightly differently. Let's take the example path:

`/docs/reference/deploy`

If there is a Markdown file on disk at `/docs/reference/deploy.md` then we're done: we give this document to [react-markdown](https://github.com/remarkjs/react-markdown) along with a couple of [remark plugins](https://github.com/remarkjs/remark) and the result is rendered inside the `<Success>` component of the cell.

However, if there is no `deploy.md` file we assume that `deploy` is instead a directory on disk and look for `deploy/index.md` instead. If one is found then we do the same as above.

If `deploy` is a directory but there's no `index.md` file we process and return option #4 above: we look in that directory for files and sub-directories and generate a virtual `index.md` file on the fly. It creates a list of all Markdown files in the directory, giving them titles and descriptions based on any frontmatter inside the files (if any). This virtual index ends up looking something like this:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1713207525313/5ceb877c-69c2-4ca7-9183-b763d5315964.png align="center")

The last piece of the puzzle is #2—MDX.

## Rendering MDX

Rendering MDX isn't normally a big deal in a standard React app, since MDX contains JSX and components that can be rendered in the browser just like any other React component. But, we're rendering MDX on the server, and that comes with the limitations of RSC—no access to things like `useState` or even `onClick` since these are all client-side concerns. Being able to click and interact with things is generally one of the main reasons you'd want to use MDX in a document!

After much lamentation and gnashing of teeth, our very own Josh Walker came up with a vite plugin that compiles MDX into `.mjs` files so that they can be slotted into the RSC workflow. This means you can include child components to be executed in the browser with `'use client'`! You can `import` components at the top of the MDX file, and then in those components you just add `'use client'` to the top if they cannot be executed on the server.

This is a slight exception to our "all docs should stay raw Markdown" original requirement, but there's really no way around it if you want to support MDX and client interactions with RSC.

## Where do the docs come from?

If you look at the Docwood repo you'll see no actual doc `.md` files. These are cloned from GitHub at build time and added to a `/docs` directory/workspace at the root of Docwood. The vite plugin mentioned above moves them into the generated `web/dist` structure at build time (still as raw `.md` files).

Any dependencies that any `.mdx` files may have will be added to the `package.json` in the `/docs` directory in redwoodjs/redwood and copied over to `/docs`, with a `yarn install` being run at build time to assure that those dependencies are installed and ready when vite compiles `.mdx` to `.mjs`.

## Deployment

Docwood is [currently live on Fly.io](https://docwood.fly.dev/docs)! Fly is one of the few hosts that support RSC at the moment, so it narrowed down the hosting options quite a bit. Another alternative today would be a [baremetal](https://redwoodjs.com/docs/deploy/baremetal) deploy to somewhere like AWS.

## But how does it feel?

Currently, clicking a link in the nav performs a full-page refresh of everything inside the layout, which is effectively the whole page. It's still relatively fast though, considering how much computation is happening under the hood (we haven't implemented caching yet so it's doing a file scan for all the docs to build the left nav, for example).

RSC is still being actively developed and new features added every day, so we expect the experience to improve quite a bit between now and the real release of [RedwoodJS Bighorn](https://redwoodjs.com/blog/bighorn-update).

## What's next?

We've got more updates coming, including styling, so stay tuned! Once RSC is ready, we'll publish this app as our official docs site, replacing what's currently available at https://redwoodjs.com/docs