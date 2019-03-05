## 生命周期

在日常使用Vue开发的过程中，我们经常会使用组件的各个钩子，而且也知道哪些钩子可以做什么，比如在created钩子就可以访问到
组件定义的data，watch，computed，props，也可以通过created请求api，如果要对DOM做一些操作，则要放到mounted中。那么
在整个组件创建的过程中，这些钩子是在什么时机被调用，以及如果嵌套了多个组件，这些钩子在父子组件中调用的顺序又是怎样的呢？

![生命周期](../images/lifecycle.png)
先放一张官网的生命周期图示，然后我们从源码的角度一个一个来分析。

### beforeCreate

前面我们分析到，在创建组件实例也就是new Vue的时候，会调用_init方法，在_init方法中，我们可以看到在beforeCreate之前调用了以下几个函数

    initLifecycle(vm)     // 完成父子组件实例的关联，往实例上添加一些标志位
    initEvents(vm)        // 处理组件上的一些自定义事件，后面分析event会涉及
    initRender(vm)        // 挂载一些和组件渲染、slots相关的属性
    callHook(vm, 'beforeCreate')

beforeCreate在开发中比较少用，不过我们熟悉的vuex和vue-router，都是通过在这个钩子中混入一些逻辑来实现的。

### created

    initInjections(vm) // resolve injections before data/props
    initState(vm)
    initProvide(vm) // resolve provide after data/props
    callHook(vm, 'created')

initInjections和initProvide是跟provide/inject相关的，如果你使用过react的context，和这个是一样的东西。然后在created之前还调用了
initState，之前我们在响应式原理的时候分析过，initState是通过调用defineReactive把组件内定义的数据变成响应式对象，所以这也是为什么在created钩子中就可以访问到组件内
定义的data、porps、computed、watch、methods。

从我们之前分析组件初始化的流程可以得到，当嵌套多个组件时，总是父组件实例先初始化再初始化子组件实例，也就是说beforeCreate和created的调用都是先父后子的。

parent beforeCreate -> parent created -> child beforeCreate -> child created

### beforeMount

在组件实例初始化之后，Vue就会调用实例上的$mount方法来执行组件的渲染，从源码可以看到beforeMount钩子是在执行mountComponent的时候调用的，而且是在创建
渲染watcher之前。

    callHook(vm, 'beforeMount')
    ...
    new Watcher(vm, updateComponent, noop, {
      before () {
        if (vm._isMounted) {
          callHook(vm, 'beforeUpdate')
        }
      }
    }, true /* isRenderWatcher */)

当嵌套多个组件时，也是先调用父组件的$mount再调用子组件的，所以beforeMount的调用也是先父后子的。

parent beforeMount -> child beforeMount

### mounted

    new Watcher(vm, updateComponent, noop, {
      before () {
        if (vm._isMounted) {
          callHook(vm, 'beforeUpdate')
        }
      }
    }, true /* isRenderWatcher */)
    ...
    if (vm.$vnode == null) {
      vm._isMounted = true
      callHook(vm, 'mounted')
    }

组件的渲染是一个递归的过程，当父组件创建渲染watcher执行updateComponent函数渲染组件时，在父组件的patch阶段如果遇到组件vnode，
则创建子组件的实例和子组件的渲染watcher，执行子组件的渲染过程，而mounted钩子是在组件创建渲染watcher执行完渲染并挂载真实DOM节点
之后才执行的，所以mounted的执行是先子后父的。

parent mounted -> child mounted

### beforeUpdate

在响应式原理的章节我们分析到，当我们改变组件中的响应式数据时，会触发getter，并执行dep.notify，dep.notify会将渲染watcher放到缓冲
队列中，并在下一个tick执行，而遍历缓冲队列执行watcher.run之前，会先执行watcher.before，而在创建渲染watcher的时候我们可以看到:

    new Watcher(vm, updateComponent, noop, {
      before () {
        if (vm._isMounted) {
          callHook(vm, 'beforeUpdate')
        }
      }
    }, true /* isRenderWatcher */)

在第四个参数传入了一个对象，里面有一个before函数调用了beforeUpdate钩子，beforeUpdate就是在组件重新渲染之前执行的，并且我们之前也分析过，
缓冲队列里面的watcher总是根据id从小到大排列，也就是说父组件的渲染watcher是先于子组件的渲染watcher被遍历的，所以beforeUpdate的执行是先父后子的。

parent beforeUpdate -> child beforeUpdate

### updated

在执行flushSchedulerQueue遍历缓冲队列后，会调用callUpdatedHooks方法:

    function callUpdatedHooks (queue) {
      let i = queue.length
      while (i--) {
        const watcher = queue[i]
        const vm = watcher.vm
        if (vm._watcher === watcher && vm._isMounted) {
          callHook(vm, 'updated')
        }
      }
    }

可以看到，这个方法就是从后往前遍历缓冲队列，并执行updated钩子，也就是先执行子组件的updated然后再执行父组件的updated。

parent updated -> child updated

### beforeDestroy destroyed

这两个钩子是调用组件实例上的$destroy方法后触发的，当$destroy方法被调用时，会立即触发beforeDestroy，然后组件实例会执行一些销毁的操作，
比如:
    callHook(vm, 'beforeDestroy')
    ...
    remove(parent.$children, vm);
    vm._watcher.teardown();
    var i = vm._watchers.length;
    while (i--) {
      vm._watchers[i].teardown();
    }
    ...
    callHook(vm, 'destroyed')

断开销毁组件与它的父组件之间的联系，将实例上的watcher teardown等等，当销毁完毕时，调用destroyed钩子。