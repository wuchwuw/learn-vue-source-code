## new Vue到Dom呈现到页面之间到底发生了什么

在我们找到Vue的定义之后，可以看到当我们new Vue时，Vue内部调用了_init方法。

    function Vue (options) {
      if (process.env.NODE_ENV !== 'production' &&
        !(this instanceof Vue)
      ) {
        warn('Vue is a constructor and should be called with the `new` keyword')
      }
      this._init(options)
    }

除此之外Vue还通过以下几个方法来扩展原型上的方法

    initMixin(Vue)        // 在原型上扩展_init方法，
    stateMixin(Vue)       // 在原型上扩展$set、$delete、$watch
    eventsMixin(Vue)      // 在原型上扩展$on、$once、$off、$emit
    lifecycleMixin(Vue)   // 在原型上扩展_update、$forceUpdate、$destroy、$emit
    renderMixin(Vue)      // 在原型上扩展$nextTick、_render

这里简单介绍一下在原型上扩展的方法，但是我们这章的主要目的是了解new Vue到Dom之间发生的事，所以这些方法不会细讲，以后涉及到相关的方法再细说。

### _init()

_init方法中可以省略直接看到最后，这里判断了是否在options上定义了el属性，如果定义了则调用vm.$mount(vm.$options.el)

    if (vm.$options.el) {
      vm.$mount(vm.$options.el)
    }

vm.$mount()定义在'src/platforms/web/runtime/index.js'中，这里主要是对传入的el参数进行处理，并调用mountComponent方法，并传入处理好的el参数

### mountComponent

    vm.$el = el
    ...
    ...
    updateComponent = () => {
      vm._update(vm._render(), hydrating)
    }
    ...
    ...
    new Watcher(vm, updateComponent, noop, {
      before () {
        if (vm._isMounted) {
          callHook(vm, 'beforeUpdate')
        }
      }
    }, true /* isRenderWatcher */)

在mountComponent方法中，主要定义了updateComponent方法，并且把updateComponent函数当做new Watcher的参数传入。
这里先简单讲一下watcher的概念，可以理解为watcher是为了监听某些操作的发生而创建，当这些操作发生时，就会通知watcher，执行相应的操作。
在Vue里面有这样的几种watcher，它们都是通过new Watcher的时候传入不同的参数来创建的:

    // Watcher定义在'src/core/observer/watcher.js'下

    renderWatcher   // 监听组件更新，执行组件重新渲染的操作
    userWatcher     // 在组件里面通过watch属性、或者$watch方法创建的，当监听的数据改变时，执行我们定义的回调函数
    computedWatcher // 在组件里面通过computed属性创建，当计算属性的依赖值改变时，对计算属性重新求值

而这里我们创建的就是renderWatcher，在首次创建、以及组件重新渲染时，就会调用我们传入的updateComponent函数。


### updateComponent

    updateComponent = () => {
      vm._update(vm._render(), hydrating)
    }

updateComponent执行的是_update()方法，它会传入_render()方法的返回值作为参数，这里只需知道，当组件重新渲染时，就会调用updateComponent

### _render

    const { render, _parentVnode } = vm.$options
    ...
    vm.$vnode = _parentVnode
    ...
    vnode = render.call(vm._renderProxy, vm.$createElement)
    ...
    vnode.parent = _parentVnode
    return vnode

_render其实很简单，就是拿到我们定义在组件的render方法，并且执行它，通过执行render方法得到vnode。[webpack]
这里我们可以打印一下render函数出来看看，
Vue在2.0的时候引入了vnode的概念，简单来说，vnode就是一个对象，用来描述dom节点的信息。这里还有几个点要记住的。

关于vnode的介绍，请移步[vnode]()
