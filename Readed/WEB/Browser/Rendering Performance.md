Optimize rendering perfomance

Users of today’s web [expect that the pages they visit will be interactive and smooth](https://paul.kinlan.me/what-news-readers-want/) and that’s where you need to increasingly focus your time and effort. Pages should not only load quickly, but also run well; scrolling should be stick-to-finger fast, and animations and interactions should be silky smooth.

Each of those frames has a budget of just over 16ms (1 second / 60 = 16.66ms). In reality, however, the browser has maintenance work to do, so all of your work needs to be completed inside **10ms**. When you fail to meet this budget the frame rate drops, and the content judders on screen. This is often referred to as **jank**, and it negatively impacts the user's experience.

## The pixel pipeline

There are five major areas that you need to know about and be mindful of when you work. They are areas you have the most control over, and key points in the pixels-to-screen pipeline:

![[gfQC4uOnIbLVtkdvbSY4.jpg]]

-   **JavaScript**. Typically JavaScript is used to handle work that will result in visual changes, whether it’s jQuery’s `animate` function, sorting a data set, or adding DOM elements to the page. It doesn’t have to be JavaScript that triggers a visual change, though: CSS Animations, Transitions, and the Web Animations API are also commonly used.
-   **Style calculations**. This is the process of figuring out which CSS rules apply to which elements based on matching selectors, for example, `.headline` or `.nav > .nav__item`. From there, once rules are known, they are applied and the final styles for each element are calculated.
-   **Layout**. Once the browser knows which rules apply to an element it can begin to calculate how much space it takes up and where it is on screen. The web’s layout model means that one element can affect others, for example the width of the `<body>` element typically affects its children’s widths and so on all the way up and down the tree, so the process can be quite involved for the browser.
-   **Paint**. Painting is the process of filling in pixels. It involves drawing out text, colors, images, borders, and shadows, essentially every visual part of the elements. The drawing is typically done onto multiple surfaces, often called layers.
-   **Compositing**. Since the parts of the page were drawn into potentially multiple layers they need to be drawn to the screen in the correct order so that the page renders correctly. This is especially important for elements that overlap another, since a mistake could result in one element appearing over the top of another incorrectly.

You won’t always necessarily touch every part of the pipeline on every frame. In fact, there are three ways the pipeline _normally_ plays out for a given frame when you make a visual change, either with JavaScript, CSS, or Web Animations:

### 1. JS / CSS > Style > Layout > Paint > Composite [#](https://web.dev/rendering-performance/#1-js-css-greater-style-greater-layout-greater-paint-greater-composite)

![The full pixel pipeline.](https://web-dev.imgix.net/image/T4FyVKpzu4WKF1kBNvXepbi08t52/gfQC4uOnIbLVtkdvbSY4.jpg?auto=format)

If you change a “layout” property, so that’s one that changes an element’s geometry, like its width, height, or its position with left or top, the browser will have to check all the other elements and “reflow” the page. Any affected areas will need to be repainted, and the final painted elements will need to be composited back together.

### 2. JS / CSS > Style > Paint > Composite [#](https://web.dev/rendering-performance/#2-js-css-greater-style-greater-paint-greater-composite)

![The  pixel pipeline without layout.](https://web-dev.imgix.net/image/T4FyVKpzu4WKF1kBNvXepbi08t52/aD7UKMbegndQijj36PHi.jpg?auto=format)

If you changed a “paint only” property, like a background image, text color, or shadows, in other words one that does not affect the layout of the page, then the browser skips layout, but it will still do paint.

### 3. JS / CSS > Style > Composite [#](https://web.dev/rendering-performance/#3-js-css-greater-style-greater-composite)

![The pixel pipeline without layout or paint.](https://web-dev.imgix.net/image/T4FyVKpzu4WKF1kBNvXepbi08t52/bfBPdciP9OSYPFI67xiY.jpg?auto=format)

If you change a property that requires neither layout nor paint, and the browser jumps to just do compositing.

This final version is the cheapest and most desirable for high pressure points in an app's lifecycle, like animations or scrolling.