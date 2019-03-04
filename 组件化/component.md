## 组件化

上一篇我们介绍了new Vue到DOM的挂载之间发生了什么，那么当父组件中嵌套了子组件，Vue是如何对子组件进行初始化并将子组件挂载到父组件上的呢？

### 生成组件vnode阶段

#### render

上一篇我们分析到，在一个组件初始化的过程中，会创建渲染watcher，并调用render函数来生成vnode，在执行render函数的过程中，
会调用实例上_c这个方法，也就是createElement，createElement对传入的参数进行处理，并调用_createElement。

#### _createElement

    if (typeof tag === 'string') {
      let Ctor
      ns = (context.$vnode && context.$vnode.ns) || config.getTagNamespace(tag)
      if (config.isReservedTag(tag)) {
        // platform built-in elements
        vnode = new VNode(
          config.parsePlatformTagName(tag), data, children,
          undefined, undefined, context
        )
      } else if (isDef(Ctor = resolveAsset(context.$options, 'components', tag))) {
        // component
        vnode = createComponent(Ctor, data, context, children, tag)
      } else {
        // unknown or unlisted namespaced elements
        // check at runtime because it may get assigned a namespace when its
        // parent normalizes children
        vnode = new VNode(
          tag, data, children,
          undefined, undefined, context
        )
      }
    } else {
      // direct component options / constructor
      vnode = createComponent(tag, data, context, children)
    }

如果_createElement中传入的tag是字符串并且tag是一个HTML的保留标签，则创建的是一个普通的vnode，如果不是保留标签，并且可以通过
resolveAsset方法在组件上拿到组件的定义，则创建一个组件vnode[关于组件vnode以及其他vnode的概念]()，组件vnode是通过createComponent
方法来创建的。

#### createComponent

createComponent传入了几个参数

    Ctor: Class<Component> | Function | Object | void,  // 组件定义，vue-loader会把通过import引入的组件定义变成一个对象
    data: ?VNodeData,           // 标签上定义的属性
    context: Component,         
    children: ?Array<VNode>,    // 写在组件中的内容，用来实现slot
    tag?: string

    if (isObject(Ctor)) {
      Ctor = baseCtor.extend(Ctor)
    }
    ...
    installComponentHooks(data)

    const name = Ctor.options.name || tag
    const vnode = new VNode(
      `vue-component-${Ctor.cid}${name ? `-${name}` : ''}`,
      data, undefined, undefined, undefined, context,
      { Ctor, propsData, listeners, tag, children },
      asyncFactory
    )
    return vnode

涉及组件化相关的代码有以上这些，首先判断我们拿到的组件定义Ctor是不是Object，如果是则调用baseCtor.extend去创建子组件的构造器，
baseCtor是Vue，也就是调用了Vue.extend

#### Vue.extend

    var Sub = function VueComponent (options) {
      this._init(options);
    };
    Sub.prototype = Object.create(Super.prototype);
    Sub.prototype.constructor = Sub;
    Sub.cid = cid++;
    Sub.options = mergeOptions(
      Super.options,
      extendOptions
    );
    Sub['super'] = Super;
    return Sub

通过原型继承的方式，创建了子构造器，子构造函数继承于Vue。

#### installComponentHooks

    在组件vnode的data上面添加了4个hook，init、prepatch、insert、destroy

### patch阶段

patch阶段是vnode -> DOM的过程，这时候会调用createElm去创建真实的DOM，在createElm中，首先会对vnode执行一个方法createComponent

#### createComponent

    let i = vnode.data
    if (isDef(i)) {
      const isReactivated = isDef(vnode.componentInstance) && i.keepAlive
      if (isDef(i = i.hook) && isDef(i = i.init)) {
        i(vnode, false /* hydrating */)
      }
      if (isDef(vnode.componentInstance)) {
        initComponent(vnode, insertedVnodeQueue)
        insert(parentElm, vnode.elm, refElm)
        if (isTrue(isReactivated)) {
          reactivateComponent(vnode, insertedVnodeQueue, parentElm, refElm)
        }
        return true
      }
    }

判断组件vnode上的data有没有init hook，如果有就执行。这就涉及之前installComponentHooks方法为组件vnode添加的init hook

#### init hook

    const child = vnode.componentInstance = createComponentInstanceForVnode(
      vnode,
      activeInstance
    )
    child.$mount(hydrating ? vnode.elm : undefined, hydrating)

通过createComponentInstanceForVnode创建子组件实例，并赋值到vnode.componentInstance，并手动调用$mount方法。

#### createComponentInstanceForVnode

    const options: InternalComponentOptions = {
      _isComponent: true,
      _parentVnode: vnode,     // 占位符vnode
      parent                   // 父组件实例
    }

    return new vnode.componentOptions.Ctor(options)

这里要记住创建子组件实例传入的几个参数，以后会涉及到。

在返回通过new创建的子组件实例后，又再次调用子组件的$mount方法，再走一遍patch流程，如果在patch的过程中又有子组件，则重复上面的步骤。
在子组件创建完成后，也就是执行完init hook之后，会执行createComponent剩余的代码:

    if (isDef(vnode.componentInstance)) {
      initComponent(vnode, insertedVnodeQueue)
      insert(parentElm, vnode.elm, refElm)
      if (isTrue(isReactivated)) {
        reactivateComponent(vnode, insertedVnodeQueue, parentElm, refElm)
      }
      return true
    }

#### initComponent

    vnode.elm = vnode.componentInstance.$el

此时子组件已经patch完成，并且将组件的真实DOM挂载在实例的$el属性上，这里拿到$el属性，并赋值给占位符vnode的elm属性。
执行完initComponent后，调用insert方法将子组件的真实DOM插入到父组件中。