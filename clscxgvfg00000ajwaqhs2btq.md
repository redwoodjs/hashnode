---
title: "Bighorn Update"
seoTitle: "Bighorn Update"
datePublished: Sun Mar 24 2024 05:00:00 GMT+0000 (Coordinated Universal Time)
cuid: clscxgvfg00000ajwaqhs2btq
slug: bighorn-update
tags: redwoodjs

---

Nine months ago, the RedwoodJS team decided to go [all-in on React Server Components](https://tom.preston-werner.com/2023/05/30/redwoods-next-epoch-all-in-on-rsc). The first version of Redwood that fully supports RSC will begin the Bighorn epoch. Today I’m excited to bring you an update on our work towards that goal.

We now have code that supports React SSR (and Suspense) and React Server Components (as well as server-side support for our traditional GraphQL data fetching). The Redwood Router has been updated to work with RSCs, and we’re busy on the next generation of Cells that will dramatically simplify Redwood’s data fetching layer.

To celebrate these milestones and better communicate our future vision and roadmap, we’ve revamped the [RedwoodJS website](https://redwoodjs.com/)! We hope this will make it easier to follow along as we approach our first Bighorn release in the coming months.

I’ve also begun a revision of the RedwoodJS Readme to more accurately describe the motivations and goals of the next epoch. Here’s a sneak peek:

> **Redwood is a framework for quickly creating React-based web applications that provide an amazing end user experience.** Our goal is to be simple and approachable enough for use in prototypes and hackathons, but performant and comprehensive enough to evolve into your next startup.
> 
> We accomplish this in two primary ways:
> 
> 1. Redwood is opinionated and full-stack. We’ve chosen the best technologies in the JS/TS ecosystem and beautifully integrated them into a cohesive framework that lets you get things done instead of endlessly evaluating technology options. You can get started using Redwood without a backend, but the framework really shines when you’re building a data driven application. Our transparent data fetching and optional GraphQL API make building and growing your application easier than you can imagine!
>     
> 2. Redwood’s declarative data fetching and simple form submission features are built on top of RSC + Server Actions and simplify common use cases so you can focus on your users’ experience. Creating the best, most responsive user interfaces requires reasoning about whether code should execute on the server or the client. Redwood makes it easy to choose the best execution context for your code by leveraging the power of React Server Components.
>     
> 
> The entire framework is built with TypeScript, so you get type safety from the router to the database and everywhere in-between. If you’d rather build your app with JavaScript, you can do that too, and still enjoy great code completion features in your favorite editor.

As we work towards full support of RSC (and friends), we will be releasing Bighorn canaries with ever-improving functionality. As we do so, we’ll make sure to document them here on the Redwood blog and social media. To ensure you don’t miss any updates, sign up for our [fortnightly newsletter.](https://mailchi.mp/redwoodjs/redwoodjs-newsletter)

Thank you to the core team and Redwood community for making Bighorn possible!