# nextTick

<!-- TOC -->

- [nextTick](#nexttick)
    - [定义](#定义)
    - [分析](#分析)
        - [timerFunc](#timerfunc)
        - [flushCallbacks](#flushcallbacks)
        - [nextTick实现](#nexttick实现)
    - [实现一个基本的nextTick](#实现一个基本的nexttick)

<!-- /TOC -->

## 定义

在下次 DOM 更新循环结束之后执行延迟回调。在修改数据之后立即使用这个方法，获取更新后的 DOM。

例子：

```JS
// 修改数据
vm.msg = 'Hello'
// DOM 还没有更新
Vue.nextTick(function () {
  // DOM 更新了
})

// 作为一个 Promise 使用 (2.1.0 起新增，详见接下来的提示)
Vue.nextTick()
  .then(function () {
    // DOM 更新了
  })
```

> 2.1.0 起新增：如果没有提供回调且在支持 Promise 的环境中，则返回一个 Promise。请注意 Vue 不自带 Promise 的 polyfill，所以如果你的目标浏览器不原生支持 Promise (IE：你们都看我干嘛)，你得自己提供 polyfill。

（IE： ????）

接下来分析，nextTick是如何保证DOM循环之后可以立即repaint的。

## 分析

在分析之前，我们需要知道event loop，这里简要说明，可以理解vue是如何使用的event loop的。

Event Loop分为宏任务（macrotask）以及微任务（microtask），在实现宏任务和微任务后，js会进入下一次tick循环，并在tick之间进行ui的绘制。

但是，宏任务执行时间比微任务的执行时间长，所以优先使用微任务；对于不同的宏任务，其执行效率也跟所处环境有关，需要根据浏览器支持情况，使用对应的宏任务。

常见的宏任务包括：I/O， setTimeout， setInterval， setImmediate， requestAnimationFrame。

常见的微任务包括：process.nextTick（node）， MutationObserver， Promise.then catch finally。

### timerFunc

在vue中，数据响应过程为：数据更改->通知watcher->更新DOM，数据变更可能发生在任何时候，且是实时的，但是render修改到DOM是异步的，我们要保证的是在实时修改DOM tree后，如何保证nextTick中的cb可以立即执行。

按照上面说的，在事件循环中，当前tick会执行完microtask队列在进行下一次循环，同时要注意，在执行microtask的过程中加入microtask队列的微任务，也会在下一次事件循环之前被执行。这也就是说在修改DOM tree的microtask执行后，我们可以插入一个microtask，用来在当前的同步代码执行完毕后，执行我想执行的异步回调。

这里有一点要注意的是，vue也并不是在DOM tree变更后立即repaint，那样每次都同步改变DOM tree一点，像下面这样：

```JS
for(let i=0; i<100; i++){
    dom.style.left = i + 'px';
}
```

每次都会改变视图么？答案当然是不会的，vue会将修改的数据同步通知所有订阅的watcher，watcher会保留一个数组，直到所有的同步修改执行完后，这时候在执行对应的变动，避免了DOM消耗与render的效率。

那么，对于macrotask而言，microtask在修改同步操作后立即插入异步处理有什么好处呢？为什么不选用macrotask呢？

我们可以看下面这个[例子](https://github.com/vuejs/vue/issues/3771#issuecomment-249692588)

这里参考这两个jsFiddle[jsFiddle1](https://jsfiddle.net/k6bgu2z6/4/)和[jsFiddle2](https://jsfiddle.net/v9q9L0hw/2/)：

两个fiddle实现的功能完全一样，就是让那个绝对定位的黄色元素起到一个fixed定位的效果：绑定scroll事件，每次滚动的时候，计算当前滚动的位置并更改到那个绝对定位元素的top属性上去，尝试几下会发现第一个fiddle中的黄元素是稳定不动的。而后一个fiddle中会出现肉眼可见的抖动。

原因是第一个jsfiddle使用的版本是Vue 2.0.0-rc.6，这个版本的nextTick实现是采用了MutationObserver，而后因为IOS9.3的WebView里的MutationObserver有bug，就换而选择window.postMessage，但是window.postMessage是macrotask任务，这也是问题所在。

如果nextTick使用的是microtask，那么在task执行完毕之后就会立即执行所有microtask，那么flushBatcherQueue（真正修改DOM）便得以在此时立即完成，而后，当前轮次的microtask全部清理完成时，执行UI rendering，把重排重绘等操作真正更新到DOM上，如果nextTick使用的是macrotask，那么会在当前的task和所有microtask执行完毕之后才在以后的某一次task执行过程中处理flushBatcherQueue，那个时候才真正执行各个指令的修改DOM操作，但那时为时已晚，错过了多次触发重绘、渲染UI的时机。在这期间，浏览器内部为了更快的响应用户UI，内部可能是有多个task queue的，这就是为什么会是在以后的某一次task执行，而不是下一次执行。

> For example, a user agent could have one task queue for mouse and key events (the user interaction task source), and another for everything else. The user agent could then give keyboard and mouse events preference over other tasks three quarters of the time, keeping the interface responsive but not starving other task queues, and never processing events from any one task source out of order.

根据这个参考，我们就会发现，对于鼠标等移动事件，优先级是会有提升的，就会导致macrotask在下几次的task中执行。

对于nextTick而言，事件循环和repaint了解到这种程度就够了，这里不做深究。

根据上面得到的结论：在nextTick中，我们选择microtask执行回调队列。

接下来，我们看vue中nextTick是如何根据浏览器的能力来选择对应方式执行回调的：

```JS
// Here we have async deferring wrappers using microtasks.
// In 2.5 we used (macro) tasks (in combination with microtasks).
// However, it has subtle problems when state is changed right before repaint
// (e.g. #6813, out-in transitions).
// Also, using (macro) tasks in event handler would cause some weird behaviors
// that cannot be circumvented (e.g. #7109, #7153, #7546, #7834, #8109).
// So we now use microtasks everywhere, again.
// A major drawback of this tradeoff is that there are some scenarios
// where microtasks have too high a priority and fire in between supposedly
// sequential events (e.g. #4521, #6690, which have workarounds)
// or even between bubbling of the same event (#6566).

let timerFunc

// The nextTick behavior leverages the microtask queue, which can be accessed
// via either native Promise.then or MutationObserver.
// MutationObserver has wider support, however it is seriously bugged in
// UIWebView in iOS >= 9.3.3 when triggered in touch event handlers. It
// completely stops working after triggering a few times... so, if native
// Promise is available, we will use it:

/* istanbul ignore next, $flow-disable-line */
if (typeof Promise !== 'undefined' && isNative(Promise)) {
  const p = Promise.resolve()
  timerFunc = () => {
    p.then(flushCallbacks)
    // In problematic UIWebViews, Promise.then doesn't completely break, but
    // it can get stuck in a weird state where callbacks are pushed into the
    // microtask queue but the queue isn't being flushed, until the browser
    // needs to do some other work, e.g. handle a timer. Therefore we can
    // "force" the microtask queue to be flushed by adding an empty timer.
    if (isIOS) setTimeout(noop)
  }
  isUsingMicroTask = true
} else if (!isIE && typeof MutationObserver !== 'undefined' && (
  isNative(MutationObserver) ||
  // PhantomJS and iOS 7.x
  MutationObserver.toString() === '[object MutationObserverConstructor]'
)) {
  // Use MutationObserver where native Promise is not available,
  // e.g. PhantomJS, iOS7, Android 4.4
  // (#6466 MutationObserver is unreliable in IE11)
  let counter = 1
  const observer = new MutationObserver(flushCallbacks)
  const textNode = document.createTextNode(String(counter))
  observer.observe(textNode, {
    characterData: true
  })
  timerFunc = () => {
    counter = (counter + 1) % 2
    textNode.data = String(counter)
  }
  isUsingMicroTask = true
} else if (typeof setImmediate !== 'undefined' && isNative(setImmediate)) {
  // Fallback to setImmediate.
  // Technically it leverages the (macro) task queue,
  // but it is still a better choice than setTimeout.
  timerFunc = () => {
    setImmediate(flushCallbacks)
  }
} else {
  // Fallback to setTimeout.
  timerFunc = () => {
    setTimeout(flushCallbacks, 0)
  }
}
```

这里我们可以看到，我们依次选择Promise，MutationObserver，setImmediate和setTimeout。

其中，对于[MutationObserver](https://developer.mozilla.org/zh-CN/docs/Web/API/MutationObserver)而言，让textNode在0/1之间切换。在`const observer = new MutationObserver(flushCallbacks)`后，会得到一个MO实例，这个回调就会在MO实例监听到变动时触发。然后通过绑定到对应DOM，可以监听节点删除还是属性修改，通过调用observer方法就可以完成这一步。

以上就是nextTick的异步处理部分。

### flushCallbacks

接下来看其中的flushCallbacks是如何实现的：

```JS
const callbacks = []
let pending = false

function flushCallbacks () {
  pending = false
  const copies = callbacks.slice(0)
  callbacks.length = 0
  for (let i = 0; i < copies.length; i++) {
    copies[i]()
  }
}
```

其中，pending设为false，是为了保证在执行完后重置异步锁，可看下面nextTick的实现；设置copies是因为有的cb执行过程中又会往cb中加入内容，比如$nextTick的回调函数里又有$nextTick，这些是在接下来的循环中执行的。所以拷贝一份当前的,遍历执行完当前的即可,避免无休止的执行下去。

### nextTick实现

```JS
export function nextTick (cb?: Function, ctx?: Object) {
  let _resolve
  callbacks.push(() => {
    if (cb) {
      try {
        cb.call(ctx)
      } catch (e) {
        handleError(e, ctx, 'nextTick')
      }
    } else if (_resolve) {
      _resolve(ctx)
    }
  })
  if (!pending) {
    pending = true
    timerFunc()
  }
  // $flow-disable-line
  if (!cb && typeof Promise !== 'undefined') {
    return new Promise(resolve => {
      _resolve = resolve
    })
  }
}
```

其中，pending会判断当前是否执行了异步锁，如果异步锁未锁上，锁上异步锁，调用异步函数，这时会将异步插入当前循环尾部，并阻止其他异步函数，准备等同步函数执行完后，就开始执行回调函数队列。

最后，在没有回调并且支持Promise时，返回一个Promise。

## 实现一个基本的nextTick

根据上面的分析，手写一个最简单的nextTick如下：

```JS
const callbacks = []
let pending = false

function nextTick (cb) {
    callbacks.push(cb)

    if (!pending) {
        pending = true
        setTimeout(flushCallback, 0)
    }
}

function flushCallback () {
    pending = false
    let copies = callbacks.slice(0)
    callbacks.length = 0
    copies.forEach(cb => {
        cb()
    })
}
```

源码地址在[这里](https://github.com/vuejs/vue/blob/dev/src/core/util/next-tick.js)