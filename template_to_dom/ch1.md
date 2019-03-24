## new Vue到Dom呈现到页面之间到底发生了什么

在我们找到Vue的定义之后，可以看到当我们new Vue时，Vue内部调用了_init方法。

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

_init方法中可以省略直接看到最后，这里判断了是否在options上定义了el属性，如果定义了则调用vm.$mount()

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

_render其实很简单，就是拿到我们定义在组件的render方法，并且执行它，通过执行render方法得到vnode。
Vue在2.0的时候引入了vnode的概念，简单来说，vnode就是一个对象，用来描述DOM节点的信息。关于vnode的介绍，请移步[vnode以及Vue中一些关于vnode的概念，必看!!](./vnode.md)。

返回创建好的vnode之后，_update函数就会被调用。

### _update

    首先，我们要知道，不管是组件的创建或者重新渲染，都会执行到_update这个方法。

    const vm: Component = this
    const prevEl = vm.$el
    const prevVnode = vm._vnode
    const prevActiveInstance = activeInstance
    activeInstance = vm
    vm._vnode = vnode
    ...
    if (!prevVnode) {
      // initial render
      vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
    } else {
      // updates
      vm.$el = vm.__patch__(prevVnode, vnode)
    }
    activeInstance = prevActiveInstance

在调用_update时，拿到实例vm上的_vnode，这个是一个组件的根vnode，例如有这样一个组件App:

    <template>
      <div>
        <span></span>
      </div>
    </template>
    // _vnode就代表着div节点生成的vnode，因为vnode是树结构，所以他包含着它的子vnode。

拿到_vnode后，赋值给prevVnode，并且将新的vnode赋值给_vnode，接下来判断有没有prevVnode，如果没有那么说明这是
组件的创建，调用__patch__函数的时候传入了vm.$el和vnode。如果已经有prevVnode了那么说明这是组件的重新渲染的过程，
所以传入了新旧vnode去进行比对。这里还有一个变量是activeInstance，这是一个全局的变量，它代表着正在激活的组件实例，
也可以说是正在patch的组件实例。因为组件的patch是一个递归的过程，所以Vue通过prevActiveInstance来保存父组件的实例
当子组件patch完毕时，又会把activeInstance赋值为父组件的实例。

### __patch__

这里我们分析的是组件的创建过程，所以patch传入的参数是:

    vm.$el  //在mountComponent函数调用时，把我们在options传入的el参数转化为dom节点，也就是我们常写的#app
    vnode   //render生成的组件vnode


### patch(oldVnode, vnode）

    if (isUndef(oldVnode)) {
      ...
    } else {
      const isRealElement = isDef(oldVnode.nodeType)
      if (!isRealElement && sameVnode(oldVnode, vnode)) {
        ...
      } else {
        // 主要逻辑
        if (isRealElement) {
          ...
          oldVnode = emptyNodeAt(oldVnode)
        }
        const oldElm = oldVnode.elm
        const parentElm = nodeOps.parentNode(oldElm)

        createElm(
          vnode,
          insertedVnodeQueue,
          // extremely rare edge case: do not insert if old element is in a
          // leaving transition. Only happens when combining transition +
          // keep-alive + HOCs. (#4590)
          oldElm._leaveCb ? null : parentElm,
          nodeOps.nextSibling(oldElm)
        )
        ...
        // 主要逻辑
      }
    }

在patch中，我们可以先忽略其他关于组件重新渲染的代码，这里我们只看创建的过程。
首先，在调用patch时，判断了有没有oldvnode，很显然是有的，所以走的是else，然后继续判断oldVnode是不是一个真实的节点，
很显然回看我们上一个函数，传入的是一个真实的节点，那么这里又继续走else。接下来，通过调用emptyNodeAt去创建一个vnode，
并且把我们传入的真实的dom节点赋值到它的elm属性上，然后通过elm去拿到parentElm，很显然这个时候parentElm是我们的body。
然后调用createElm这个方法，我们分析一下这时候传入的参数:

    createElm(
      vnode,          // 根节点vnode
      insertedVnodeQueue, // 后面我们会分析这个参数，先不管
      oldElm._leaveCb ? null : parentElm,  // 因为oldElm没有_leaveCb这个属性，所以传入的是parentElm，也就是body
      nodeOps.nextSibling(oldElm) // null
    )

### createElm

    const data = vnode.data
    const children = vnode.children
    const tag = vnode.tag
    ...
    vnode.elm = nodeOps.createElement(tag, vnode)
    ...
    createChildren(vnode, children, insertedVnodeQueue)
    if (isDef(data)) {
      invokeCreateHooks(vnode, insertedVnodeQueue)
    }
    insert(parentElm, vnode.elm, refElm)

我们知道，组件的patch是一个递归的过程，所以这里先通过vnode的tag生成一个真实的节点，并保存在elm属性上，
然后调用createChildren，继续patch子vnode。

### createChildren

    for (let i = 0; i < children.length; ++i) {
      createElm(children[i], insertedVnodeQueue, vnode.elm, null, true, children, i)
    }

在createChildren中，遍历了children，递归调用createElm函数，然后我们分析一下createElm这时传入的参数

    children[i]         // 下一个要patch的vnode
    insertedVnodeQueue
    vnode.elm           // 父vnode的真实节点
    null

然后继续调用createElm，如果子vnode还有children的话，又会继续遍历子vnode的children。
当执行完createChildren时，代表已经没有叶子节点，这时会调用一个insert方法，就是通过这个方法，来插入我们创建好的dom节点
我们看一下insert方法:

    function insert (parent, elm, ref) {
      if (isDef(parent)) {
        if (isDef(ref)) {
          if (ref.parentNode === parent) {
            nodeOps.insertBefore(parent, elm, ref)
          }
        } else {
          nodeOps.appendChild(parent, elm)
        }
      }
    }

insert方法很简单，就是把我们生成的节点插入到父节点中。

那么简单总结一下patch的过程，其实就是递归的调用createElm，通过vnode上面保存的dom节点的信息来生成真实的dom节点，并保存在
vnode的elm属性上，然后通过每次传入下一个要创建的vnode，以及当前vnode的elm属性作为下一个vnode生成的真实节点的父节点来调用createElm，在遍历完chlidren时，在递归回溯的过程中调用insert方法来插入子节点。从而完成组件的patch过程，通过分析，我们也得知，组件的patch过程中，节点的插入顺序是先子后父的。
