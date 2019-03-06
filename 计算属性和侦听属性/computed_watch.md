## computed vs watch

### 计算属性依赖收集

计算属性也是在initState方法中初始化的:

    if (opts.computed) initComputed(vm, opts.computed)

#### initComputed

    function initComputed (vm: Component, computed: Object) {
      const watchers = vm._computedWatchers = Object.create(null)
      const isSSR = isServerRendering()

      for (const key in computed) {
        const userDef = computed[key]
        const getter = typeof userDef === 'function' ? userDef : userDef.get
        ...
        if (!isSSR) {
          // create internal watcher for the computed property.
          watchers[key] = new Watcher(
            vm,
            getter || noop,
            noop,
            computedWatcherOptions      // { computed: true }
          )
        }
        ...
        defineComputed(vm, key, userDef)
      }
    }

initComputed方法就是遍历computed上的所有key，拿到computed的定义，computed可以的值可以是函数，也可以是一个对象，但是必须
设置手动设置get方法。拿到computed的定义getter后，通过new Watcher 传入了computed:true这样的配置，来创建computed watcher。

    if (this.computed) {
      this.value = undefined
      this.dep = new Dep()
    }

在Watcher的定义中可以看到，当我们传入的computed配置为true时，一开始并不会调用get方法去计算value值，而是将value设置为undefined，
并创建一个dep，这也说明了，计算属性只有当我们访问的时候，才会去求值。这里我们也可以知道，computed也是通过Watcher来实现的。
创建完watcher之后，调用defineComputed将computed挂载到组件实例上，并通过defineProperty来设置getter和setter。

#### defineComputed

    function defineComputed (
      target: any,
      key: string,
      userDef: Object | Function
    ) {
      const shouldCache = !isServerRendering()
      if (typeof userDef === 'function') {
        sharedPropertyDefinition.get = shouldCache
          ? createComputedGetter(key)
          : userDef
        sharedPropertyDefinition.set = noop
      } else {
        sharedPropertyDefinition.get = userDef.get
          ? shouldCache && userDef.cache !== false
            ? createComputedGetter(key)
            : userDef.get
          : noop
        sharedPropertyDefinition.set = userDef.set
          ? userDef.set
          : noop
      }
      ...
      Object.defineProperty(target, key, sharedPropertyDefinition)
    }

从defineComputed可以看出，当变量shouldCache是true时，get方法赋值的是createComputedGetter返回的方法，而在非服务端渲染的时候，
shouldCache一定是true的，除非设置cache属性为false。缓存主要是用来减少重复计算的，这也是计算属性的特性之一。

#### createComputedGetter

    function createComputedGetter (key) {
      return function computedGetter () {
        const watcher = this._computedWatchers && this._computedWatchers[key]
        if (watcher) {
          watcher.depend()
          return watcher.evaluate()
        }
      }
    }

createComputedGetter返回了一个函数，当我们访问计算属性的值时，就会执行createComputedGetter返回的方法。这个方法通过key值拿到保存在实例属性_computedWatchers上的computed watcher并调用watcher.depend()。watcher.depend是专门为computed实现的逻辑:

    depend () {
      if (this.dep && Dep.target) {
        this.dep.depend()
      }
    }

调用自身的dep属性的depend方法，然后当前的Dep.target就会调用addDep把computed watcher上的dep添加到它的deps中，完成对计算属性的依赖收集。完成依赖收集后，返回
watcher.evaluate()的返回值。

#### watcher.evaluate

    evaluate () {
      if (this.dirty) {
        this.value = this.get()
        this.dirty = false
      }
      return this.value
    }

evaluate方法调用了get方法得到计算属性的值，并返回。

### 计算属性更新

当计算属性依赖的值改变时，计算属性会重新计算。比如现在有一个计算属性a，它依赖了b:

    computed: {
      a () {
        return thia.b + 1
      }
    }

那么当我们访问a属性时，会执行定义在cumputed上的a这个方法，并返回this.b + 1的值，这个时候就会访问到依赖b的值并触发b的getter，从而b的dep就会订阅computed
watcher，把computed watcher保存到属性b的dep中。当b更改时，触发b的setter，执行dep.notify，此时就会触发computed watcher的update方法，重新计算计算属性
的值。

#### update

    update () {
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
    }

这跟渲染watcher重新渲染调用的是同个方法，不过computed watcher走的是computed为true的分支，这里Vue根据computed watcher的dep属性的subs中有没有值，来走不同的逻辑
当computed的值改变会触发其他watcher的update时，会调用this.getAndInvoke，并执行回调this.dep.notify来触发更新，如果subs中没有值，则computed并不会重新计算，只是将dirty置为true。

#### getAndInvoke

    getAndInvoke (cb: Function) {
      const value = this.get()
      if (
        value !== this.value ||
        isObject(value) ||
        this.deep
      ) {
        // set new value
        const oldValue = this.value
        this.value = value
        this.dirty = false
        ...
        cb.call(this.vm, value, oldValue)
      }
    }

当computed的值改变会触发其他watcher的update时，会通过get方法重新计算computed的值，并与旧值进行对比，只有当新旧值不同时，computed才会调用cb，也就是this.dep.notify去触发依赖computed的其他值的更新。

也就是说，computed本身是一个watcher，可以被它的依赖值订阅，当它的依赖值改变时，会触发它的更新。computed也可以是一个被依赖的值(具有dep属性)，当它在模板中
使用，或者被watch时，computed的改变就会触发它的dep属性里的watcher的更新。

所以可以总结一下computed的几个特性:

1、只有当computed被访问时，才去计算它的值。

2、当新旧值不同时，才触发重新渲染。

### 侦听属性

Vue提供了侦听属性，让我们可以在数据变更的时候执行一些操作，侦听属性的值可以是一个函数，也可以是一个对象:

    watch: {                         watch: {
      a (val, oldVal) {}               a: {
    }                                    handler (val, oldVal) {},
                                         deep,
                                         immediate,
                                         sync
                                       }
    
#### initWatch

    for (const key in watch) {
      const handler = watch[key]
      if (Array.isArray(handler)) {
        for (let i = 0; i < handler.length; i++) {
          createWatcher(vm, key, handler[i])
        }
      } else {
        createWatcher(vm, key, handler)
      }
    }

    if (isPlainObject(handler)) {
      options = handler
      handler = handler.handler
    }
    if (typeof handler === 'string') {
      handler = vm[handler]
    }

    return vm.$watch(expOrFn, handler, options)

首先遍历watch上的所有key，拿到watch的定义传入createWatcher，通过createWatcher来处理一下参数，定义是一个对象，就拿对象的handler属性，
并把对象当做options传入，通过调用$watch来创建user watcher

#### $watch

    Vue.prototype.$watch = function (
      expOrFn: string | Function,
      cb: any,
      options?: Object
    ): Function {
      const vm: Component = this
      if (isPlainObject(cb)) {
        return createWatcher(vm, expOrFn, cb, options)
      }
      options = options || {}
      options.user = true
      const watcher = new Watcher(vm, expOrFn, cb, options)
      if (options.immediate) {
        cb.call(vm, watcher.value)
      }
      return function unwatchFn () {
        watcher.teardown()
      }
    }

可以看到$watch也是通过new Watcher来创建user watcher，这里把传入的option的user置为true，然后如果设置了immediate为true，则先直接调用一次
我们传入的callback，返回watcher的销毁方法。

### 侦听属性依赖收集

根据上面的分析，可以知道侦听属性也是通过Watcher来实现的，那么在new Watcher的时候，有这样的逻辑:

    if (typeof expOrFn === 'function') {
      this.getter = expOrFn
    } else {
      this.getter = parsePath(expOrFn)
    }
    ...
    this.value = this.get()

很显然，我们创建user watcher传入的exoOrFn是watch对象的key值，也就是一个字符串，所以Vue会调用parsePath，把expOrFn转化为一个函数，并赋值给getter方法

#### parsePath

    const bailRE = /[^\w.$]/
    function parsePath (path: string): any {
      if (bailRE.test(path)) {
        return
      }
      const segments = path.split('.')
      return function (obj) {
        for (let i = 0; i < segments.length; i++) {
          if (!obj) return
          obj = obj[segments[i]]
        }
        return obj
      }
    }

parsePath通过'.'来分割字符串，并且返回了一个函数，这个函数接收一个对象作为参数，通过我们分割字符串得到的路径来获取对象上的值，并返回。
也就是说当我们调用user watcher的getter方法时，传入了当前的实例对象，getter方法就就返回根据expOrFn得到的路径对应的实例对象上的值:

    data () {
      return {       ->   vm.xx 
        xx: 'xx'
      }
    }

    watch: {
      xx () {}    ->   parsePath('xx') ->  this.getter.call(vm, vm)  ->  return vm['xx']
    }

在访问实例对象上的值时，就会触发响应式属性的依赖收集，当响应式属性的值更改时，就会触发user watcher的update方法。

### 侦听属性回调的执行

    if (this.computed) {
      this.dirty = true
    } else if (this.sync) {
      this.run()
    } else {
      queueWatcher(this)
    }

当触发user watcher的update方法时，如果在watch中设置了sync为true，则直接执行run方法，否则，就跟渲染watcher一样添加到缓冲队列，
并在下一个tick中执行run方法，在run方法中执行getAndInvoke并传入我们定义的回调函数。

    const value = this.get()
    if (
      value !== this.value ||
      isObject(value) ||
      this.deep
    ) {
      const oldValue = this.value
      this.value = value
      this.dirty = false
      if (this.user) {
        try {
          cb.call(this.vm, value, oldValue)
        } catch (e) {
          handleError(e, this.vm, `callback for watcher "${this.expression}"`)
        }
      } else {
        cb.call(this.vm, value, oldValue)
      }
    }

在getAndInvoke重新执行get方法拿到新的value值，并进行比对，如果新旧值不相等或者新值是对象或者设置了deep为true，都会执行回调，并传入新旧值。

#### traverse

当我们watch一个对象，并设置deep为true时，不管修改了对象的哪个key，都会触发watch回调的执行。这是因为在调用get方法时，如果deep为true会执行traverse方法

    // "touch" every property so they are all tracked as
    // dependencies for deep watching
    if (this.deep) {
      traverse(value)
    }

Vue这里也有解释，通过traverse方法来"touch"响应式对象的每一个key，让每个key的dep都订阅当前的watcher

    function _traverse (val: any, seen: SimpleSet) {
      let i, keys
      const isA = Array.isArray(val)
      if ((!isA && !isObject(val)) || Object.isFrozen(val) || val instanceof VNode) {
        return
      }
      if (val.__ob__) {
        const depId = val.__ob__.dep.id
        if (seen.has(depId)) {
          return
        }
        seen.add(depId)
      }
      if (isA) {
        i = val.length
        while (i--) _traverse(val[i], seen)
      } else {
        keys = Object.keys(val)
        i = keys.length
        while (i--) _traverse(val[keys[i]], seen)
      }
    }

traverse方法就是简单的对我们监听的对象进行遍历，当对象有__ob__属性时，并且通过一个set来保存depId，防止对象上有相互引用，导致无限循环。