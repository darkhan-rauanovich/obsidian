# Part 1

## CPU, GPU, Memory, and multi-process architecture

[[rendering pipline]]

### CPU

First is the **C**entral **P**rocessing **U**nit - or **CPU**. The CPU can be considered your computer’s brain. A CPU core, pictured here as an office worker, can handle many different tasks one by one as they come in. It can handle everything from math to art while knowing how to reply to a customer call. In the past most CPUs were a single chip. A core is like another CPU living in the same chip. In modern hardware, you often get more than one core, giving more computing power to your phones and laptops.

![[Wx90M7DlxzVdXEeg5UhL.png]]


### GPU

**G**raphics **P**rocessing **U**nit - or **GPU** is another part of the computer. Unlike CPU, GPU is good at handling simple tasks but across multiple cores at the same time. As the name suggests, it was first developed to handle graphics. This is why in the context of graphics "using GPU" or "GPU-backed" is associated with fast rendering and smooth interaction. In recent years, with GPU-accelerated computing, more and more computation is becoming possible on GPU alone.

![[W6kFjvrwk1yDEhs8lFm1.png]]

When you start an application on your computer or phone, the CPU and GPU are the ones powering the application. Usually, applications run on the CPU and GPU using mechanisms provided by the Operating System

![[9M8aKlSl3207o9C3QVVp.png]]

### Executing program on Process and Thread

Another concept to grasp before diving into browser architecture is Process and Thread. A process can be described as an application’s executing program. A thread is the one that lives inside of process and executes any part of its process's program.

![[ICtmZ85CWgSJ7UZjomd1.png]]

When you start an application, a process is created. The program might create thread(s) to help it do work, but that's optional. The Operating System gives the process a "slab" of memory to work with and all application state is kept in that private memory space. When you close the application, the process also goes away and the Operating System frees up the memory.

![[x5h2ZL6SWI1vF5jSa8YB.svg]]

A process can ask the Operating System to spin up another process to run different tasks. When this happens, different parts of the memory are allocated for the new process. If two processes need to talk, they can do so by using **I**nter **P**rocess **C**ommunication (**IPC**). Many applications are designed to work this way so that if a worker process get unresponsive, it can be restarted without stopping other processes which are running different parts of the application.

![[OdFbLc2ufRmkJoHinTUL.svg]]


### Browser Architecture

So how is a web browser built using processes and threads? Well, it could be one process with many different threads or many different processes with a few threads communicating over IPC.

![[BG4tvT7y95iPAelkeadP.png]]

The important thing to note here is that these different architectures are implementation details. There is no standard specification on how one might build a web browser. One browser’s approach may be completely different from another.

For the sake of this blog series, we are going to use Chrome’s recent architecture described in the diagram below.

At the top is the browser process coordinating with other processes that take care of different parts of the application. For the renderer process, multiple processes are created and assigned to each tab. Until very recently, Chrome gave each tab a process when it could; now it tries to give each site its own process, including iframes (see [Site Isolation](https://developer.chrome.com/blog/inside-browser-part1/#site-isolation)).

![[JvSL0B5q1DmZAKgRHj42.png]]

### Which process controls what?

The following table describes each Chrome process and what it controls:

<table class="responsive"><tbody><tr><th colspan="2">Process and What it controls</th></tr><tr><td>Browser</td><td>Controls "chrome" part of the application including address bar, bookmarks, back and forward buttons. <br>Also handles the invisible, privileged parts of a web browser such as network requests and file access.</td></tr><tr><td>Renderer</td><td>Controls anything inside of the tab where a website is displayed.</td></tr><tr><td>Plugin</td><td>Controls any plugins used by the website, for example, flash.</td></tr><tr><td>GPU</td><td>Handles GPU tasks in isolation from other processes. It is separated into different process because GPUs handles requests from multiple apps and draw them in the same surface.</td></tr></tbody></table> 

![[vl5sRzL8pFwlLSN7WW12.png]]

### The benefit of multi-process architecture in Chrome

Earlier, I mentioned Chrome uses multiple renderer process. In the most simple case, you can imagine each tab has its own renderer process. Let’s say you have 3 tabs open and each tab is run by an independent renderer process. If one tab becomes unresponsive, then you can close the unresponsive tab and move on while keeping other tabs alive. If all tabs are running on one process, when one tab becomes unresponsive, all the tabs are unresponsive. That’s sad.

![[ZVkrl0QErFtITKPwa6Cq.png]]

Another benefit of separating the browser's work into multiple processes is security and sandboxing. Since operating systems provide a way to restrict processes’ privileges, the browser can sandbox certain processes from certain features. For example, the Chrome browser restricts arbitrary file access for processes that handle arbitrary user input like the renderer process.

Because processes have their own private memory space, they often contain copies of common infrastructure (like V8 which is a Chrome's JavaScript engine). This means more memory usage as they can't be shared the way they would be if they were threads inside the same process. In order to save memory, Chrome puts a limit on how many processes it can spin up. The limit varies depending on how much memory and CPU power your device has, but when Chrome hits the limit, it starts to run multiple tabs from the same site in one process.

### Saving more memory - Servicification in Chrome

The same approach is applied to the browser process. Chrome is undergoing architecture changes to run each part of the browser program as a service allowing to easily split into different processes or aggregate into one.

General idea is that when Chrome is running on powerful hardware, it may split each service into different processes giving more stability, but if it is on a resource-constraint device, Chrome consolidates services into one process saving memory footprint. Similar approach of consolidating processes for less memory usage have been used on platform like Android before this change.

![[8zHB7KNXrIKv5yAWvtBy.svg]]

Figure 11: Diagram of Chrome’s servicification moving different services into multiple processes and a single browser process

### Per-frame renderer processes - Site Isolation

[Site Isolation](https://developers.google.com//web/updates/2018/07/site-isolation) is a recently introduced feature in Chrome that runs a separate renderer process for each cross-site iframe. We’ve been talking about one renderer process per tab model which allowed cross-site iframes to run in a single renderer process with sharing memory space between different sites. Running a.com and b.com in the same renderer process might seem okay. The [Same Origin Policy](https://developer.mozilla.org/docs/Web/Security/Same-origin_policy) is the core security model of the web; it makes sure one site cannot access data from other sites without consent. Bypassing this policy is a primary goal of security attacks. Process isolation is the most effective way to separate sites. With [Meltdown and Spectre](https://developers.google.com/web/updates/2018/02/meltdown-spectre), it became even more apparent that we need to separate sites using processes. With Site Isolation enabled on desktop by default since Chrome 67, each cross-site iframe in a tab gets a separate renderer process.

![[7ilepBEw6b2yUuyABbpZ.png]]


Enabling Site Isolation has been a multi-year engineering effort. Site Isolation isn’t as simple as assigning different renderer processes; it fundamentally changes the way iframes talk to each other. Opening devtools on a page with iframes running on different processes means devtools had to implement behind-the-scenes work to make it appear seamless. Even running a simple Ctrl+F to find a word in a page means searching across different renderer processes. You can see the reason why browser engineers talk about the release of Site Isolation as a major milestone!

# Part 2

## What happens in navigation

This is part 2 of a 4 part blog series looking at the inner workings of Chrome. In [the previous post](https://developers.google.com/web/updates/2018/09/inside-browser-part1), we looked at how different processes and threads handle different parts of a browser. In this post, we dig deeper into how each process and thread communicate in order to display a website.

Let’s look at a simple use case of web browsing: you type a URL into a browser, then the browser fetches data from the internet and displays a page. In this post, we’ll focus on the part where a user requests a site and the browser prepares to render a page - also known as a navigation.

### It starts with a browser process

As we covered in [part 1: CPU, GPU, Memory, and multi-process architecture](https://developers.google.com/web/updates/2018/09/inside-browser-part1), everything outside of a tab is handled by the browser process. The browser process has threads like the UI thread which draws buttons and input fields of the browser, the network thread which deals with network stack to receive data from the internet, the storage thread that controls access to the files and more. When you type a URL into the address bar, your input is handled by browser process’s UI thread.

![[lo3x7Zt4LZ4ltsQQjLns.png]]

Figure 1: Browser UI at the top, diagram of the browser process with UI, network, and storage thread inside at the bottom

### A simple navigation

#### Step 1: Handling input

When a user starts to type into the address bar, the first thing UI thread asks is "Is this a search query or URL?". In Chrome, the address bar is also a search input field, so the UI thread needs to parse and decide whether to send you to a search engine, or to the site you requested.

![[HDAB6c70Jo2IvsUl0giY.png]]

Figure 1: UI Thread asking if the input is a search query or a URL

#### Step 2: Start navigation

When a user hits enter, the UI thread initiates a network call to get site content. Loading spinner is displayed on the corner of a tab, and the network thread goes through appropriate protocols like [[DNS]] lookup and establishing [[TLS Connection]] for the request.

![[nSD7ognQ9hNFoFOnFQlw.png]]

Figure 2: the UI thread talking to the network thread to navigate to mysite.com

At this point, the network thread may receive a server redirect header like HTTP 301. In that case, the network thread communicates with UI thread that the server is requesting redirect. Then, another URL request will be initiated.

#### Step 3: Read response

Once the response body (payload) starts to come in, the network thread looks at the first few bytes of the stream if necessary. The response's Content-Type header should say what type of data it is, but since it may be missing or wrong, [MIME Type sniffing](https://developer.mozilla.org/docs/Web/HTTP/Basics_of_HTTP/MIME_types) ([[MIME Type]]) is done here. This is a "tricky business" as commented in [the source code](https://cs.chromium.org/chromium/src/net/base/mime_sniffer.cc?sq=package:chromium&dr=CS&l=5). You can read the comment to see how different browsers treat content-type/payload pairs.

![[PTmbGdEyTDdLDrAbJw4v.png]]

Figure 3: response header which contains Content-Type and payload which is the actual data

If the response is an HTML file, then the next step would be to pass the data to the renderer process, but if it is a zip file or some other file then that means it is a download request so they need to pass the data to download manager.

![[pn0zlnxoYgbyzFVKoTc9.png]]

This is also where the [SafeBrowsing](https://safebrowsing.google.com/) check happens. If the domain and the response data seems to match a known malicious site, then the network thread alerts to display a warning page. Additionally, [**C**ross **O**rigin **R**ead **B**locking (**CORB**)](https://www.chromium.org/Home/chromium-security/corb-for-developers)  ([[CORB]]) check happens in order to make sure sensitive cross-site data does not make it to the renderer process.

#### Step 4: Find a renderer process

Once all of the checks are done and Network thread is confident that browser should navigate to the requested site, the Network thread tells UI thread that the data is ready. UI thread then finds a renderer process to carry on rendering of the web page.

![[VAR3s7k8rIgTrfwEWMIo.png]]

Figure 5: Network thread telling UI thread to find Renderer Process

Since the network request could take several hundred milliseconds to get a response back, an optimization to speed up this process is applied. When the UI thread is sending a URL request to the network thread at step 2, it already knows which site they are navigating to. The UI thread tries to proactively find or start a renderer process in parallel to the network request. This way, if all goes as expected, a renderer process is already in standby position when the network thread received data. This standby process might not get used if the navigation redirects cross-site, in which case a different process might be needed.

#### Step 5: Commit navigation

Now that the data and the renderer process is ready, an IPC is sent from the browser process to the renderer process to commit the navigation. It also passes on the data stream so the renderer process can keep receiving HTML data. Once the browser process hears confirmation that the commit has happened in the renderer process, the navigation is complete and the document loading phase begins.

At this point, address bar is updated and the security indicator and site settings UI reflects the site information of the new page. The session history for the tab will be updated so back/forward buttons will step through the site that was just navigated to. To facilitate tab/session restore when you close a tab or window, the session history is stored on disk.

![[kL6CLP7fLay9L99vRR3F.png]]

Figure 6: IPC between the browser and the renderer processes, requesting to render the page

#### Extra Step: Initial load complete

Once the navigation is committed, the renderer process carries on loading resources and renders the page. We will go over the details of what happens at this stage in the next post. Once the renderer process "finishes" rendering, it sends an IPC back to the browser process (this is after all the `onload` events have fired on all frames in the page and have finished executing). At this point, the UI thread stops the loading spinner on the tab.

I say "finishes", because client side JavaScript could still load additional resources and render new views after this point.

![[DwMkwQndYadDqnMtp8T3.png]]

Figure 7: IPC from the renderer to the browser process to notify the page has "loaded"

### Navigating to a different site

The simple navigation was complete! But what happens if a user puts different URL to address bar again? Well, the browser process goes through the same steps to navigate to the different site. But before it can do that, it needs to check with the currently rendered site if they care about [`beforeunload`](https://developer.mozilla.org/docs/Web/Events/beforeunload) event.

`beforeunload` can create "Leave this site?" alert when you try to navigate away or close the tab. Everything inside of a tab including your JavaScript code is handled by the renderer process, so the browser process has to check with current renderer process when new navigation request comes in.

**Caution**

Do not add unconditional `beforeunload` handlers. It creates more latency because the handler needs to be executed before the navigation can even be started. This event handler should be added only when needed, for example if users need to be warned that they might lose data they've entered on the page.

![[u7EEPH9S2PpycpbQQRFk.png]]

Figure 8: IPC from the browser process to a renderer process telling it that it's about to navigate to a different site

If the navigation was initiated from the renderer process (like user clicked on a link or client-side JavaScript has run `window.location = "https://newsite.com"`) the renderer process first checks `beforeunload` handlers. Then, it goes through the same process as browser process initiated navigation. The only difference is that navigation request is kicked off from the renderer process to the browser process.

When the new navigation is made to a different site than currently rendered one, a separate render process is called in to handle the new navigation while current render process is kept around to handle events like `unload`. For more, see [an overview of page lifecycle states](https://developers.google.com/web/updates/2018/07/page-lifecycle-api#overview_of_page_lifecycle_states_and_events) and how you can hook into events with [the Page Lifecycle API](https://developers.google.com/web/updates/2018/07/page-lifecycle-api) ([[The Page lifecycle API]]). 

![[5tThsmZamrpxFydJFePg.png]]

Figure 9: 2 IPCs from a browser process to a new renderer process telling to render the page and telling old renderer process to unload


### In case of Service Worker

One recent change to this navigation process is the introduction of [service worker](https://developers.google.com/web/fundamentals/primers/service-workers/)  ([[Service Worker]]). Service worker is a way to write network proxy in your application code; allowing web developers to have more control over what to cache locally and when to get new data from the network. If service worker is set to load the page from the cache, there is no need to request the data from the network.

The important part to remember is that service worker is JavaScript code that runs in a renderer process. But when the navigation request comes in, how does a browser process know the site has a service worker?

When a service worker is registered, the scope of the service worker is kept as a reference (you can read more about scope in this [The Service Worker Lifecycle](https://developers.google.com/web/fundamentals/primers/service-workers/lifecycle) article). When a navigation happens, network thread checks the domain against registered service worker scopes, if a service worker is registered for that URL, the UI thread finds a renderer process in order to execute the service worker code. The service worker may load data from cache, eliminating the need to request data from the network, or it may request new resources from the network.

![[x65o4xjohKMgf5QDEWPG.png]]

Figure 10: the network thread in the browser process looking up service worker scope

![[fuk5vjgLg4sZZTLAMCEB.png]]

Figure 11: the UI thread in a browser process starting up a renderer process to handle service workers; a worker thread in a renderer process then requests data from the network

### Navigation Preload

You can see this round trip between the browser process and renderer process could result in delays if service worker eventually decides to request data from the network. [Navigation Preload](https://developers.google.com/web/updates/2017/02/navigation-preload) is a mechanism to speed up this process by loading resources in parallel to service worker startup. It marks these requests with a header, allowing servers to decide to send different content for these requests; for example, just updated data instead of a full document.

![[xAESXRJNybpxPK5dL3m7.png]]

Figure 12: the UI thread in a browser process starting up a renderer process to handle service worker while kicking off network request in parallel

# Part 3

## Inner workings of a Renderer Process

This is part 3 of 4 part blog series looking at how browsers work. Previously, we covered [multi-process architecture](https://developers.google.com/web/updates/2018/09/inside-browser-part1) and [navigation flow](https://developers.google.com/web/updates/2018/09/inside-browser-part2). In this post, we are going to look at what happens inside of the renderer process.

Renderer process touches many aspects of web performance. Since there is a lot happening inside of the renderer process, this post is only a general overview. If you'd like to dig deeper, [the Performance section of Web Fundamentals](https://developers.google.com/web/fundamentals/performance/why-performance-matters/) has many more resources.

## Renderer processes handle web contents

The renderer process is responsible for everything that happens inside of a tab. In a renderer process, the main thread handles most of the code you send to the user. Sometimes parts of your JavaScript is handled by worker threads if you use a web worker or a service worker. Compositor and raster threads are also run inside of a renderer processes to render a page efficiently and smoothly.

The renderer process's core job is to turn HTML, CSS, and JavaScript into a web page that the user can interact with.

![[uIqf0QQZxF6mHPDWFEjz.png]]

Figure 1: Renderer process with a main thread, worker threads, a compositor thread, and a raster thread inside

## Parsing

## Style calculation

Having a DOM is not enough to know what the page would look like because we can style page elements in CSS. The main thread parses CSS and determines the computed style for each DOM node. This is information about what kind of style is applied to each element based on CSS selectors. You can see this information in the `computed` section of DevTools.

![[hGqtsAuYpEYX4emJd5Jw.png]]

Even if you do not provide any CSS, each DOM node has a computed style. `<h1>` tag is displayed bigger than `<h2>` tag and margins are defined for each element. This is because the browser has a default style sheet. If you want to know what Chrome's default CSS is like, [you can see the source code here](https://cs.chromium.org/chromium/src/third_party/blink/renderer/core/html/resources/html.css).


### Construction of a DOM

When the renderer process receives a commit message for a navigation and starts to receive HTML data, the main thread begins to parse the text string (HTML) and turn it into a **D**ocument **O**bject **M**odel (**[[DOM]]**).

The DOM is a browser's internal representation of the page as well as the data structure and API that web developer can interact with via JavaScript.

Parsing an HTML document into a DOM is defined by the [HTML Standard](https://html.spec.whatwg.org/). ([[HTML Standart]]) You may have noticed that feeding HTML to a browser never throws an error. For example, missing closing `</p>` tag is a valid HTML. Erroneous markup like `Hi! <b>I'm <i>Chrome</b>!</i>` (b tag is closed before i tag) is treated as if you wrote `Hi! <b>I'm <i>Chrome</i></b><i>!</i>`. This is because the HTML specification is designed to handle those errors gracefully. If you are curious how these things are done, you can read on "[An introduction to error handling and strange cases in the parser](https://html.spec.whatwg.org/multipage/parsing.html#an-introduction-to-error-handling-and-strange-cases-in-the-parser)" [[Error handling in the parcer]] section of the HTML spec.

### Subresource loading

A website usually uses external resources like images, CSS, and JavaScript. Those files need to be loaded from network or cache. The main thread _could_ request them one by one as they find them while parsing to build a DOM, but in order to speed up, "preload scanner" is run concurrently. If there are things like `<img>` or `<link>` in the HTML document, preload scanner peeks at tokens generated by HTML parser and sends requests to the network thread in the browser process.

![[qmuN5aduuEit6SZfwVOi.png]]

### JavaScript can block the parsing

When the HTML parser finds a `<script>` tag, it pauses the parsing of the HTML document and has to load, parse, and execute the JavaScript code. Why? because JavaScript can change the shape of the document using things like `document.write()` which changes the entire DOM structure ([overview of the parsing model](https://html.spec.whatwg.org/multipage/parsing.html#overview-of-the-parsing-model) in the HTML spec has a nice diagram). This is why the HTML parser has to wait for JavaScript to run before it can resume parsing of the HTML document. If you are curious about what happens in JavaScript execution, [[V8]]  [the V8 team has talks and blog posts on this](https://mathiasbynens.be/notes/shapes-ics) .

### Hint to browser how you want to load resources

There are many ways web developers can send hints to the browser in order to load resources nicely. If your JavaScript does not use `document.write()`, you can add [`async`](https://developer.mozilla.org/docs/Web/HTML/Element/script#attr-async) or [`defer`](https://developer.mozilla.org/docs/Web/HTML/Element/script#attr-defer) attribute to the `<script>` tag. The browser then loads and runs the JavaScript code asynchronously and does not block the parsing. You may also use [JavaScript module](https://developers.google.com/web/fundamentals/primers/modules) if that's suitable. `<link rel="preload">` is a way to inform browser that the resource is definitely needed for current navigation and you would like to download as soon as possible. You can read more on this at [Resource Prioritization – Getting the Browser to Help You](https://developers.google.com/web/fundamentals/performance/resource-prioritization).

## Layout

Now the renderer process knows the structure of a document and styles for each nodes, but that is not enough to render a page. Imagine you are trying to describe a painting to your friend over a phone. "There is a big red circle and a small blue square" is not enough information for your friend to know what exactly the painting would look like.

![[GbgUOpTYR0nZBX1YUEDl.png]]

The layout is a process to find the geometry of elements. The main thread walks through the DOM and computed styles and creates the layout tree which has information like x y coordinates and bounding box sizes. Layout tree may be similar structure to the DOM tree, but it only contains information related to what's visible on the page. If `display: none` is applied, that element is not part of the layout tree (however, an element with `visibility: hidden` is in the layout tree). Similarly, if a pseudo class with content like `p::before{content:"Hi!"}` is applied, it is included in the layout tree even though that is not in the DOM.

![[0JqiVwHxNab2YL6qWHbS.png]]

Figure 5: The main thread going over DOM tree with computed styles and producing layout tree

Determining the Layout of a page is a challenging task. Even the simplest page layout like a block flow from top to bottom has to consider how big the font is and where to line break them because those affect the size and shape of a paragraph; which then affects where the following paragraph needs to be.

![[rXSCtc21M00XrRqcw56C.mp4]]

CSS can make element float to one side, mask overflow item, and change writing directions. You can imagine, this layout stage has a mighty task. In Chrome, a whole team of engineers works on the layout. If you want to see details of their work, [few talks from BlinkOn Conference](https://www.youtube.com/watch?v=Y5Xa4H2wtVA) are recorded and quite interesting to watch.

## Paint

Having a DOM, style, and layout is still not enough to render a page. Let's say you are trying to reproduce a painting. You know the size, shape, and location of elements, but you still have to judge in what order you paint them.

![[Il8s9Bl7vS9pBLhMCLju.png]]

Figure 7: A person in front of a canvas holding paintbrush, wondering if they should draw a circle first or square first

For example, `z-index` might be set for certain elements, in that case painting in order of elements written in the HTML will result in incorrect rendering.

![[4x9etJ64cg0x4a6Ktt5T.png]]

Figure 8: Page elements appearing in order of an HTML markup, resulting in wrong rendered image because z-index was not taken into account

At this paint step, the main thread walks the layout tree to create paint records. Paint record is a note of painting process like "background first, then text, then rectangle". If you have drawn on `<canvas>` element using JavaScript, this process might be familiar to you.

![[zs8wNimWDPhu7NIhJDcc.png]]

Figure 9: The main thread walking through layout tree and producing paint records

### Updating rendering pipeline is costly

The most important thing to grasp in rendering pipeline is that at each step the result of the previous operation is used to create new data. For example, if something changes in the layout tree, then the Paint order needs to be regenerated for affected parts of the document.

![[d7zOpwpNIXIoVnoZCtI9.mp4]]

Figure 10: DOM+Style, Layout, and Paint trees in order it is generated

If you are animating elements, the browser has to run these operations in between every frame. Most of our displays refresh the screen 60 times a second (60 fps); animation will appear smooth to human eyes when you are moving things across the screen at every frame. However, if the animation misses the frames in between, then the page will appear "janky".

![[b3nyw5eLlDIM7rl9bxFC.png]]

Figure 11: Animation frames on a timeline

Even if your rendering operations are keeping up with screen refresh, these calculations are running on the main thread, which means it could be blocked when your application is running JavaScript.

![[FryonpF90Ei9JYYGi1UI.png]]

You can divide JavaScript operation into small chunks and schedule to run at every frame using `requestAnimationFrame()`. For more on this topic, please see [Optimize JavaScript Execution](https://developers.google.com/web/fundamentals/performance/rendering/optimize-javascript-execution) . You might also run your [JavaScript in Web Workers](https://www.youtube.com/watch?v=X57mh8tKkgE) to avoid blocking the main thread.

![[ypzLFiu34WCuhNHm7F0B.png]]

Figure 13: Smaller chunks of JavaScript running on a timeline with animation frame

## Compositing

### How would you draw a page?

Now that the browser knows the structure of the document, the style of each element, the geometry of the page, and the paint order, how does it draw a page? Turning this information into pixels on the screen is called rasterizing.

![[AiIny83Lk4rTzsM8bxSn.mp4]]


Perhaps a naive way to handle this would be to raster parts inside of the viewport. If a user scrolls the page, then move the rastered frame, and fill in the missing parts by rastering more. This is how Chrome handled rasterizing when it was first released. However, the modern browser runs a more sophisticated process called compositing.

### What is compositing

Compositing is a technique to separate parts of a page into layers, rasterize them separately, and composite as a page in a separate thread called compositor thread. If scroll happens, since layers are already rasterized, all it has to do is to composite a new frame. Animation can be achieved in the same way by moving layers and composite a new frame.

![[Aggd8YLFPckZrBjEj74H.mp4]]

### Dividing into layers

In order to find out which elements need to be in which layers, the main thread walks through the layout tree to create the layer tree (this part is called "Update Layer Tree" in the DevTools performance panel). If certain parts of a page that should be separate layer (like slide-in side menu) is not getting one, then you can hint to the browser by using `will-change` attribute in CSS.

![[V667Geh9MtTviJjDkGZq.png]]

Figure 16: The main thread walking through layout tree producing layer tree

You might be tempted to give layers to every element, but compositing across an excess number of layers could result in slower operation than rasterizing small parts of a page every frame, so it is crucial that you measure rendering performance of your application. For more about on topic, see [Stick to Compositor-Only Properties and Manage Layer Count](https://developers.google.com/web/fundamentals/performance/rendering/stick-to-compositor-only-properties-and-manage-layer-count).

### Raster and composite off of the main thread

Once the layer tree is created and paint orders are determined, the main thread commits that information to the compositor thread. The compositor thread then rasterizes each layer. A layer could be large like the entire length of a page, so the compositor thread divides them into tiles and sends each tile off to raster threads. Raster threads rasterize each tile and store them in GPU memory.

![[SL4KO5UsGgBNLrOwb0wC.png]]

Figure 17: Raster threads creating the bitmap of tiles and sending to GPU

The compositor thread can prioritize different raster threads so that things within the viewport (or nearby) can be rastered first. A layer also has multiple tilings for different resolutions to handle things like zoom-in action.

Once tiles are rastered, compositor thread gathers tile information called **draw quads** to create a **compositor frame**.

Draw quads

Contains information such as the tile's location in memory and where in the page to draw the tile taking in consideration of the page compositing.

Compositor frame

A collection of draw quads that represents a frame of a page.

A compositor frame is then submitted to the browser process via IPC. At this point, another compositor frame could be added from UI thread for the browser UI change or from other renderer processes for extensions. These compositor frames are sent to the GPU to display it on a screen. If a scroll event comes in, compositor thread creates another compositor frame to be sent to the GPU.

![[tG4AzFeS3IdfTSawnFL6 (1).png]]

Figure 18: Compositor thread creating compositing frame. Frame is sent to the browser process then to GPU

The benefit of compositing is that it is done without involving the main thread. Compositor thread does not need to wait on style calculation or JavaScript execution. This is why [compositing only animations](https://www.html5rocks.com/en/tutorials/speed/high-performance-animations/) are considered the best for smooth performance. If layout or paint needs to be calculated again then the main thread has to be involved.

# Part 4

## Input is coming to the Compositor

This is the last of the 4 part blog series looking inside of Chrome; investigating how it handles our code to display a website. In the previous post, we looked at [the rendering process and learned about the compositor](https://developers.google.com//web/updates/2018/09/inside-browser-part3). In this post, we'll look at how compositor is enabling smooth interaction when user input comes in.

## Input events from the browser's point of view

When you hear "input events" you might only think of a typing in textbox or mouse click, but from the browser's point of view, input means any gesture from the user. Mouse wheel scroll is an input event and touch or mouse over is also an input event.

When user gesture like touch on a screen occurs, the browser process is the one that receives the gesture at first. However, the browser process is only aware of where that gesture occurred since content inside of a tab is handled by the renderer process. So the browser process sends the event type (like `touchstart`) and its coordinates to the renderer process. Renderer process handles the event appropriately by finding the event target and running event listeners that are attached.

![[ahDODQbpiTZX6lauff5T.png]]

Figure 1: Input event routed through the browser process to the renderer process

## Compositor receives input events

In the previous post, we looked at how the compositor could handle scroll smoothly by compositing rasterized layers. If no input event listeners are attached to the page, Compositor thread can create a new composite frame completely independent of the main thread. But what if some event listeners were attached to the page? How would the compositor thread find out if the event needs to be handled?

![[Aggd8YLFPckZrBjEj74H.mp4]]

## Understanding non-fast scrollable region

Since running JavaScript is the main thread's job, when a page is composited, the compositor thread marks a region of the page that has event handlers attached as "Non-Fast Scrollable Region". By having this information, the compositor thread can make sure to send input event to the main thread if the event occurs in that region. If input event comes from outside of this region, then the compositor thread carries on compositing new frame without waiting for the main thread.

![[F2nDPjKxnlXxuG1SAUnt.png]]

Figure 3: Diagram of described input to the non-fast scrollable region

### Be aware when you write event handlers

A common event handling pattern in web development is event delegation. Since events bubble, you can attach one event handler at the topmost element and delegate tasks based on event target. You might have seen or written code like the below.

```javascript
document.body.addEventListener('touchstart', event => {    
	if (event.target === area) {
	    event.preventDefault();    
	}
});
```

Since you only need to write one event handler for all elements, ergonomics of this event delegation pattern are attractive. However, if you look at this code from the browser's point of view, now the entire page is marked as a non-fast scrollable region. This means even if your application doesn't care about input from certain parts of the page, the compositor thread has to communicate with the main thread and wait for it every time an input event comes in. Thus, the smooth scrolling ability of the compositor is defeated.

![[X46tkweWpcsUIKzfAtuh.png]]

Figure 4: Diagram of described input to the non-fast scrollable region covering an entire page

In order to mitigate this from happening, you can pass `passive: true` options in your event listener. This hints to the browser that you still want to listen to the event in the main thread, but compositor can go ahead and composite new frame as well.

```javascript
document.body.addEventListener('touchstart', event => {    
	if (event.target === area) {
	    event.preventDefault();    
	}
}, {passive: true});
```

## Check if the event is cancelable

Imagine you have a box in a page that you want to limit scroll direction to horizontal scroll only.

![[Y3cPoWi9S1uczLToDxcx.png]]
Figure 5: A web page with part of the page fixed to horizontal scroll

Using `passive: true` option in your pointer event means that the page scroll can be smooth, but vertical scroll might have started by the time you want to `preventDefault` in order to limit scroll direction. You can check against this by using `event.cancelable` method.

```javascript
document.body.addEventListener('pointermove', event => {    
	if (event.cancelable) {        
		event.preventDefault(); // block the native scroll        
		/*        *  do what you want the application to do here        */
    }
}, {passive: true});
```

Alternatively, you may use CSS rule like `touch-action` to completely eliminate the event handler.

```css
#area {
	touch-action: pan-x;
}
```

## Finding the event target

When the compositor thread sends an input event to the main thread, the first thing to run is a hit test to find the event target. Hit test uses paint records data that was generated in the rendering process to find out what is underneath the point coordinates in which the event occurred.

![[6dN5zsCK46dMNqwkO7EG 1.png]]

Figure 6: The main thread looking at the paint records asking what's drawn on x.y point

## Minimizing event dispatches to the main thread

In the previous post, we discussed how our typical display refreshes screen 60 times a second and how we need to keep up with the cadence for smooth animation. For input, a typical touch-screen device delivers touch event 60-120 times a second, and a typical mouse delivers events 100 times a second. Input event has higher fidelity than our screen can refresh.

If a continuous event like `touchmove` was sent to the main thread 120 times a second, then it might trigger excessive amount of hit tests and JavaScript execution compared to how slow the screen can refresh.

![[1cyHbX3uaB0CCSCuX8vJ.png]]

To minimize excessive calls to the main thread, Chrome coalesces continuous events (such as `wheel`, `mousewheel`, `mousemove`, `pointermove`, `touchmove` ) and delays dispatching until right before the next `requestAnimationFrame`.

![[XRCMvR1Us631HNEg8g62.png]]


## Use `getCoalescedEvents` to get intra-frame events

For most web applications, coalesced events should be enough to provide a good user experience. However, if you are building things like drawing application and putting a path based on `touchmove` coordinates, you may lose in-between coordinates to draw a smooth line. In that case, you can use the `getCoalescedEvents` method in the pointer event to get information about those coalesced events.

![[P0kK5GGwmOEBXd3TjLDq.png]]

Figure 9: Smooth touch gesture path on the left, coalesced limited path on the right

```javascript
window.addEventListener('pointermove', event => {
	const events = event.getCoalescedEvents();
    for (let event of events) {
        const x = event.pageX;
		const y = event.pageY;        
		// draw a line using x and y coordinates.    
	}
});
```

# Next steps

### Use Lighthouse

If you want to make your code be nice to the browser but have no idea where to start, [Lighthouse](https://developer.chrome.com/docs/lighthouse/overview/) ([[Lighthouse]]) is a tool that runs audit of any website and gives you a report on what's being done right and what needs improvement. Reading through the list of audits also gives you an idea of what kind of things a browser cares about.

### Learn how to measure performance

Performance tweaks may vary for different sites, so it is crucial that you measure the performance of your site and decide what fits the best for your site. Chrome DevTools team has few tutorials on [how to measure your site's performance](https://developers.google.com/web/tools/chrome-devtools/speed/get-started).

### Add Feature Policy to your site

If you want to take an extra step, [Feature Policy](https://developers.google.com/web/updates/2018/06/feature-policy) is a new web platform feature that can be a guardrail for you when you are building your project. Turning on feature policy guarantees the certain behavior of your app and prevents you from making mistakes. For example, If you want to ensure your app will never block the parsing, you can run your app on synchronous scripts policy. When `sync-script: 'none'` is enabled, parser-blocking JavaScript will be prevented from executing. This prevents any of your code from blocking the parser, and the browser doesn't need to worry about pausing the parser.

