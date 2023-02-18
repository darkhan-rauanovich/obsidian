
Можете ли вы ответить на вопрос о том, в чем заключается разница между [`requestAnimationFrame`](https://developer.mozilla.org/ru/docs/Web/API/window/requestAnimationFrame) и [`requestIdleCallback`](https://developer.mozilla.org/ru/docs/Web/API/Window/requestIdleCallback)?

tags: 

Если можете, то я завидую глубине ваших знаний. Я не смог, когда меня об этом спросили. Более того, в тот момент я даже не знал о существовании интерфейса `requestIdleCallback`. Теперь знаю и хочу с вами этими знаниями поделиться.

Сразу уточним, что названные интерфейсы предоставляются браузером и к [`ECMAScript`](https://www.ecma-international.org/publications-and-standards/standards/ecma-262/) отношения не имеют.

Что касается поддержки, то с `requestAnimationFrame` [все хорошо](https://caniuse.com/requestanimationframe), а с `requestIdleCallback`, в основном из-за `Safari`, этого современного `IE`, [ситуация хуже](https://caniuse.com/requestidlecallback).

Рассматриваемые интерфейсы позволяют разработчикам получать доступ к процессу рендеринга страницы. Также они очень тесно связаны с циклом событий (event loop) браузера.

В сети имеется большое количество замечательных материалов, посвященных рендерингу и циклу событий. Я не вижу смысла дублировать здесь эти материалы.

Что касается рендеринга, могу порекомендовать следующее (от абстрактного к конкретному):

-   [[Browser render]]  +
-   [[Inside modern browser]]  
-   [[Optimize JavaScript execution]]  +
-   [[Rendering Performance]] +
-   [[Using requestIdleCallback]] 

  

Также советую взглянуть на эти видео, посвященные циклу событий:

  

-   [Филипп Робертс: Что за чертовщина такая event loop? | JSConf EU 2014](https://www.youtube.com/watch?v=8aGhZQkoFbQ)
-   [Джейк Арчибальд. В цикле — JSConf.Asia](https://www.youtube.com/watch?v=cCOL7MC4Pl0)

  

Потратьте время на ознакомление с данными материалами, не пожалеете. Я подожду :)

  

Итак, что мы имеем в сухом остатке?

  

## `requestAnimationFrame`

  

**Метод `requestAnimationFrame`** предоставляет разработчикам доступ к жизненному циклу фрейма, позволяя выполнять операции перед вычислением стилей и формированием макета (layout) документа браузером. Вот почему данный метод отлично подходит для реализации анимации. Собственно, для этого он и предназначен.

  

Во-первых, он вызывается не чаще и не реже, чем браузер вычисляет макет (правильная частота). Во-вторых, он вызывается перед формированием макета (правильное время). Поэтому `rAF` также отлично подходит для внесения изменений в `DOM` или `CSSOM`. Он синхронизирован с [`vsync`](https://en.wikipedia.org/wiki/Vsync_(computing)), как и любой другой механизм рендеринга, используемый браузером.

  

Рассмотрим пример анимирования элемента с помощью `rAF`.

  

```
<div class="box_for_animation"></div>
<button class="button_for_animation">Start animation</button>
```

  

У нас имеется анимируемый контейнер и кнопка для запуска анимации.

  

Стили, с вашего позволения, я опущу, поскольку в них нет ничего особенного (в конце раздела будет ссылка на песочницу).

  

```
const animationBox = document.querySelector('.box_for_animation')
const animationButton = document.querySelector('.button_for_animation')

let animationStart
let requestId
```

  

Получаем ссылки на DOM-элементы и определяем глобальные переменные для времени начала анимации и идентификатора запроса.

  

```
function startAnimation() {
 requestId = window.requestAnimationFrame(animate)

 animationButton.style.opacity = 0
}
```

  

Определяем функцию для запуска анимации. `requestAnimationFrame` возвращает идентификатор запроса, который мы присваиваем созданной ранее переменной. Кнопка для запуска анимации после клика по ней плавно скрывается.

  

```
animationButton.addEventListener('click', startAnimation, { once: true })
```

  

Добавляем одноразовый обработчик события `click`.

  

```
function animate(timestamp) {
 if (!animationStart) {
   animationStart = timestamp
 }

 const progress = timestamp - animationStart

 animationBox.style.transform = `translateX(${progress / 5}px)`

 const x = animationBox.getBoundingClientRect().x + 100

 // 6px - scrollbar width
 if (x <= window.innerWidth - 6) {
   window.requestAnimationFrame(animate)
 } else {
   window.cancelAnimationFrame(requestId)
 }
}
```

  

Определяем функцию для анимирования контейнера. Функция принимает `timestamp` — время, прошедшее с начала выполнения запроса в мс. Анимирование заключается в постепенном сдвиге элемента до достижения им правой границы области просмотра. `100` — это ширина контейнера, а `6` — ширина панели прокрутки, установленные с помощью стилей. Когда значение координаты `x` элемента с учетом его ширины становится равным или больше значения ширины области просмотра, анимация отменяется.

  

Зацикливание анимации с помощью `rAF` часто используется при рисовании на `canvas`, например, при разработке 2D-игр.

  

`rAF` иногда также применяется для оптимизации обработчиков события `scroll`. Делается это следующим образом ([источник](https://developers.google.com/web/fundamentals/performance/rendering/debounce-your-input-handlers)):

  

```
let scheduledAnimationFrame

// читаем и обновляем страницу
function readAndUpdatePage(){
 console.log('read and update')

 scheduledAnimationFrame = false
}

function onScroll () {
 // сохраняем значение прокрутки для будущего использования
 const lastScrollY = window.scrollY

 // предотвращаем множественный вызов колбека, переданного `rAF`
 if (scheduledAnimationFrame) {
   return
 }

 scheduledAnimationFrame = true

 window.requestAnimationFrame(readAndUpdatePage)
}

window.addEventListener('scroll', onScroll)
```

  

В теории, использование данного паттерна откладывает выполнение операции `readAndUpdatePage`, как минимум, до следующего фрейма — действительно, изменять макет чаще, чем его рендерит браузер, не имеет смысла.

  

Однако индикатор `scheduledAnimationFrame` бесполезен, поскольку событие `scroll` возникает при рендеринге позиции прокрутки браузером. Это означает, что данное событие синхронизировано с рендерингом. По сути, это то, что делает `rAF` — позволяет синхронизировать запуск колбека с рендерингом страницы.

  

Вот как можно убедиться в бесполезности `scheduledAnimationFrame`:

  

```
if (scheduledAnimationFrame) {
 console.log('prevented rAF callback')
 return
}
```

  

Сообщение `prevented rAF callback` никогда не попадет в консоль. Следовательно, данный код является мертвым (dead code).

  

Для того чтобы получить максимальную выгоду от использования `rAF`, колбек должен запускаться под определенным условием.

  

Рассмотрим пример вывода сообщения о том, что элемент находится в области просмотра.

  

```
<p class="message_for_scroll"></p>
<div class="box_for_scroll"></div>
```

  

У нас имеется сообщение и контейнер, за нахождением которого в области просмотра мы будем следить при обработке события прокрутки.

  

```
const scrollMessage = document.querySelector('.message_for_scroll')
const scrollBox = document.querySelector('.box_for_scroll')
```

  

Получаем ссылки на DOM-элементы.

  

```
function showMessage() {
 // код функции будет выполняться только один раз
 if (!scrollMessage.textContent) {
   scrollMessage.textContent = 'scrollBox is in viewport'
   scrollMessage.style.opacity = 1
 }
}
```

  

Определяем функцию для отображения сообщения о том, что контейнер находится в области просмотра.

  

```
function onScroll() {
 const { top, bottom } = scrollBox.getBoundingClientRect()

 // если контейнер находится в области просмотра
 if (top < window.scrollY && bottom > 0) {
   // зацикливаем "анимацию"
   window.requestAnimationFrame(showMessage)

   // иначе, если имеется сообщение
 } else if (scrollMessage.textContent) {
   scrollMessage.style.opacity = 0

   // выполняем задержку для плавного скрытия сообщения
   const timerId = setTimeout(() => {
     scrollMessage.textContent = ''

     clearTimeout(timerId)
   }, 500)
 }
}

window.addEventListener('scroll', onScroll)
```

  

Определяем обработчик прокрутки и регистрируем его на объекте `window`.

  

Демо анимирования элемента и обработки прокрутки с помощью `rAF`:

  

  

## `requestIdleCallback`

  

**Метод `requestIdleCallback`** позволяет выполнять низкоприоритетные операции в период простоя браузера (отсюда `idle`) внутри фрейма (обычно, это происходит после вычисления браузером макета и его перерисовки, когда осталось какое-то время перед синхронизацией). Даже если с точки зрения пользователя страница "подвисает", могут быть периоды, когда браузер находится в режиме ожидания. Максимальная продолжительность времени, формально предоставляемая `rIC` для выполнения задачи, составляет 50 мс. Фактически же в нашем распоряжении имеется всего 0.5-10 мс. Поэтому, если внутри `rIC` вызывается функция для изменения `DOM`, ее следует вызывать с помощью `rAF`. Это объясняется тем, что модификация `DOM` — это потенциально продолжительная операция, на выполнение которой в `rIC` может не хватить времени.

  

Обработку события прокрутки вполне можно отнести к низкоприоритетным задачам. Поэтому для задержки вызова таких обработчиков можно использовать `rIC`. При этом, для реализации дополнительной задержки можно использовать `setTimeout`. [Здесь](https://github.com/aFarkas/lazysizes/blob/57e719f873ed552e43c877c2c8f8deb0364d6c6e/lazysizes-umd.js#L216-L240) можно найти примеры реализации `debounce` и `throttle` с помощью `rIC` и `setTimeout`.

  

Рассмотрим более интересный пример: совместное использование `rIC` и `rAF` для выполнения низкоприоритетной, но потенциально продолжительной задачи — "пакетному" рендерингу новых `DOM-элементов`.

  

```
<div class="buttons">
 <button data-type="square" class="button">Create square</button>
 <button data-type="polygon" class="button">Create polygon</button>
 <button data-type="circle" class="button">Create circle</button>
 <button data-type="render" class="button">Render shapes</button>
</div>

<div class="stat">
 <p>Squares: <span class="counter" data-for="square">0</span></p>
 <p>Polygons: <span class="counter" data-for="polygon">0</span></p>
 <p>Circles: <span class="counter" data-for="circle">0</span></p>
</div>
```

  

У нас имеются кнопки для создания "виртуальных" фигур (квадрата, многоугольника и круга) и рендеринга этих фигур, а также статистика виртуальных фигур.

  

```
// ссылки на DOM-элементы
const shapeButtons = document.querySelector('.buttons')
const statBox = document.querySelector('.stat')

// поисковая таблица для значений счетчиков
const counterByShape = {
 square: statBox.querySelector("[data-for='square']"),
 polygon: statBox.querySelector("[data-for='polygon']"),
 circle: statBox.querySelector("[data-for='circle']")
}

// см. ниже
let nextUnitOfWork = null
let shapesToRender = []
let render = false
let randomShape = true

// поисковая таблица для определения следующей (произвольной) фигуры
const randomShapeMap = {
 square: 'polygon',
 polygon: 'circle',
 circle: 'square'
}
```

  

Создаем 4 глобальных переменных:

  

-   `nextUniOfWork` — следующая единица работы: задача, которая будет выполняться браузером в период простоя
-   `shapesToRender` — хранилище для виртуальных фигур
-   `render` — индикатор начала рендеринга виртуальных фигур
-   `randomShape` — индикатор произвольной фигуры

  

```
window.requestIdleCallback =
 window.requestIdleCallback ||
 function (handler) {
   const start = Date.now()

   return setTimeout(() => {
     handler({
       didTimeout: false,
       timeRemaining: () => Math.max(0, 50 - (Date.now() - start))
     })
   }, 1)
 }
```

  

Поскольку поддержка `requestIdleCallback` оставляет желать лучшего, нам необходим такой `shim`. Это не `polyfill` — настоящий `rIC` работает немного иначе.

  

```
function workLoop(deadline) {
 while (nextUnitOfWork && deadline.timeRemaining() > 0) {
   nextUnitOfWork = performUnitOfWork(nextUnitOfWork)
 }

 if (!nextUnitOfWork && render) {
   window.requestAnimationFrame(updateDom)
 }

 window.requestIdleCallback(workLoop)
}

window.requestIdleCallback(workLoop)
```

  

С помощью 2 `requestIdleCallback` мы создаем бесконечный цикл `workLoop`. В колбеке выполняется 2 проверки:

  

-   если имеется следующая единица работы (`nextUnitOfWork`) и у браузера есть время на ее выполнение (`deadline.timeRemaining() > 0`), вызывается функция `performUnitOfWork`, выполняющая задачу и возвращающая следующую задачу (при наличии таковой)
-   если все единицы работы выполнены и индикатор рендеринга имеет значение `true`, с помощью `rAF` вызывается функция `updateDom`, выполняющая рендеринг виртуальных элементов

  

```
function performUnitOfWork(type) {
 const shape = document.createElement('div')
 shape.className = `shape ${type}`
 shapesToRender.push(shape)

 counterByShape[type].textContent =
   Number(counterByShape[type].textContent) + 1

 if (randomShape) {
   randomShape = false
   return randomShapeMap[type]
 }

 return null
}
```

  

В функции `performUnitOfWork` на основе типа (`type`) создается та или иная фигура, которая не рендерится сразу, а помещается в хранилище (`shapesToRender`). Затем обновляется значение соответствующего счетчика (мы можем позволить себе выполнение этой операции, поскольку уверены в ее "легкости" с точки зрения производительности и времени выполнения). Наконец, в качестве следующей единицы работы возвращается тип произвольной фигуры. _Обратите внимание_, что тип произвольной фигуры возвращается один раз для каждой "пользовательской" фигуры.

  

```
function updateDom() {
 const shapeBox = document.createElement('div')
 shapeBox.className = 'shapes'

 shapesToRender.forEach((el, i) => {
   el.classList.add('show')
   el.style.animationDelay = i * 0.5 + 's'

   shapeBox.append(el)
 })

 document.body.append(shapeBox)

 Object.values(counterByShape).forEach((counter) => {
   counter.textContent = '0'
 })

 shapesToRender = []
 render = false
}
```

  

В функции `updateDom` создается контейнер, в который помещаются фигуры из хранилища, и который затем рендерится в теле документа. Значения счетчиков обнуляются. Хранилище очищается. Индикатору начала рендеринга присваивается значение `false`.

  

```
shapeButtons.addEventListener('click', (e) => {
 const { type } = e.target.dataset
 if (!type) return

 if (type === 'render') {
   render = true
   return
 }

 randomShape = true
 nextUnitOfWork = type
})
```

  

В обработчике нажатия кнопки мы получаем тип из атрибута `data-type`. Если типом является `render`, индикатору начала рендеринга присваивается значение `true`. Это приводит к вызову `updateDom` с помощью `rAF` внутри `workLoop`, зацикленной с помощью `rIC`. Иначе индикатору произвольной фигуры присваивается значение `true`, а следующей единице работы — тип пользовательской фигуры.

  

Таким образом, мы имеем следующий `flow`:

  

-   у пользователя есть возможность генерировать любое количество виртуальных фигур без ущерба для производительности приложения (представим, что вместо фигур у нас выполняются "тяжелые" задачи, что может плохо повлиять на пользовательский опыт за счет снижения интерактивности страницы)
-   рендеринг виртуальных фигур полностью контролируется пользователем — при нажатии кнопки `Render` пользователь осознает возможные негативные последствия, о которых говорилось выше, и готов к ним (снижение производительности является ожидаемым)
-   браузер выполняет создание виртуальных фигур, только если у него имеется такая возможность

  

Тонкий момент: в качестве второго опционального параметра `rIC` принимает объект с настройками. Единственной доступной на сегодняшний день настройкой является `timeout`:

  

```
requestIdleCallback(callback, { timeout: 1000 })
```

  

Эта настройка позволяет гарантировать запуск колбека по истечение указанного времени (в мс), даже если браузер при этом не находится в режиме ожидания. Проверяется это следующим образом:

  

```
while (nextUnitOfWork && deadline.timeRemaining() > 0 || deadline.didTimeout) {
 nextUnitOfWork = performUnitOfWork(nextUnitOfWork)
}
```

  

В данном случае по истечении 1 сек с момента вызова `rIC` свойство `didTimeout` получит значение `true`. Это приведет к принудительному запуску колбека.

  

Обратной стороной медали является то, что принудительный вызов колбека может произойти в неподходящее время, например, когда у браузера имеется более важная задача, такая как обработка пользовательского ввода или плавное анимирование элемента.

  

Учитывая частоту проверок (60 раз в секунду или 1 раз в 16,67 мс), вероятность того, что запуск колбека будет отложен на ощутимое для пользователя время, практически исключается. Этим объясняется отсутствие `timeout` в нашем примере.

  

Демо приложения:

  

  

Пожалуй, это все, чем я хотел поделиться с вами в данной статье.

  

Благодарю за внимание и хорошего дня!