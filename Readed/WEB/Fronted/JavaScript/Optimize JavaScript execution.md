-   Avoid setTimeout or setInterval for visual updates; always use requestAnimationFrame     instead.
-   Move long-running JavaScript off the main thread to Web Workers.
-   Use micro-tasks to make DOM changes over several frames.
-   Use Chrome DevTools’ Timeline and JavaScript Profiler to assess the impact of JavaScript.


Use [[requestAnimationFrame]]  

The only way to guarantee that your JavaScript will run at the start of a frame is to use `requestAnimationFrame`.

![[iq5yVSd4wRskoD8GywR7.jpg]]


## Reduce complexity or use Web Workers

[[web worker]] 

```js
var dataSortWorker = new Worker("sort-worker.js");  
dataSortWorker.postMesssage(dataToSort);  
  
// The main thread is now free to continue working on other things...  
  
dataSortWorker.addEventListener('message', function(evt) {  
	var sortedData = evt.data;  
	// Update data on screen...  
});
```



## Avoid micro-optimizing your JavaScript [#](https://web.dev/optimize-javascript-execution/#avoid-micro-optimizing-your-javascript)

It may be cool to know that the browser can execute one version of a thing 100 times faster than another thing, like that requesting an element’s `offsetTop` is faster than computing `getBoundingClientRect()`, but it’s almost always true that you’ll only be calling functions like these a small number of times per frame, so it’s normally wasted effort to focus on this aspect of JavaScript’s performance. You'll typically only save fractions of milliseconds.

If you’re making a game, or a computationally expensive application, then you’re likely an exception to this guidance, as you’ll be typically fitting a lot of computation into a single frame, and in that case everything helps.

In short, you should be very wary of micro-optimizations because they won’t typically map to the kind of application you’re building.