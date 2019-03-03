## 响应式原理

熟悉Vue的都知道，Vue的响应式是通过Object.defineProperty来实现的。但是仅仅知道这个是不够的，我们需要了解，响应式数据是如何创建的，以及响应式数据的改变是如何触发组件重新渲染的。

这里先放一张Vue官网对响应式原理说明。
![响应式原理](../images/data.png)
从图中我们可以大致的了解整个响应式原理的过程，当组件创建的过程中，会把我们定义在组件上的data变成响应式对象，也就是通过Object.defineProperty为对象添加getter和setter(图中紫色圈圈部分)，当组件渲染时，就会访问到我们定义在template中的响应式数据(图中的"touch")，这时就会触发数据的getter从而收集依赖(图中的collect as Dependency),当我们对数据进行更改时，会触发setter，从而执行dep.notify通知watcher执行组件的重新渲染。
接下来我们就从响应式对象的创建开始，来分析Vue的响应式原理

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

这个方法很简单，就是分别拿到组件options的props、methods、data、computed、watch，分别执行它们对应的init方法。
这里我们先看initData

### initData

首先，拿到data并且校验了props和methods上的key是不是和data上有重复。校验之后调用了observe方法，就是通过这个方法来创建响应式对象的。

### observe

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

### walk

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

### dep.depend

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

##