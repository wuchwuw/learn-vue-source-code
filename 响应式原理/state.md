## 响应式原理

熟悉Vue的都知道，Vue的响应式是通过Object.defineProperty来实现的。但是仅仅知道这个是不够的，我们需要了解，响应式数据是如何创建的，以及响应式数据的改变是如何触发组件重新渲染的。

这里先放一张Vue官网对响应式原理说明。
![响应式原理](../images/data.png)
从图中我们可以大致的了解整个响应式原理的过程，当组件创建的过程中，会把我们定义在组件上的data变成响应式对象，也就是通过Object.defineProperty为对象添加getter和setter(图中紫色圈圈部分)，当组件渲染时，就会访问到我们定义在template中的响应式数据(图中的"touch")，这时就会触发数据的getter从而收集依赖(图中的collect as Dependency),当我们对数据进行更改时，会触发setter，从而执行dep.notify通知watcher执行组件的重新渲染。
接下来我们就从响应式对象的创建开始，来分析Vue的响应式原理。

### 响应式对象的创建

在执行new Vue时，会执行_init()方法，在_init方法中，会调用initState(vm)，这个方法就是关于响应式对象的创建。

#### initState

    const opts = vm.$options
    if (opts.props) initProps(vm, opts.props)
    if (opts.methods) initMethods(vm, opts.methods)
    if (opts.data) {
      initData(vm)
    } else {
      observe(vm._data = {}, true /* asRootData */)
    }
    if (opts.computed) initComputed(vm, opts.computed)
    if (opts.watch && opts.watch !== nativeWatch) {
      initWatch(vm, opts.watch)
    }

这个方法很简单，就是分别拿到组件options的props、methods、data、computed、watch，分别执行它们对应的init方法，
这里我们先看initData。

#### initData

首先，拿到data并且校验了props和methods上的key是不是和data上有重复。校验之后调用了observe方法，就是通过这个方法来创建响应式对象的。

#### observe

observe方法主要做了几件事:

1、判断data上面是不是有__ob__这个对象，如果有则直接返回
2、判断data有没有符合成为响应式对象的条件

如果没有__ob__对象，并且符合，则通过Observer这个类来创建一个
observer

#### new Observer

    constructor (value: any) {
      this.value = value
      this.dep = new Dep()
      this.vmCount = 0
      def(value, '__ob__', this)
      if (Array.isArray(value)) {
        const augment = hasProto
          ? protoAugment
          : copyAugment
        augment(value, arrayMethods, arrayKeys)
        this.observeArray(value)
      } else {
        this.walk(value)
      }
    }

    walk () {...}

    observeArray () {...}

当创建类的实例时，就通过def这个方法往对象上面添加__ob__这个属性，def是对defineProperty的封装，这里主要是为了将__ob__这个属性设置为不可遍历，才需要通过defineProperty这个方法。
而且这里还有另一个类，Dep，这个类可以说是连接数据和Watcher的一个桥梁。后面我们会介绍，这里只需知道__ob__上有一个dep属性。
接下来，判断传入的数据是数组还是对象，如果是对象，则调用walk，如果是数组，则调用observeArray，这里先分析对象的。

#### walk

    const keys = Object.keys(obj)
    for (let i = 0; i < keys.length; i++) {
      defineReactive(obj, keys[i])
    }

walk很简单，遍历对象上的属性，调用defineReactive，这也是为什么__ob__属性要通过defineProperty来创建，否则这里就会遍历到__ob__属性。


#### defineReactive
    const dep = new Dep()
    ...
    if ((!getter || setter) && arguments.length === 2) {
      val = obj[key]
    }
    let childOb = !shallow && observe(val)
    Object.defineProperty(obj, key, {
      get: function reactiveGetter () {
        const value = getter ? getter.call(obj) : val
        if (Dep.target) {
          dep.depend()
          if (childOb) {
            childOb.dep.depend()
            if (Array.isArray(value)) {
              dependArray(value)
            }
          }
        }
        return value
      },
      set: function reactiveSetter (newVal) {
        ...
        if (setter) {
          setter.call(obj, newVal)
        } else {
          val = newVal
        }
        childOb = !shallow && observe(newVal)
        dep.notify()
      }
    })

首先通过key拿到val值，然后递归调用observe，也就是说，如果定义对象的某个
值还是一个对象，那么就会继续调用observe将子对象变成响应式对象。子对象也会有一个
__ob__属性。

    data: {
      __ob__: {}
      a: {
        __ob__: {}
      }
    }

递归调用observe后，为每个key设置getter和setter。

### 依赖收集

在调用render函数渲染组件时，会访问到定义在模板上的属性，这个时候就会触发响应式对象的getter

    get: function reactiveGetter () {
      const value = getter ? getter.call(obj) : val
      if (Dep.target) {
        dep.depend()
        if (childOb) {
          childOb.dep.depend()
          if (Array.isArray(value)) {
            dependArray(value)
          }
        }
      }
      return value
    }

在拿到getter的返回值之后，如果Dep.target存在则调用dep.depend()收集依赖

#### Dep.target

之前我们提到，在调用render函数之前，都会创建一个渲染watcher:

    if (typeof expOrFn === 'function') {
      this.getter = expOrFn
    }
    ...
    this.value = this.get()

我们将updateComponent传入到watcher中，在watcher中将updateComponent
赋值给getter，然后最后调用了get方法:

    pushTarget(this)
    ...
    value = this.getter.call(vm, vm)
    ...
    popTarget()

在执行getter也就是updateComponent之前，会执行pushTarget这个操作。
在'src/core/instance/observer/dep.js'中，定义了一个全局变量Dep.target
和一个targetStack栈。

    Dep.target = null
    const targetStack = []

    export function pushTarget (_target: ?Watcher) {
      if (Dep.target) targetStack.push(Dep.target)
      Dep.target = _target
    }

    export function popTarget () {
      Dep.target = targetStack.pop()
    }

而我们都知道，组件的渲染是一个递归的过程，也就是说当父组件渲染的时候
Dep.target就是父组件的渲染watcher，如果渲染的过程中有子组件，那么子组件也会创建它自己的
渲染watcher，并且把父组件的渲染watcher push到targetStack中，同时把Dep.target赋值
为子组件的渲染watcher。当子组件渲染完成时会调用popTarget，将Dep.target重新赋值为
父组件的渲染watcher，也就是说，Vue通过一个栈来维护父子组件的渲染watcher，Dep.target
也就代表着当前组件的渲染watcher。

ps: 在组件中watch属性创建的watcher也会push到targetStack中，之后分析watch的时候会说到。

#### dep.depend

回看上面的defineReactive函数，每次调用时都会创建一个dep的实例，之前我们也说过，这个是用来链接
数据和watcher的桥梁。这里调用了dep.depend:

    // dep
    depend () {
      if (Dep.target) {
        Dep.target.addDep(this)
      }
    }

根据我们上面的分析Dep.target是当前组件的渲染watcher，我们可以看到这里调用了wacher的addDep属性，
然后把当前的dep传入了。

    // watcher
    addDep (dep: Dep) {
      const id = dep.id
      if (!this.newDepIds.has(id)) {
        this.newDepIds.add(id)
        this.newDeps.push(dep)
        if (!this.depIds.has(id)) {
          dep.addSub(this)
        }
      }
    }

在addDep中，通过调用wacher的addSub方法，将当前的渲染watcher添加到dep中。这里可能有点绕，不过我们
只需要知道通过dep.depend，dep将渲染watcher保存在自身的subs属性上，而渲染watcher将dep保存在自身的
deps上。这就完成了依赖的收集。

### 派发更新

    set: function reactiveSetter (newVal) {
      const value = getter ? getter.call(obj) : val
      /* eslint-disable no-self-compare */
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return
      }
      /* eslint-enable no-self-compare */
      if (process.env.NODE_ENV !== 'production' && customSetter) {
        customSetter()
      }
      if (setter) {
        setter.call(obj, newVal)
      } else {
        val = newVal
      }
      childOb = !shallow && observe(newVal)
      dep.notify()
    }

当我们修改响应式数据时，会触发数据的setter，如果设置的新值也是一个对象或者数组的话，就会再次调用observe将数据变成响应式对象，
并且调用dep.notify触发组件重新渲染。


#### dep.notify

    notify () {
      // stabilize the subscriber list first
      const subs = this.subs.slice()
      for (let i = 0, l = subs.length; i < l; i++) {
        subs[i].update()
      }
    }

可以看到dep.notify就是遍历subs中保存的所有watcher，并调用它们的update方法。

#### watcher.update

    if (this.computed) {
      if (this.dep.subs.length === 0) {
        this.dirty = true
      } else {
        this.getAndInvoke(() => {
          this.dep.notify()
        })
      }
    } else if (this.sync) {
      this.run()
    } else {
      queueWatcher(this)
    }

因为我们这里调用的是渲染watcher的update，所以this.computed、this.sync都是false，这里直接执行queueWatcher并传入渲染watcher。

### queueWatcher

    function queueWatcher (watcher: Watcher) {
      const id = watcher.id
      if (has[id] == null) {
        has[id] = true
        if (!flushing) {
          queue.push(watcher)
        } else {
          // if already flushing, splice the watcher based on its id
          // if already past its id, it will be run next immediately.
          let i = queue.length - 1
          while (i > index && queue[i].id > watcher.id) {
            i--
          }
          queue.splice(i + 1, 0, watcher)
        }
        // queue the flush
        if (!waiting) {
          waiting = true
          nextTick(flushSchedulerQueue)
        }
      }
    }

我们都知道，在Vue中数据的变更到组件的重新渲染是一个异步的过程。这是因为如果每次数据变更都去重新渲染组件，这样是很消耗性能的，
所以Vue在这里通过一个队列来保存所有即将要更新的watcher，并在下一个tick中一次性更新。所以这里我们可以看到，flushing是一个标志位，
代表着是否正在遍历这个队列，如果不是的话，则直接将watcher push到队列中。当队列正在被遍历时，这时如果又有新的watcher进来，则从队列的最后
开始遍历，找到一个id比即将要插入的watcher的id更小的位置，然后将即将要插入的watcher插到它的后面，之所以要这样做，是因为队列里的所有watcher在
被遍历之前，都有一个根据id从小到大排列的操作。为什么要从小到大排列，在flushSchedulerQueue函数里面有说明。

#### flushSchedulerQueue

    function flushSchedulerQueue () {
      // Sort queue before flush.
      // This ensures that:
      // 1. Components are updated from parent to child. (because parent is always
      //    created before the child)
      // 2. A component's user watchers are run before its render watcher (because
      //    user watchers are created before the render watcher)
      // 3. If a component is destroyed during a parent component's watcher run,
      //    its watchers can be skipped.
      queue.sort((a, b) => a.id - b.id)

      // do not cache length because more watchers might be pushed
      // as we run existing watchers
      for (index = 0; index < queue.length; index++) {
        watcher = queue[index]
        if (watcher.before) {
          watcher.before()
        }
        ...
        watcher.run()
        ...
      }

在遍历队列之前，先对队列里的watcher进行根据id从小到大的排列，通过Vue这里注释的解释可以看到Vue为什么要这样做:

1、 组件的更新都是由父到子的(父组件总是在子组件之前被创建)

2、 组件内定义的userWatcher要比组件的渲染watcher先执行(user watcher总是在render watcher之前被创建)

3、 如果一个组件在它的父组件执行watcher的期间被销毁，那么可以跳过这个组件的watcher。

在排序之后，就遍历队列，可以注意到每次循环都去拿到queue.length，这是因为在遍历的过程中有可能有新的watcher进来，队列的长度会改变。
如果watcher有before函数，则先执行before，然后执行watcher.run()

#### watcher.run

    run () {
      if (this.active) {
        this.getAndInvoke(this.cb)
      }
    }

渲染watcher的cb属性是空。

#### getAndInvoke

    const value = this.get()

    // 因为渲染watcher调用get的返回值都是undefined，所以getAndInvoke方法只执行到了这里，接下来的逻辑是computed和user watcher的。

在getAndInvoke中可以看到，再次调用了get方法:

    pushTarget(this)
    ...
    value = this.getter.call(vm, vm)
    ...
    popTarget()

在get方法里面会再次调用watcher的getter方法，之前我们也分析过getter就是传入的updateComponent，所以再次执行updateComponent又会
再次调用render函数，重新渲染。

### nextTick

之前分析到flushSchedulerQueue时，我们可以看到flushSchedulerQueue是传入nextTick函数里执行的。而在日常的开发中，我们也经常遇到，比如说，修改了
数据，可以看到界面上的DOM是更新了，但是我们获取的DOM却不是最新的，根据官网的教程:

    var vm = new Vue({
      el: '#example',
      data: {
        message: '123'
      }
    })
    vm.message = 'new message' // 更改数据
    vm.$el.textContent === 'new message' // false
    Vue.nextTick(function () {
      vm.$el.textContent === 'new message' // true
    })

可以看到，Vue建议我们将DOM操作放到$nextTick中来保证获取的DOM是最新的。那么为什么在$nextTick中获取的DOM就是更新之后的呢，在分析nextTick之前，希望你
可以先了解一下javascript的事件循环机制。

    function nextTick (cb?: Function, ctx?: Object) {
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
        if (useMacroTask) {
          macroTimerFunc()
        } else {
          microTimerFunc()
        }
      }
      // $flow-disable-line
      if (!cb && typeof Promise !== 'undefined') {
        return new Promise(resolve => {
          _resolve = resolve
        })
      }
    }

可以看到在同一个tick中调用nextTick方法时，都是将传入的函数push到callbacks中，然后执行macroTimerFunc或者microTimerFunc。

macroTimerFunc走的是宏任务，在Vue中做了以下的兼容，setImmediate -> MessageChannel -> setTimeout 优先级从左到右。

microTimerFunc走的是微任务，调用的是Promise.then。

不管执行macroTimerFunc或者microTimerFunc，最后都是遍历callbacks中的所有方法，并执行。这也解释了为什么需要将DOM操作放在nextTick中，
因为当我们修改数据时，首先会触发重新渲染，并将flushSchedulerQueue放入到nextTick中等待执行，此时如果我们直接执行DOM操作，那么DOM操作在组件
重新渲染之前已经执行了，而如果把DOM操作放到nextTick中，它将和flushSchedulerQueue一起被放进callbacks中顺序执行，这样就确保了我们的DOM操作
是在组件更新之后执行的。
