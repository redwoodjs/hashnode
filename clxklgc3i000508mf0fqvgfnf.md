---
title: ""Love reloaded": A DX Story"
seoTitle: ""Love reloaded": A DX Story"
datePublished: Tue Jun 18 2024 16:03:40 GMT+0000 (Coordinated Universal Time)
cuid: clxklgc3i000508mf0fqvgfnf
slug: love-reloaded-a-dx-story
tags: web-development, dx, developer-experience, redwoodjs

---

Redwood aims is to provide a fast, robust and comprehensible React Server Component experience. We're iterating towards improving that experience by adding a RSC development server that supports live reload.

## What is Live Reload?

Live reload is a feature that lets you see changes in your code by automatically refreshing your browser.

For development, live reload is the simplest and most basic method for "change something, see something," a fundamental web development technique.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1718709050390/cf7ef5f6-03a1-4df9-9460-e50044ac511c.png align="center")

We'll gradually enhance our implementation to achieve fast refresh (The best development experience). However, to release code quickly and iteratively improve the development experience we felt this was a valid improvement.

Here's how the three most common instant feedback mechanisms stack up against each other:

|  | Pro's ✅ | Con's ❌ |
| --- | --- | --- |
| Live reload | Simple and straightforward to implement. | The entire application restarts, causing a complete page reload. This can be slow, especially for large applications, and results in losing the current state of the application. |
| Hot module replacement (HMR) | Faster feedback loop as only the modified components are reloaded. Reduces the time taken to see changes. | The state of the reloaded components is lost, which might interrupt the development process if the state is significant for the current work. |
| Fast refresh | Combines fast feedback with state preservation, making it easier to see changes without losing the current state of your components. | Slightly more complex to implement compared to basic live reload, but the improved DX usually outweighs this complexity. |

## The implementation

All of the instant feedback mechanisms require that the front code running in the browser is able to communicate with the development server. The majority of these use a web socket connection that enables two way communication.

In our implementation we opted for the higher level `EventSource` web API which consumes server sent events.

> An `EventSource` instance opens a persistent connection to an [HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP) server, which sends [events](https://developer.mozilla.org/en-US/docs/Learn/JavaScript/Building_blocks/Events) in `text/event-stream` format. The connection remains open until closed by calling [`EventSource.close()`](https://developer.mozilla.org/en-US/docs/Web/API/EventSource/close).

The downside is that this only allows one-way communication, but it's really simple: The client listens for a `reload` event:

```javascript
const sse = new EventSource('http://localhost:8913');
sse.addEventListener('reload', () => {
    window.location.reload();
});
```

Which is sent from the server:

```javascript
const sendReloadEvent = () => {
    if (typeof client !== 'undefined') {
      client.write('event: reload\n')
      client.write('data: \n\n')
    }
}
```

## Conclusion

We're early in improving the development experience journey for React Server components and are focusing on making sure that we release early and often to gather feedback from the community.