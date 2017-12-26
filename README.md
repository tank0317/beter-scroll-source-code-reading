# better-scroll 源码阅读笔记

better-scroll 是一款重点解决移动端（未来可能会考虑 PC 端）各种滚动场景需求的插件。由于工作中需要使用better-scroll进行开发，然而better-scroll有着非常多的配置选项和可配置的功能，因此希望通过阅读它的源码，从而在开发中能够得心应手使用。

话不多说直接进入正题。

先看一下源码的目录结构：

![better-scroll-project-structure](http://om0jxp12h.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-12-26%20%E4%B8%8B%E5%8D%885.31.10.png)

`example`文件夹下是项目提供的demo，对于刚开始使用better-scroll的同学非常有用。而我们主要关注`src`目录下的文件。其中`src/index.js`是代码的入口文件，`src/scroll`文件夹下是项目的核心代码，`src/util`即项目的中使用的工具方法或常量设置。

## 入口文件

我们先来看下入口文件`src/index.js`。这个文件比较小，直接拿过来，内容如下。

```javascript
import { eventMixin } from './scroll/event'
import { initMixin } from './scroll/init'
import { coreMixin } from './scroll/core'
import { snapMixin } from './scroll/snap'
import { wheelMixin } from './scroll/wheel'
import { scrollbarMixin } from './scroll/scrollbar'
import { pullDownMixin } from './scroll/pulldown'
import { pullUpMixin } from './scroll/pullup'

import { warn } from './util/debug'

//创建better-scroll的构造函数
function BScroll(el, options) {
  this.wrapper = typeof el === 'string' ? document.querySelector(el) : el
  if (!this.wrapper) {
    warn('can not resolve the wrapper dom')
  }
  this.scroller = this.wrapper.children[0]
  if (!this.scroller) {
    warn('the wrapper need at least one child element to be scroller')
  }
  // cache style for better performance
  this.scrollerStyle = this.scroller.style

  this._init(el, options)
}

//将分散的方法都挂载到BScroll上。
initMixin(BScroll)
coreMixin(BScroll)
eventMixin(BScroll)
snapMixin(BScroll)
wheelMixin(BScroll)
scrollbarMixin(BScroll)
pullDownMixin(BScroll)
pullUpMixin(BScroll)

BScroll.Version = '1.6.3'

export default BScroll
```

可以看到入口文件主要就做了一件事情，创建`BScroll`构造函数。构造函数内部设置了容器`wrapper`和可滚动元素`scroller`后就进入了`this.init(el, options)`方法。这个`init()`方法定义在`src/sroll/init.js`文件中，通过`initMixin(BScrol)`方法把`init()`等方法挂载到`BScroll`构造函数原型上。类似的操作还有，`coreMinxin(BScrol)`（滚动相关的核心代码）, `eventMinxin(BScrol)`（实现自定义事件）等，。可以看到作者把不同的功能放在的分立的文件中，像插件一样把各个功能添加到`better-scroll`的主体上。**今天的任务就是，弄清楚`better-scroll`的初始化过程都做了哪些事情，即`this.init(el, options)`的执行过程。**

## better-scroll的初始化过程

先大致看下初始化方法的代码：

```javascript
BScroll.prototype._init = function (el, options) {
  this._handleOptions(options) //处理配置参数

  // init private custom events
  this._events = {} //初始化自定义事件缓存，自定义事件的实现在src/scroll/event.js中

  this.x = 0 //scroll 横轴坐标。
  this.y = 0 //scroll 纵轴坐标。
  this.directionX = 0 //判断 scroll 滑动结束后相对于开始滑动位置的方向（左右）。-1 表示从左向右滑，1 表示从右向左滑，0 表示没有滑动。
  this.directionY = 0 //判断 scroll 滑动结束后相对于开始滑动位置的方向（上下）。-1 表示从左向下滑，1 表示从右向上滑，0 表示没有滑动。

  this._addDOMEvents() //添加原生事件监听

  this._initExtFeatures() //根据配置参数初始化betterScroll的额外功能

  this._watchTransition() //监听是否处于过渡状态

  if (this.options.observeDOM) { 
    this._initDOMObserver() //会检测 scroller 内部 DOM 变化，自动调用 refresh 方法重新计算来保证滚动的正确性。
  }

  this.refresh() // 重新计算容器和可滚动元素（this.scroller)宽高，可滚动最大距离，并重置滚动元素的位置

  if (!this.options.snap) {
    this.scrollTo(this.options.startX, this.options.startY) // 滚动到初始位置
  }

  this.enable() //开启滚动功能
}
```

初始化过程中主要做了以下工作：

1. **处理配置参数**
2. 初始化自定义事件缓存
3. **添加原生事件监听回调**
4. 根据配置参数初始化better-scroll的额外功能
5. 调用_watchTransition()，监听是否处于滚动过渡动画状态，做相应操作
6. 初始化DOMObserver，检测 scroller 内部 DOM 变化，自动调用 refresh 方法
7. **调用refresh()方法，重新计算容器和可滚动元素（this.scroller)宽高，可滚动最大距离，并重置滚动元素的位置**
8. 滚动到初始位置（默认情况下，snap配置为false，我们这里关注默认状态）
9. 开启滚动功能。

我们前两篇文章只关注better-scroll实现滚动过程的核心功能。所以这里我们重点关注1、3、7的过程，其他部分会简单带过，以后再具体分析。

### 处理配置参数

`_handleOptions()`过程，主要初始化配置参数。合并默认配置和用户配置项，判断浏览器是否支持CSS3的`transition`, `transform`属性，另外可以看到`eventPassthrough`属性会影响到其他配置。相关的一些配置项的含义可以查看better-scroll[文档](https://ustbhuangyi.github.io/better-scroll/doc/zh-hans/#better-scroll %E6%98%AF%E4%BB%80%E4%B9%88)

```javascript
BScroll.prototype._handleOptions = function (options) {
  this.options = extend({}, DEFAULT_OPTIONS, options) //将默认参数与用户配置合并
   
  //HWCompositing 是否开启硬件加速，开启它会在 scroller 上添加 translateZ(0) 来开启硬件加速从而提升动画性能，有很好的滚动效果。
  this.translateZ = this.options.HWCompositing && hasPerspective ? ' translateZ(0)' : '' //暂时不关心
  
  //是否支持transition transform属性
  this.options.useTransition = this.options.useTransition && hasTransition
  this.options.useTransform = this.options.useTransform && hasTransform
  
  //是否是否阻止浏览器默认行为，eventPassthrough preventDefault属性涵义可以查看[文档](https://ustbhuangyi.github.io/better-scroll/doc/zh-hans/options.html#preventdefault)
  this.options.preventDefault = !this.options.eventPassthrough && this.options.preventDefault

  // 如果启用eventPassthrough，配置相应滚动方向
  this.options.scrollX = this.options.eventPassthrough === 'horizontal' ? false : this.options.scrollX
  this.options.scrollY = this.options.eventPassthrough === 'vertical' ? false : this.options.scrollY

  // With eventPassthrough we also need lockDirection mechanism
  this.options.freeScroll = this.options.freeScroll && !this.options.eventPassthrough
  this.options.directionLockThreshold = this.options.eventPassthrough ? 0 : this.options.directionLockThreshold

  if (this.options.tap === true) { //暂不关心
    this.options.tap = 'tap'
  }
}
```

这里有些属性可能还不清楚具体作用，暂时略过，只需要知道这里只是做初始化配置参数。

### 添加原生事件监听回调

先看`_addDOMEvents`内容：

```javascript
BScroll.prototype._addDOMEvents = function () {
  let eventOperation = addEvent
  this._handleDOMEvents(eventOperation)
}
```
在看`addEvent`和`_handleDOMEvents`，由于这里`eventOperation`为`addEvent`，所以`_handleDOMEvents`主要是调用了`addEventListener`添加事件监听回调。
```javascript
export function addEvent(el, type, fn, capture) {
  el.addEventListener(type, fn, {passive: false, capture: !!capture})
}
```
```javascript
BScroll.prototype._handleDOMEvents = function (eventOperation) {
  let target = this.options.bindToWrapper ? this.wrapper : window
  eventOperation(window, 'orientationchange', this)
  eventOperation(window, 'resize', this)

  if (this.options.click) {
    eventOperation(this.wrapper, 'click', this, true)
  }

  if (!this.options.disableMouse) {
    eventOperation(this.wrapper, 'mousedown', this)
    eventOperation(target, 'mousemove', this)
    eventOperation(target, 'mousecancel', this)
    eventOperation(target, 'mouseup', this)
  }

  if (hasTouch && !this.options.disableTouch) {
    eventOperation(this.wrapper, 'touchstart', this)
    eventOperation(target, 'touchmove', this)
    eventOperation(target, 'touchcancel', this)
    eventOperation(target, 'touchend', this)
  }

  eventOperation(this.scroller, style.transitionEnd, this)
}
```
值得注意的是，这里`eventOperation(window, 'orientationchange', this)`的第三个参数是`this`，这是怎么回事？这里不应该是一个函数吗？实际上，查看[addEventListener](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget/addEventListener)的介绍，可以得到，这里不仅可以是一个函数，还可以是一个对象，只要这个对象提供了一个`handleEvent`属性即可。相应的我们可以在`src/scroll/init.js`文件中找到`BScroll`原型下`handleEvent`的定义。
```javascript
  BScroll.prototype.handleEvent = function (e) {
    switch (e.type) {
      case 'touchstart':
      case 'mousedown':
        this._start(e)
        break
      case 'touchmove':
      case 'mousemove':
        this._move(e)
        break
      case 'touchend':
      case 'mouseup':
      case 'touchcancel':
      case 'mousecancel':
        this._end(e)
        break
      case 'orientationchange':
      case 'resize':
        this._resize()
        break
      case 'transitionend':
      case 'webkitTransitionEnd':
      case 'oTransitionEnd':
      case 'MSTransitionEnd':
        this._transitionEnd(e)
        break
      case 'click':
        if (this.enabled && !e._constructed) {
          if (!preventDefaultException(e.target, this.options.preventDefaultException)) {
            e.preventDefault()
            e.stopPropagation()
          }
        }
        break
    }
  }
```

可以看到这里，定义了不同事件下相应的处理函数。这些处理函数是`better-scroll`实现的核心代码，也是我们下一篇文章分析的关键内容。然后有一个问题是**`addEventListener`中不是函数而是提供一个具有handleEvent方法的对象，这么做的好处是什么？**我们知道`addEventListener`中如果提供的是一个函数，那么使用`removeEventListener`清除这个事件监听时要提供一个完全相同的函数才可以。因为我们这里"add"了很多事件回调，因此在最终清除操作的时候，仅仅是因为每个事件的处理函数不同，我们要重复写很多个"remove"的方法。而如果提供的是一个对象的话，即上面的做法，那么清除事件监听时，只需要把`addEventListener`改为`removeEventListener`即可，因此`_handleDOMEvents`把这个操作写成了一个变量，从而在清除事件监听时，可以重用`_handleDOMEvents`的代码。我们可以在找到相应的`_removeDOMEvents`和`removeEvent`方法。

```javascript
  BScroll.prototype._removeDOMEvents = function () {
    let eventOperation = removeEvent
    this._handleDOMEvents(eventOperation)
  }
```
```javascript
export function removeEvent(el, type, fn, capture) {
  el.removeEventListener(type, fn, {passive: false, capture: !!capture})
}
```

在`src/scroll/core.js`中的`destory()`方法总会看到清除事件监听的操作。当然我们这里暂时不关心，先了解一下。

```javascript
  BScroll.prototype.destroy = function () {
    this.destroyed = true
    this.trigger('destroy')

    this._removeDOMEvents() //清除所有事件监听
    // remove custom events
    this._events = {}
  }
```

### `refresh()方法`

`refresh()`方法是一个很重要的方法。作用是重新计算 better-scroll，当 DOM 结构发生变化的时候务必要调用确保滚动的效果正常。原因可以从代码中看到，它主要做了：获取容器和滚动元素的宽高，计算最大滚动距离。因此DOM结构变化后应该调用重新计算。 
```javascript
  BScroll.prototype.refresh = function () {
    let wrapperRect = getRect(this.wrapper) //获取容器的宽高
    this.wrapperWidth = wrapperRect.width
    this.wrapperHeight = wrapperRect.height

    let scrollerRect = getRect(this.scroller) //获取滚动元素的宽高
    this.scrollerWidth = scrollerRect.width
    this.scrollerHeight = scrollerRect.height

    const wheel = this.options.wheel
    if (wheel) {
      ... //暂时不关心
    } else {
      this.maxScrollX = this.wrapperWidth - this.scrollerWidth //计算最大的横向滚动距离
      this.maxScrollY = this.wrapperHeight - this.scrollerHeight //计算最大的纵向滚动距离
    }

    this.hasHorizontalScroll = this.options.scrollX && this.maxScrollX < 0 //是否可以横向滚动
    this.hasVerticalScroll = this.options.scrollY && this.maxScrollY < 0 //是否可以纵向滚动

    if (!this.hasHorizontalScroll) { 
      this.maxScrollX = 0
      this.scrollerWidth = this.wrapperWidth
    }

    if (!this.hasVerticalScroll) {
      this.maxScrollY = 0
      this.scrollerHeight = this.wrapperHeight
    }

    //初始化内部使用参数
    this.endTime = 0 //滚动结束时间
    this.directionX = 0 //横向滚动方向， 0表示没有滚动
    this.directionY = 0 //纵向滚动方向， 0表示没有滚动
    this.wrapperOffset = offset(this.wrapper) 

    this.trigger('refresh') //触发"refresh"事件

    this.resetPosition() //重置滚动元素位置
  }
```
这里需要注意的是，在web页面中，坐标系的原点在左上角。即如果向上滑动，滑动的距离是一个负值。因此`maxScrollX`和`maxScrollY`为负值时，表示可以滚动。

![default_grid](http://om0jxp12h.bkt.clouddn.com/Canvas_default_grid.png)

然后看一下`resetPosition()`方法做了哪些事情。

```javascript
  BScroll.prototype.resetPosition = function (time = 0, easeing = ease.bounce) {
    let x = this.x
    let roundX = Math.round(x)
    if (!this.hasHorizontalScroll || roundX > 0) { //如果没有横向滚动或者向右滚动了
      x = 0
    } else if (roundX < this.maxScrollX) { //如果向左滚动了
      x = this.maxScrollX
    }

    let y = this.y
    let roundY = Math.round(y)
    if (!this.hasVerticalScroll || roundY > 0) { //如果没有纵向滚动或者向下滚动了
      y = 0
    } else if (roundY < this.maxScrollY) { //如果向上滚动了
      y = this.maxScrollY
    }

    if (x === this.x && y === this.y) {
      return false
    }

    this.scrollTo(x, y, time, easeing) //滚动到指定x, y的位置，time和easeing指定过渡时间和动画

    return true
  }
```
可以看到`resetPosition()`方法只做了一件事情，调用`scrollTo()`方法滚动到指定位置，这个位置要么是原始位置`x=0 || y=0`，即没有任何滚动的状态，要么是滚动到最底端或最右端`x=this.maxScrollX || y=this.maxScrollY`。而这个位置根据一定规则设置, 具体看上面的注释。

实际上上面的过程已经完成了我们今天的任务，我们已经大致看完了better-scroll的初始化过程。下一篇文章我们将从`scrollTo()`方法的实现开始，然后分析better-scroll的核心代码，滚动的开始，滚动ing,滚动结束这三个时间点都做了哪些事情。这里暂时给出`scrollTo()`的实现，大致看一下内容。

```javascript
  BScroll.prototype.scrollTo = function (x, y, time = 0, easing = ease.bounce) {
    this.isInTransition = this.options.useTransition && time > 0 && (x !== this.x || y !== this.y)

    if (!time || this.options.useTransition) {
      this._transitionProperty()
      this._transitionTimingFunction(easing.style)
      this._transitionTime(time)
      this._translate(x, y) // 这一句实现了滚动，我们下次再说。

      if (time && this.options.probeType === 3) {
        this._startProbe()
      }

      if (this.options.wheel) {
        ... //暂不关心
      }
    } else {
      this._animate(x, y, time, easing.fn)
    }
  }
```
