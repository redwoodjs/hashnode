---
title: "Techniques for Fetching Data: Comparing Next.js (app and pages API), Remix, and RedwoodJS"
seoTitle: "Techniques for Fetching Data: Comparing 3 Different Frameworks"
seoDescription: "Take a look at Next.js, Remix, RedwoodJS data fetching for SaaS CRUD; compare APIs, static generation, SSR for best developer experience"
datePublished: Mon Apr 08 2024 17:12:21 GMT+0000 (Coordinated Universal Time)
cuid: clur7n6k7000108l6b5wfaf9q
slug: techniques-for-fetching-data
tags: reactjs, nextjs, redwoodjs, remix

---

All SaaS applications involve CRUD – Creating, Reading, Updating, and Deleting. 

Therefore, the way we fetch data naturally becomes a major piece of the developer experience and one of the many problems that a framework is able to solve.

Next.js app API, Next.js pages API, Remix, and RedwoodJS have all approached this problem differently, forming different opinions. Let’s look at each.

# **Next.js**

Out of all the frameworks, Next.js offers the most options and flexibility when it comes to fetching data. The business goals for your application will determine which method you reach for. So, let’s approach it by looking at the **problem** you’re trying to solve.

### **Problem: Your pages don’t change very often. You want the fastest page load possible, with maximum SEO and OG Tag benefits.**

I’d start by building static pages at build time. Next.js will try to deliver as much HTML and CSS to the browser as possible, resulting in snappy pages and happy Google crawlers.

Let’s look at some specific code examples.

First, it’s worth noting that these functions only work on the page level, when using the pages API.

![Screenshot of the pages directory within my Next.js application](https://cdn.hashnode.com/res/hashnode/image/upload/v1712255309774/485053be-e842-4e0a-b992-1c78ad97283a.png align="center")

Inside my **pages/about.tsx** file, I have some generic information for my podcast, [Compressed.fm](http://Compressed.fm), that I want to display:

* FAQs
    
* Recommended episodes for getting started with our podcast
    
* Our most popular episodes
    

We rarely update this information.

I have a standard page component:

```javascript
export default function About({gettingStarted, mostPopular, faqs}) {
  return (
      <InteriorLayout>
          <AboutPage
            faqs={faqs}
            gettingStarted={gettingStarted}
            mostPopular={mostPopular}
          />
      </InteriorLayout>
  );
}
```

I’m passing in the `gettingStarted`, `mostPopular`, and `faqs` content through props. Inside, I have a component called `InteriorLayout` that wraps the `AboutPage`. 

Then, within the same file, but outside the `About` page component, I’m going to call the `getStaticProps` function:

```javascript
export async function getStaticProps() {
  // get FAQs
  const faqs = await client.fetch(FaqQuery);


  // get Getting Started Episodes
  const gettingStarted = await client.fetch(GettingStartedEpisodesQuery);


  // get Popular Episodes
  const mostPopular = await client.fetch(PopularEpisodesQuery);


  return {
    props: {
      faqs,
      gettingStarted,
      mostPopular,
    },
  };
}
```

You’ll notice I’m `using client.fetch` to get some data. In this particular instance, all of the content is stored inside [Sanity](https://sanity.io), a content management system. I’m using GROQ to query their database and get the information I need. 

Then, I’m returning all the data within a props object.

You’ll notice that these props, `faqs`, `gettingStarted`, and `mostPopular` match the prop  names for the `About` page component.

```javascript
export default function About({gettingStarted, mostPopular, faqs}) {
```

What’s really happening here?

When the project gets deployed and built on the server, it runs the code inside `getStaticProps` and then it passes it on to the `About` page component. These values are essentially hard coded. If I change any of the content within Sanity, then I’ll need to rebuild the project, in order for the change to take an effect.

Since the content is “hard coded” in, it’s going to appear incredibly fast within the browser because we don’t need to query anything. The browser should have everything it needs. Google crawlers also have all the meta data they need.

This method works great in situations where you know what the URL is: `/about`.

But, what happens if you’re using dynamic URLs? With the podcast, I’m using `[slug].js` for an individual episode route.

The first episode is `/1` but the 150th episode is `/150`. All episodes reference the same page. The code will look at the URL, or the slug, and determine which episode content needs to be retrieved and served.

We can still use the `getStaticProps` method, but at build time, Next.js needs to know every possible URL variation and what episodes it needs to render.

That’s why you’ll frequently see `getStaticProps` paired with `getStaticPaths`.

**getStaticPaths**

Let’s take a look at our individual episode page. I have a standard episode page component:

```javascript
export default function Episode({ episode}) {
  return (
      <InteriorLayout>
        <IndividualEpisodePage episode={episode} />
      </InteriorLayout>
  );
}
```

This is similar to the `About` page. I still have an `InteriorLayout` component, but inside I’m displaying my `IndividualEpisodePage` and passing the appropriate `episode` data in via a prop.

Outside the `Episode` component, but within the same file, I also have a function called `getStaticPaths`.

```javascript
export async function getStaticPaths() {
  const allEpisodes = await client.fetch(AllEpisodesQuery);


  // Get the paths we want to pre-render based on episodes
  const paths = allEpisodes.map((episode) => ({
    params: { slug: episode.slug.current },
  }));


  return { paths, fallback: 'blocking' };
}
```

Here I’m querying Sanity for `allEpisodes`. Then, I’m looping over all the content that I get back from the CMS to build a `paths` array. This includes all the possible URLS and paths for the individual episodes that I might need.

You’ll notice that this is formatted in a very specific way. Each element in the `paths` array contains an `object` with a `params` property. `params` is also an object that includes the slug for that particular episode.

Depending on the size of your project, this might take an incredibly long time to build. So, there’s also an option to set a `fallback`. When it’s set to true, then, only a small subset of pages will be generated. When someone requests a page that is not generated yet, the user will see a loading component while the page builds and will eventually be replaced with the requested data. From then on, everyone else who requests the page will see the statically pre-rendered page. ([Documentation](https://nextjs.org/docs/pages/api-reference/functions/get-static-paths#when-is-fallback-true-useful))

Our work is not done yet, in order for `getStaticPaths` to work, we need to fetch the data for each path. In the same file, we’ll have another function:

```javascript
export async function getStaticProps({ params }) {
  const { slug } = params;
  const episode = await client.fetch(IndividualEpisodeQuery, { slug });
  return {
    props: {
      episode,
    },
  };
}
```

Here, the `getStaticProps` is similar to the implementation we used on the `about.jsx` page. The only difference is that we’re also accepting an object of `params`. We can grab the slug and use it to fetch data from Sanity. The `client.fetch(IndividualEpisodeQuery, { slug })` syntax is specific to Sanity. Regardless of what backend service you’re using, though, there’s probably a similar implementation. In our case, we’re using the slug within our `IndividualEpisodeQuery` to get the individual episode information for the slug that matches.

Lastly, we’ll return the `props` object with the `episode` information included.

Now, our `Episode` page component can put it to good use:

Same as before, anytime the site updates – regardless of whether you’re making a change on an existing episode or releasing a new one, you’ll need to trigger a new build in order for the change to take effect.

If you're interested, you can look at [the page code, all together](https://github.com/ahaywood/compressedfm/tree/52b8f1cc8dd2e311cee0e21668950de53a3165ac).

When the [Compressed.fm](http://Compressed.fm) site was built on Next.js, this was the method that we used. The site was incredibly fast, but each week when we’d drop a new episode, we’d also have to rebuild the site. For someone, like me, that’s tech savvy this isn’t a big deal, but if you’re using this implementation on a client project, this concept can be difficult to comprehend. So, let’s look at another option.

### **Problem: If your site changes from time to time and you don’t want to rebuild the site any time a change is made.**

If we’re not going to generate the site and hard code the values at build time, then we’ll need to do the same work on the server, when the user requests it. This time, we’ll reach for `getServerSideProps`.

**getServerSideProps**  
Let’s look at some more code examples. On the home page, I want any changes within my CMS to appear instantaneously on the site. I have my `Home` page component:

```javascript
export default function Home({ episodes }) {
  return (
    <>
      <MyHead />
      <HomeLayout>
        <HomePage episodes={episodes} />
      </HomeLayout>
    </>
  );
}
```

Nothing special to see here. Just a regular ‘ole React component that accepts an `episodes` prop, the data gets passed on to one of my child components.

Outside of the `Home` component, but inside the same file, Iet’s reach for the `getServerSideProps` function. Inside, we can `fetch` the data and then return it using the `props` object.

```javascript
export async function getServerSideProps() {
  const episodes = await client.fetch(RecentEpisodesQuery);
  return {
    props: { episodes },
  };
}
```

Remember, our `Home` page component is ready for it:

```javascript
export default function Home({ episodes }) {
```

The end result might be slightly slower. Google crawlers can still get what they need. And, the admin experience is better. This removes the need to rebuild the project every time a change is made, which could take anywhere from a couple of minutes to 15 or 30, depending on the size of your project.

  

You probably noticed in all our example code, I’m referencing the same project, but different pages. – And that’s the beauty of Next.js. You have several options at your disposal, depending on the use case. In fact, you can implement all these methods within a single project, making decisions on a page-by-page basis. 

  

All the options that we’ve looked at so far are within the pages API. But, you can also use the app API simultaneously. Let’s look at that use case:

### **Problem: If you want the server to do as much work as possible and send as much HTML and CSS to the browser as possible.**

Let’s reach for React Server Components (RSC)! 

React Server Components is the latest and greatest method. Currently, Next.js is the only framework that supports a *production ready* RSC implementation. (Redwood has recently [released a canary version](https://redwoodjs.com/blog/rsc-now-in-redwoodjs), available for experimentation.)

### **React Server Components (RSC)**

React Server Components allow you to render React on the server *and* the browser. With previous methods (like `getServerSideProps`), we’re only fetching data and running JavaScript functions on the server. With RSC, we’re doing that *plus* running React on the server.

Maybe you’re looking at this wondering why is that significant? React *seems* like a frontend templating language. Why would I want to use React on the server as well? 

> Unlike in rendering techniques like SSR and SSG, the HTML generated by RSCs is not hydrated on the server and no JS is shipped to the client. This significantly increases page load time and reduces the total JavaScript bundle size.  
> [**Log Rocket: React Server Components: A comprehensive guide**](https://blog.logrocket.com/react-server-components-comprehensive-guide/#:~:text=React%20Server%20Components%20allow%20you,executed%20exclusively%20on%20the%20server.)

There are several more advantages:

* **Improved performance** - intensive tasks are handled on the server, reducing the workload on the client. This improves Core Web Vitals like Largest Contentful Paint (LCP) and First Input Delay (FID).
    
* **SEO** - RSCs generate HTML on the server, meaning search engines can quickly and easily index the content and rank pages.
    
* **Enhanced security** - Sensitive information, like auth tokens, API keys, and SDKs are handled on the server and never exposed to the browser, preventing unintentional security leaks.
    
* **Data Fetching** - Data fetching is much faster, resulting in a better web experience.
    

Now, let’s take a look at the code.

In order to use RSC within Next.js, you have to use the app API. For this example, I’ll reference a gallery that features projects made with [Xata](https://xata.io/) (a backend as a service provider).

```javascript
export default async function Home() {
  const projects = await xata.db.project.filter('isApproved', true).getAll();
  const featuredProjects = projects.filter(
    (project) => project.featuredInCarousel
  );
```

Here, I have a `Home` page component. Take special notice, my data fetching happens directly within the component!

```javascript
xata.db.project.filter('isApproved', true).getAll();
```

I’m using Xata’s API (somewhat similar to Sanity’s) to grab all the projects where `isApproved` is marked as `true`. Then, I’m filtering the pens that are specifically marked as `featuredInCarousel`.

Then, within the JSX being returned by my component:

```javascript
<div className="grid grid-cols-2 gap-x-[60px] max-w-pageWidth mx-auto card-grid mb-[200px]">
    {projects.map((project) => (
        <div key={project.id}>
            <Card project={project} />
        </div>
    ))}
</div>
```

I can loop over the `projects` array. There’s a tighter coupling between where the data is fetched and then used. 😍

This is the future.

# **Remix**

Let’s look at how Remix solves the same problems.  

One of the ways where Remix shines is that it tries to lean into the existing Browser APIs and functionality. If the browser can do it, use it! There’s no need to reinvent the wheel or change what the browser has already solved natively. These APIs have been around for years and turns out, they’re quite good.

Already, you can probably tell Remix is more opinionated. Instead of providing 3 different possible solutions, it prescribes a single solution: the Data Loader Pattern.

With this setup, each page can have a `loader` function that can fetch data on the server and pass it to the page using a hook.

The component renders on the server and in the browser. But, the `loader` only runs on the server, meaning it’s safe to use API keys and SDKs. This is also for better processor intensive tasks, like parsing Markdown.

Let’s look at how this gets implemented in code. We’ll use the same project before, revisiting the individual episode page.

```javascript
export const loader = async ({ params }: LoaderArgs) => {
  const slug = params.slug;
  const episode = await getClient().fetch(IndividualEpisodeQuery, { slug });
  return { episode };
};
```

The `loader` function accepts an object that contains `params`. This is how I’m able to access the URL and know the specific episode that the user is looking for. Specifically, on line 2, I’m able to grab the `slug` from the `params` object.

On line 3, I’m going to `getClient().fetch(IndividualEpisodeQuery, { slug });` This should look similar to the code before. We’re still referencing Sanity and grabbing all the information we need from their system.

Then, we’re returning an object that has our `episode` included.

Within the same file, I have an `IndividualEpisode` component. You’ll notice this uses `export default` and contains all the page component code.

```javascript
export default function IndividualEpisode() {
  const { episode } = useLoaderData();
```

Within my component, I’m using the useLoaderData hook. That comes from Remix ([documentation](https://remix.run/docs/en/main/hooks/use-loader-data)):

```javascript
import type { LoaderArgs } from "@remix-run/node";
```

Now, I can use the `episode` data and display it directly on the page or pass it on to the appropriate component, all displayed within the browser.

# **RedwoodJS**

Redwood is at an interesting crossroads within our roadmap. Historically, we’ve been tied to GraphQL, supporting it as a first class citizen (and in case you’re worried about where this is going, we will always support GraphQL as a first class citizen). 

### **The Old(er) Way: GraphQL and Cells**

GraphQL plays beautifully with Redwood cells. Cells are a unique concept, specific to Redwood.

When you’re working with data on the client, there are several states that you have to take into consideration. You need to know when:

* **Loading** - If you’re still waiting on the server to get the data
    
* **Empty** - If the data set you received from the server is empty
    
* **Failure** - If there was an error with the data or on the server
    
* **Success** - You’ve successfully gotten the data you need from the server
    

This is a lot of state and a lot of boilerplate code to write *every time* you need to query the database. Redwood handles all these use cases for you!

The easiest way to create a Redwood cell is to use the command line tool that comes with Redwood:

```javascript
yarn redwood generate cell photos
// or, the shorthand:
yarn rw g cell photos
```

This will create a folder within the **components** directory that contains 4 files: a Storybook file, a mock data file, a component file (my cell), and a test file. 

> Redwood FTW! The Storybook file that’s generated already includes stories for each of our states: loading, empty, fail, and success.

If you pop open the cell, you’ll see something like this:

```javascript
export const QUERY = gql`
 query FindBlogListQuery($id: Int!) {
   blogList: blogList(id: $id) {
     id
     title
     date
   }
 }
`

export const Loading = () => <div>Loading...</div>

export const Empty = () => <div>Empty</div>

export const Failure = ({ error } => (
 <div style={{ color: 'red' }}>Error: {error?.message}</div>
)

export const Success = ({ blogList }) => {
 return <div>{JSON.stringify(blogList)}</div>
}
```

At the top we have a GraphQL Query. This component is simplistic. It’s grabbing a blog list, based on the `id` that we’re passing in. For each listing, we’re getting the `id`, `title`, and `date`.

Then, we have 4 components, one for each piece of state. It’s very important that you don’t change the name of these components because that’s how Redwood knows what to deliver.

***In the Redwood, anytime you want to query the database for data, use a cell.***

One of the beautiful things about this particular implementation is that the data QUERY and the component are bundled together.

I don’t have to prop drill, passing data down from a parent, down to it’s child, down to it’s child, down to it’s child… The component that *needs* the data is right next to the query.

If I want to drop this blog list component on multiple pages throughout my project, then, I can drop in `<BlogListCell />` and the component already has everything it needs. I don’t have to call the data *again* from the new page and prop drill.

I know. This solution is beautiful. So, what are the downfalls? Not everyone loves GraphQL. If you don’t know GraphQL it can feel like one.more.thing.™ you have to learn. For others, it’s considered as overhead.

<div data-node-type="callout">
<div data-node-type="callout-emoji">✏</div>
<div data-node-type="callout-text">Personal Note: I love GraphQL. I love the fact that I can say exactly what data I’m looking for, without over fetching. Plus, the result I get back is already formatted in a way that’s easy to consume.</div>
</div>

Now, I’ll admit, if you want to use a 3rd party service like Sanity or Xata or need to reference a REST API, you’ll have to filter it through Redwood’s GraphQL services, which can be cumbersome. – which is why we’re providing a new way.

### **The New Way: RSC and Developer’s Choice**

With the Bighorn epoch, we’re implementing React Server Components. This will give you all the benefits of the cell pattern, plus all the other benefits of RSC (improved performance, SEO, enhanced security, and faster data fetching.)

What does this look like?

A cell is still available, but instead of using a *client* cell with a GraphQL `QUERY` at the top of the file, you’ll use a *server* cell, with a data function instead.

In this code example, I have a photo editing and sharing application. At the top of my PhotoCell  I’ll get the data I need, based on the id I’m passing in:

```javascript
export const data = async ({ id }) => {
  return { photo: await photo(id) }
}
```

There’s no GraphQL query, instead I’m calling the photo method to get the data and process it on the server. The photo function is a custom function, written specifically for this project. But, the point is ***it’s running on the server!***

```javascript
export const Loading = () => <Blank title="Loading photo..." />


export const Empty = () => (
  <Blank
    title="No photos found"
    subtitle="Add photos to <code>web/public/photos</code> to get started."
  />
)


export const Failure = ({ error }) => <div>Error: {error.message}</div>


export const Success = ({ photo }) => {
  return <PhotoSuccess photo={photo} />
}
```

The rest of the cell should look familiar. Just like a client cell, you can drop this component anywhere within your Redwood project and it will gather the data it needs and display it properly within your application. 🎉

# **Concluding Thoughts**

**I appreciate the flexibility that Next.js provides.** The fact that you can choose on a page by page basis how the content will be fetched and served is incredibly powerful. However, that puts a lot of responsibility on the developer and can create decision fatigue.

I’ve worked on client projects before where they’ve said, “Yes, our content doesn’t update often. SEO is incredibly important to us. We want the fastest page load possible.” This sounds like the perfect candidate for `getStaticProps` and `getStaticPaths`. But, once that setup is delivered, confusion sets in. “Our administrative assistant is updating the site, within the CMS, and we aren’t seeing any changes on the site. Something’s wrong!” 

“Oh, well, first you need to *build* the site. Click on the deploy button, wait 10-15 minutes and you should see the updates.”

“You mean I have to wait 10-15 minutes every time a change is made?!”

“Yes.” 😬 

“That’s not what we want.”

Back to work! 💃I have to refactor everything for `getServerSideProps`.

**The data loading pattern that Remix has invented is interesting.** Within a single file, it becomes obvious what’s happening on the server and what’s happening within the browser. You load data in the `loader` function and then you serve it within the component. I love the clear delineation. 

But, I’ve encountered some interesting situations where I have very “fat files.” There’s a large `loader` function at the top that does *a lot* of work on the server to get everything I need. Then, prop drilling ensues, disseminating data to the appropriate component on the page. This just feels messy and smelly.

*It’s also worth mentioning that Remix is working on their own implementation of React Server Components.*

**I like the RedwoodJS client and server cells pattern.** I like having the query and component co-located. I can move the component around and it has everything it needs to display properly. Plus, it handles all the variations of state that come with fetching data.

Admittedly, there is a lot of set up that comes with the GraphQL / Client cell implementation. We only looked at the frontend and how the data is displayed, but there’s also some backend setup. (However, you’re going to have similar overhead with any backend implementation.) 

The move toward RSC should provide more options and naturally resolve some of these issues.