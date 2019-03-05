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


