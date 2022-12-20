# 103 Early Hints

For a dynamic web application Time To First Byte ([TTFB](https://web.dev/ttfb/)) can take time, this is certainly the case with the applications I work with. For each request the server recieves, in order to respond correctly it could be performing a number of things that can add up; multiple api requests (some in a serial nature), computation of data, constructing a html response etc. All this takes time and whilst this is happening the browser is just… waiting!

This window is known as “server think time”, having sent the request the browser now puts its feet up and twiddles its fingers awaiting a response from the server!


![before-early-hints](https://user-images.githubusercontent.com/5073300/208551694-15b9ed7c-dc8c-498a-bbfd-1aa2d9441159.png)

What if we could utilise that waiting time and get the browser doing something useful that could potentially aid the performance of that page? This is where “Early Hints” comes in to play.

Early Hints is an additional response status code, **103**. Its currently only supported in Chrome and is classed in [experimental status](https://developer.mozilla.org/en-US/docs/MDN/Writing_guidelines/Experimental_deprecated_obsolete#experimental).

It’s used to send a preliminary response from the server whilst it is still preparing its final response. The intention is to send hints to the browser which it can use to begin to preconnect to required origins or preload required resources during the server think time phase. Examples could be critical CSS, JS resources, connections for 3rd party domains.


An early hints response looks something like this

```
HTTP/2 103
link: </styles1.css>; rel=preload; as=style
```

This attempts to get the browser doing some work in the server think period when normally it would be idle, fetching resources that would otherwise be delayed waiting for the server response. This has the potential to have a positive impact on the pages performance.

![after-early-hints](https://user-images.githubusercontent.com/5073300/208553166-db900b3f-8bde-4975-98e9-ae95d231d86e.png)

In the above example the style.css file is fetched alot earlier, during the server think time phase. Previously the browser would have had to wait for the first bytes of the server response to arrive before it had a chance to discover further resources such as the css file which it then had to fetch to render the page.

### This sounds like HTTP2/Server push...
Early hints on the surface sound similar to server push. There is one significant difference.

The challenge HTTP2/Server push had was it allowed the server to **push** subresources alongside the server response. This meant the server could push resources that the browser already had, i.e. resources already stored in its cache. This had the potential to lead to over fetching, network contention and it led to [no positive impact upon performance, in some cases it had an negative result](https://developer.chrome.com/blog/removing-push/).

The difference with Early Hints is that it does exactly what it suggests, it’s just a hint! This hands the control back to the browser, it can choose to listen to or ignore the hint. So if the browser reasons it does not need to fetch the resource highlighted in the hint (e.g. its already in its cache) it can choose not to. This provides an alternative that is less prone to error.

### Implementing Early Hints
Currently the only browser to support Early Hints is Chrome from version 103. Firefox looks to have [some work in progress](https://bugzilla.mozilla.org/show_bug.cgi?id=1407355)
 contributing towards supporting it.
 
There are some constraints in its usage. Probably the most noticable when you start playing is that Chrome will ignore hints sent over HTTP/1.1 or earlier. They are also ignored if they are sent on an iframe navigation or if its sent on a subresource request. Alongside this it currently is supporting just `preconnect` and `preload`.

From a server perspective node.js shipped support recently in [v18.11.0](https://nodejs.org/en/blog/release/v18.11.0/) for http and http2 on the server response object.

It can be simply implemented like the snippet below

```javascript
const earlyHintsLink = '</styles1.css>; rel=preload; as=style';

res.writeEarlyHints({
 'link': earlyHintsLink,
});
```

CDN's make the setup alot easier, provided your one supports it (cough, cough Akamai :wink:).

Fastly and Cloudflare both support this response status and offer an approach where you may not need to make any tweaks to your origin application. Cloudflare for example provides ability to switch on this feature and cache and serve 103 Early Hints from their edge servers, providing even greater opportunity for speed improvement. It works by looking at Link response headers from your origin responses, caches them and serves these cached hints on subsequent request. In the future theres a suggestion that machine learning could be used to generate these hints based on historical data!


### How to test Early Hints

The usual suspects can be used to test early hints, dev tools, WebPageTest etc.

In chrome If a resource is preloaded through Early Hints the associated PerformanceResourceTiming object will report that the initiatorType is `early-hints`

```javascript
performance.getEntriesByName('https://www.example-site.com/style.css')[0].initiatorType
// => 'early-hints'
```

### Early results
Early results and its important to emphasise **early***, have been promising. 

Cloudflare reported a [30% improvement in Largest Contentful Paint](https://blog.cloudflare.com/early-hints/) ([LCP](https://web.dev/lcp/)) from its initial tests.

Shopify (using Cloudflare) saw a [500ms improvement for LCP at p50](https://twitter.com/colinbendell/status/1539322190541295616).

### Next steps
Clearly its early days and I'm interested in experimenting more with Early Hints in the applications I contribute to discover alot more. 

None of the applications I work with in the day-to-day can return a 200 response immediatley. They are dynamic, personalised applications with a decent number of api dependencies, so there is a lot of **server think time** that could be taken advantage of. Early hints has the potential to benefit our users experiences with positive side affects such as SEO rankings. Watch this space...


