## Vue Router

Vue Router是Vue.js官方的路由管理器，通过Vue Router我们可以很方便的构建单页应用，接下来我们将通过分析Vue Router的主要源码，了解其实现过程。

### 入口

通过[Vue Router](https://github.com/vuejs/vue-router)的git仓库拉取源码，打开package.json可以看到，Vue Router构建的时候执行的是:

    node build/build.js

我们找到build目录下的build.js文件，可以看到这里引入了configs.js作为配置来打包，在configs.js下可以看到:

    const config = {
      input: {
        input: resolve('src/index.js'),
        plugins: [
          flow(),
          node(),
          cjs(),
          replace({
            __VERSION__: version
          }),
          buble()
        ]
      },
      output: {
        file: opts.file,
        format: opts.format,
        banner,
        name: 'VueRouter'
      }
    }

在input配置项中，Vue Router以src/index.js文件作为入口来打包，所以我们可以通过这个文件来入手，分析Vue Router的源码。

### index.js

打开index.js可以看到，这里定义了VueRouter类，并在类上挂载了一个install方法。

    export default class VueRouter {
      ...
    }
    VueRouter.install = install

install方法将在Vue.use中作为插件的注册方法调用，如果不清楚Vue插件的可以移步官网[Vue-插件](https://cn.vuejs.org/v2/guide/plugins.html)。

按照Vue Router官网的教程，使用Vue Router有以下几步:

    //1.注册Vue Router插件
    Vue.use(VueRouter)

    // 2. 定义 (路由) 组件。
    // 可以从其他文件 import 进来
    const Foo = { template: '<div>foo</div>' }
    const Bar = { template: '<div>bar</div>' }

    // 3. 定义路由
    // 每个路由应该映射一个组件。 其中"component" 可以是
    // 通过 Vue.extend() 创建的组件构造器，
    // 或者，只是一个组件配置对象。
    // 我们晚点再讨论嵌套路由。
    const routes = [
      { path: '/foo', component: Foo },
      { path: '/bar', component: Bar }
    ]

    // 4. 创建 router 实例，然后传 `routes` 配置
    // 你还可以传别的配置参数, 不过先这么简单着吧。
    const router = new VueRouter({
      routes // (缩写) 相当于 routes: routes
    })

    // 5. 创建和挂载根实例。
    // 记得要通过 router 配置参数注入路由，
    // 从而让整个应用都有路由功能
    const app = new Vue({
      router
    }).$mount('#app')

在调用Vue.use注册Vue Router的时候，会调用之前定义在VueRouter类上的install方法，install方法定义在src/install.js下。

### install.js

    const registerInstance = (vm, callVal) => {
      let i = vm.$options._parentVnode
      if (isDef(i) && isDef(i = i.data) && isDef(i = i.registerRouteInstance)) {
        i(vm, callVal)
      }
    }

    Vue.mixin({
      beforeCreate () {
        if (isDef(this.$options.router)) {
          this._routerRoot = this
          this._router = this.$options.router
          this._router.init(this)
          Vue.util.defineReactive(this, '_route', this._router.history.current)
        } else {
          this._routerRoot = (this.$parent && this.$parent._routerRoot) || this
        }
        registerInstance(this, this)
      },
      destroyed () {
        registerInstance(this)
      }
    })

    Object.defineProperty(Vue.prototype, '$router', {
      get () { return this._routerRoot._router }
    })

    Object.defineProperty(Vue.prototype, '$route', {
      get () { return this._routerRoot._route }
    })

    Vue.component('RouterView', View)
    Vue.component('RouterLink', Link)

在install方法中，定义了registerInstance函数，这个函数是跟组件的实例相关的，后面会分析到。接着通过Vue.mixin传入了beforeCreate和destroyed两个钩子
可以看到beforeCreate在所有的组件实例上挂载了_routerRoot属性，而根组件的实例多挂载了一个_router属性，来自于new Vue时传入的VueRouter实例。然后调用VueRouter
实例的init方法。此外通过Object.defineProperty往Vue.prototype上挂载了$router和$route属性，这也是为什么我们可以在组件中通过this来访问这两个属性的原因，
可以看到他们分别指向了this._routerRoot._router和this._routerRoot._route，而根据上面的分析我们得知，_routerRoot属性指向的是根组件的实例，所以
this._routerRoot._router指向的就是new Vue时传入的VueRouter实例，this._routerRoot._route指向的就是根组件实例上的_route属性(这个属性将会在后面挂载到根实例上)。
最后，在Vue上注册了两个Component分别是RouterView和RouterLink，后面我们也会逐个分析。

分析完install方法之后，我们回到beforeCreate钩子中，可以看到当组件实例的$options上有router时，也就是根组件调用beforeCreate时，会调用VueRouter实例上的init方法。

### VueRouter init()

    在将

