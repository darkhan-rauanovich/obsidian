Many sites and apps have a lot of scripts to execute. Your JavaScript often needs to be run as soon as possible, but at the same time you don’t want it to get in the user’s way. If you send analytics data when the user is scrolling the page, or you append elements to the DOM while they happen to be tapping on the button, your web app can become unresponsive, resulting in a poor user experience.

![[7eNdNvBkSvylDlf4cPGB.png]]

The good news is that there’s now an API that can help: `requestIdleCallback`. In the same way that adopting `requestAnimationFrame` allowed us to schedule animations properly and maximize our chances of hitting 60fps, `requestIdleCallback` will schedule work when there is free time at the end of a frame, or when the user is inactive. This means that there’s an opportunity to do your work without getting in the user’s way. It’s available as of Chrome 47, so you can give it a whirl today by using Chrome Canary! It is an _experimental feature_, and the spec is still in flux, so things could change in the future.

## Why should I use requestIdleCallback?

Scheduling non-essential work yourself is very difficult to do. It’s impossible to figure out exactly how much frame time remains because after `requestAnimationFrame` callbacks execute there are style calculations, layout, paint, and other browser internals that need to run. A home-rolled solution can’t account for any of those. In order to be sure that a user _isn’t_ interacting in some way you would also need to attach listeners to every kind of interaction event (`scroll`, `touch`, `click`), even if you don’t need them for functionality, _just_ so that you can be absolutely sure that the user isn’t interacting. The browser, on the other hand, knows exactly how much time is available at the end of the frame, and if the user is interacting, and so through `requestIdleCallback` we gain an API that allows us to make use of any spare time in the most efficient way possible.

Let’s take a look at it in a little more detail and see how we can make use of it.

## Checking for requestIdleCallback

It’s early days for `requestIdleCallback`, so before using it you should check that it’s available for use:

```js
if ('requestIdleCallback' in window) {    
	// Use requestIdleCallback to schedule work.
} else {    
	// Do what you’d do today.
}
```

You can also shim its behavior, which requires falling back to `setTimeout`:

```js
window.requestIdleCallback =
window.requestIdleCallback ||
function (cb) {
	var start = Date.now();
	return setTimeout(function () {
		cb({
			didTimeout: false,
			timeRemaining: function () {
				return Math.max(0, 50 - (Date.now() - start));
			}
		});
	}, 1);
}

window.cancelIdleCallback =
window.cancelIdleCallback ||
function (id) {
	clearTimeout(id);
}
```

Using `setTimeout` isn't great because it doesn't know about idle time like `requestIdleCallback` does, but since you would call your function directly if `requestIdleCallback` wasn't available, you are no worse off shimming in this way. With the shim, should `requestIdleCallback` be available, your calls will be silently redirected, which is great.

For now, though, let’s assume that it exists.

## Using requestIdleCallback

Calling `requestIdleCallback` is very similar to `requestAnimationFrame` in that it takes a callback function as its first parameter:

```js
requestIdleCallback(myNonEssentialWork);
```

When `myNonEssentialWork` is called, it will be given a `deadline` object which contains a function which returns a number indicating how much time remains for your work:

```js
function myNonEssentialWork (deadline) {
    while (deadline.timeRemaining() > 0)
    doWorkIfNeeded();
}
```

The `timeRemaining` function can be called to get the latest value. When `timeRemaining()` returns zero you can schedule another `requestIdleCallback` if you still have more work to do:

```js
function myNonEssentialWork (deadline) {
    while (deadline.timeRemaining() > 0 && tasks.length > 0)
    doWorkIfNeeded();

    if (tasks.length > 0)
    requestIdleCallback(myNonEssentialWork);
}
```

## Guaranteeing your function is called

What do you do if things are really busy? You might be concerned that your callback may never be called. Well, although `requestIdleCallback` resembles `requestAnimationFrame`, it also differs in that it takes an optional second parameter: an options object with **a timeout** property. This timeout, if set, gives the browser a time in milliseconds by which it must execute the callback:

```js
// Wait at most two seconds before processing events.
requestIdleCallback(processPendingAnalyticsEvents, { timeout: 2000 });
```

If your callback is executed because of the timeout firing you’ll notice two things:

-   `timeRemaining()` will return zero.
-   The `didTimeout` property of the `deadline` object will be true.

If you see that the `didTimeout` is true, you will most likely just want to run the work and be done with it:

```js
function myNonEssentialWork (deadline) {

    // Use any remaining time, or, if timed out, just run through the tasks.
    while ((deadline.timeRemaining() > 0 || deadline.didTimeout) &&
            tasks.length > 0)
    doWorkIfNeeded();

    if (tasks.length > 0)
    requestIdleCallback(myNonEssentialWork);
}
```

Because of the potential disruption this timeout can cause to your users (the work could cause your app to become unresponsive or janky) be cautious with setting this parameter. Where you can, let the browser decide when to call the callback.

## Using requestIdleCallback for sending analytics data

Let’s take a look using `requestIdleCallback` to send analytics data. In this case, we probably would want to track an event like -- say -- tapping on a navigation menu. However, because they normally animate onto the screen, we will want to avoid sending this event to Google Analytics immediately. We will create an array of events to send and request that they get sent at some point in the future:

```js
var eventsToSend = [];

function onNavOpenClick () {

    // Animate the menu.
    menu.classList.add('open');

    // Store the event for later.
    eventsToSend.push(
    {
        category: 'button',
        action: 'click',
        label: 'nav',
        value: 'open'
    });

    schedulePendingEvents();
}
```

Now we will need to use `requestIdleCallback` to process any pending events:

```js
function schedulePendingEvents() {

    // Only schedule the rIC if one has not already been set.
    if (isRequestIdleCallbackScheduled)
    return;

    isRequestIdleCallbackScheduled = true;

    if ('requestIdleCallback' in window) {
    // Wait at most two seconds before processing events.
    requestIdleCallback(processPendingAnalyticsEvents, { timeout: 2000 });
    } else {
    processPendingAnalyticsEvents();
    }
}
```

Here you can see I’ve set a timeout of 2 seconds, but this value would depend on your application. For analytics data, it makes sense that a timeout would be used to ensure data is reported in a reasonable timeframe rather than just at some point in the future.

Finally we need to write the function that `requestIdleCallback` will execute.

```js
function processPendingAnalyticsEvents (deadline) {

    // Reset the boolean so future rICs can be set.
    isRequestIdleCallbackScheduled = false;

    // If there is no deadline, just run as long as necessary.
    // This will be the case if requestIdleCallback doesn’t exist.
    if (typeof deadline === 'undefined')
    deadline = { timeRemaining: function () { return Number.MAX_VALUE } };

    // Go for as long as there is time remaining and work to do.
    while (deadline.timeRemaining() > 0 && eventsToSend.length > 0) {
    var evt = eventsToSend.pop();

    ga('send', 'event',
        evt.category,
        evt.action,
        evt.label,
        evt.value);
    }

    // Check if there are more events still to send.
    if (eventsToSend.length > 0)
    schedulePendingEvents();
}
```

For this example I assumed that if `requestIdleCallback` didn’t exist that the analytics data should be sent immediately. In a production application, however, it would likely be better to delay the send with a timeout to ensure it doesn’t conflict with any interactions and cause jank.

## Using requestIdleCallback to make DOM changes

Another situation where `requestIdleCallback` can really help performance is when you have non-essential DOM changes to make, such as adding items to the end of an ever-growing, lazy-loaded list. Let’s look at how `requestIdleCallback` actually fits into a typical frame.

![[i5IYAvSfMB8JIAelSAze.jpg]]

It’s possible that the browser will be too busy to run any callbacks in a given frame, so you shouldn’t expect that there will be _any_ free time at the end of a frame to do any more work. That makes it different to something like `setImmediate`, which _does_ run per frame.

If the callback _is_ fired at the end of the frame, it will be scheduled to go after the current frame has been committed, which means that style changes will have been applied, and, importantly, layout calculated. If we make DOM changes inside of the idle callback, those layout calculations will be invalidated. If there are any kind of layout reads in the next frame, e.g. `getBoundingClientRect`, `clientWidth`, etc, the browser will have to perform a [Forced Synchronous Layout](https://developers.google.com/web/fundamentals/performance/rendering/avoid-large-complex-layouts-and-layout-thrashing#avoid-forced-synchronous-layouts), which is a potential performance bottleneck.

Another reason not trigger DOM changes in the idle callback is that the time impact of changing the DOM is unpredictable, and as such we could easily go past the deadline the browser provided.

The best practice is to only make DOM changes inside of a `requestAnimationFrame` callback, since it is scheduled by the browser with that type of work in mind. That means that our code will need to use a document fragment, which can then be appended in the next `requestAnimationFrame` callback. If you are using a VDOM library, you would use `requestIdleCallback` to make changes, but you would _apply_ the DOM patches in the next `requestAnimationFrame` callback, not the idle callback.

So with that in mind, let’s take a look at the code:

```js
function processPendingElements (deadline) {

    // If there is no deadline, just run as long as necessary.
    if (typeof deadline === 'undefined')
    deadline = { timeRemaining: function () { return Number.MAX_VALUE } };

    if (!documentFragment)
    documentFragment = document.createDocumentFragment();

    // Go for as long as there is time remaining and work to do.
    while (deadline.timeRemaining() > 0 && elementsToAdd.length > 0) {

    // Create the element.
    var elToAdd = elementsToAdd.pop();
    var el = document.createElement(elToAdd.tag);
    el.textContent = elToAdd.content;

    // Add it to the fragment.
    documentFragment.appendChild(el);

    // Don't append to the document immediately, wait for the next
    // requestAnimationFrame callback.
    scheduleVisualUpdateIfNeeded();
    }

    // Check if there are more events still to send.
    if (elementsToAdd.length > 0)
    scheduleElementCreation();
}
```

Here I create the element and use the `textContent` property to populate it, but chances are your element creation code would be more involved! After creating the element `scheduleVisualUpdateIfNeeded` is called, which will set up a single `requestAnimationFrame` callback that will, in turn, append the document fragment to the body:

```js
function scheduleVisualUpdateIfNeeded() {

    if (isVisualUpdateScheduled)
    return;

    isVisualUpdateScheduled = true;

    requestAnimationFrame(appendDocumentFragment);
}

function appendDocumentFragment() {
    // Append the fragment and reset.
    document.body.appendChild(documentFragment);
    documentFragment = null;
}
```

