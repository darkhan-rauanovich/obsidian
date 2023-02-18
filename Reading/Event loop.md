https://www.youtube.com/watch?v=8aGhZQkoFbQ

Callback

![[Screenshot_60.png]]

memory allocation

What Does Memory Allocation Mean? Memory allocation is **a process by which computer programs and services are assigned with physical or virtual memory space**. Memory allocation is the process of reserving a partial or complete portion of computer memory for the execution of programs and processes.

Call stack

![[Screenshot_1.png]]

callback queue 

the call stack 

one thread == one call stack == one thing at time

![[Screenshot_2.png]]

up to down execute stack function

![[Screenshot_3 1.png]]

![[Screenshot_4 1.png]]

stack переполнен

![[Screenshot_5.png]]

javaScritt однопоточный при запуске какого либо запроса мы блокируем главный поток и браузер зависает и ждет окончание функций и дальше не может выполнять больше никаких вычислений и операций.

Solution is: 
asynchronus callback

```js
console.log("hi");

setTimeout(function() {
	console.log("there");
}, 5000);

console.log("EcmaScript")
```

``` 
hi
EcmaScript

----5s later----

there
```

![[Screenshot_6.png]]


https://www.youtube.com/watch?v=cCOL7MC4Pl0

main thread

js, rendering execute in main thread 

race conditions - A race condition **occurs when two threads access a shared variable at the same time**. The first thread reads the variable, and the second thread reads the same value from the variable

https://betterways.dev/javascript-async-race-conditions

event loop running process rendering and js execute. If event loop execute infite task he stoping other task like rendering and execute infinite task

```html
<!DOCTYPE html>

<html lang="en">

<head>

  <meta charset="UTF-8">

  <meta http-equiv="X-UA-Compatible" content="IE=edge">

  <meta name="viewport" content="width=device-width, initial-scale=1.0">

  <title>Document</title>

</head>

<body>

  <main>

    <p>Lorem ipsum dolor sit amet consectetur, adipisicing elit. Obcaecati impedit molestiae voluptate asperiores quidem delectus quaerat, voluptas et officia libero porro repudiandae quae accusamus eius minima vitae facere neque aut!</p>

    <button id="button">Salam</button>

  </main>

  <script>

    const button = document.querySelector("#button");

  

    button.addEventListener("click", () => {

      while (true);

    })

  </script>

</body>

</html>
```


[[requestAnimationFrame]]

microtasks

[7:08](https://www.youtube.com/watch?v=cCOL7MC4Pl0&t=428s) Task Queues
[8:55](https://www.youtube.com/watch?v=cCOL7MC4Pl0&t=535s) render steps
[12:15](https://www.youtube.com/watch?v=cCOL7MC4Pl0&t=735s) hence the timeOut loop is not 'render blocking'.
[12:29](https://www.youtube.com/watch?v=cCOL7MC4Pl0&t=749s) but if you want to run code that has anything to do with rendering, a task is the wrong place to do it, coz it's on the 'opposite' side of the world from rendering, as far as the event loop is concerned.
[24:00](https://www.youtube.com/watch?v=cCOL7MC4Pl0&t=1440s) MicroTasks (RAF - RequestAnimationFrame finishes here )

