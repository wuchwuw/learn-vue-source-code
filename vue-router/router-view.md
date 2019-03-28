## router-view

当我们切换路由触发视图重新渲染时，router-view组件渲染的就是当前路由将要渲染的组件，router-view组件定义在
'src/components/view.js'下，它是一个函数式的组件，如果不了解函数式组件可以移步到[Vue函数式组件](https://cn.vuejs.org/v2/guide/render-function.html#%E5%87%BD%E6%95%B0%E5%BC%8F%E7%BB%84%E4%BB%B6)先了解
一下。

### render

简化render函数的代码后可以看到：

    data.routerView = true

    const h = parent.$createElement

    let depth = 0

    while (parent && parent._routerRoot !== parent) {
      if (parent.$vnode && parent.$vnode.data.routerView) {
        depth++
      }
      if (parent._inactive) {
        inactive = true
      }
      parent = parent.$parent
    }

    const matched = route.matched[depth]

    const component = matched.components[name]

    return h(component, data, children)

当router-view调用render函数来返回vnode时，会先在自己的data上挂载一个routerView的属性，并且拿到parent的$createElement函数来渲染component返回Vnode。紧接着通过一个变量depth来记录router-view的嵌套深度，当父实例parent存在并且parent._routerRoot !== parent时，代表parent不是根组件，那么就拿到父实例parent的$vnode($vnode为占位符vnode)。如果父实例parent的$vnode上data的routerView是true，那么代表着当前router-view的父组件也是通过router-view渲染出来了，那么此时depth就应该加一，我们来举一个例子：

    如果当前的路由配置为：
    const routes = [
      path: '/hi',
      component: Hi,
      children: [
        {
          path: 'hi-child',
          component: HiChild
        }
      ]
    ]

    那么我们在模板中通常会这样定义：
    // App.vue
    <template>
      <div>
        <router-view></router-view>
      </div>
    </template>

    // Hi.vue
    <template>
      <div>
        <router-view></router-view>
      </div>
    </template>

    // HiChild.vue
    <template>
      <div>
        hi-child
      </div>
    </template>

这样当我们访问'/hi'的时候，在App.vue中的router-view渲染的就是Hi.vue，当我们访问'/hi/hi-child'的时候，Hi.vue中
的router-view就渲染的是HiChild.vue。

此时App.vue中的router-view的父实例parent的$vnode也就是占位符Vnode是<App></App>，这个Vnode并不是<router-view>,
所以他的data属性上没有routerView这个属性，所以App.vue中的router-view的render函数中的depth就为0，而Hi.vue中的
router-view的父实例parent的$vnode也就是占位符Vnode是<router-view>，也就是说它是嵌套在<router-view>下的，所以
它的depth为1。

那么当计算完depth之后，将会通过depth拿到我们route.matched上的record：

    const matched = route.matched[depth]

而根据我们之前分析Vue Router时，route中的matched属性包含了当前path中的record以及它的所有父record，也就是说：

    '/hi': [hiRecord]
    '/hi/hi-child': [hiRecord, hiChildRecord]

那么不难看出，当前的router-view的深度就对应着当前path的record在route.matched中的位置，而拿到recored之后就可以通过
components属性获得将要渲染的组件的定义，并通过调用$createElement来返回Vnode。