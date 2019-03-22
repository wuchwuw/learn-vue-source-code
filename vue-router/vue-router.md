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

在install方法中，定义了registerInstance函数，这个函数是跟组件的实例相关的，后面会分析到。接着通过Vue.mixin混入了beforeCreate和destroyed两个钩子
可以看到beforeCreate在所有的组件实例上挂载了_routerRoot属性，而根组件的实例多挂载了一个_router属性，来自于new Vue时传入的VueRouter实例。然后调用VueRouter
实例的init方法。此外通过Object.defineProperty往Vue.prototype上挂载了$router和$route属性，这也是为什么我们可以在组件中通过this来访问这两个属性的原因，
可以看到他们分别指向了this._routerRoot._router和this._routerRoot._route，而根据上面的分析我们得知，_routerRoot属性指向的是根组件的实例，所以
this._routerRoot._router指向的就是new Vue时传入的VueRouter实例，this._routerRoot._route指向的就是根组件实例上的_route属性(这个属性将会在后面挂载到根实例上)。
最后，在Vue上注册了两个Component分别是RouterView和RouterLink，后面我们也会逐个分析。

分析完install方法之后，我们得知当根组件调用beforeCreate时，会调用VueRouter实例上的init方法。不过在分析init方法之前，我们先来分析一个new VueRouter的过程。


### new VueRouter


    const routes = [
      { path: '/foo', component: Foo },
      { path: '/bar', component: Bar }
    ]

    const router = new VueRouter({
      routes // (缩写) 相当于 routes: routes
    })

根据上面的例子，我们在注册完Vue Router插件之后，就会传入路由配置并创建Vue Router实例，我们回到src/index.js下，来分析Vue Router实例创建的主要流程：

    this.matcher = createMatcher(options.routes || [], this)

    this.mode = mode

    switch (mode) {
      case 'history':
        this.history = new HTML5History(this, options.base)
        break
      case 'hash':
        this.history = new HashHistory(this, options.base, this.fallback)
        break
      case 'abstract':
        this.history = new AbstractHistory(this, options.base)
        break
      default:
        if (process.env.NODE_ENV !== 'production') {
          assert(false, `invalid mode: ${mode}`)
        }
    }

1、通过createMatcher方法来创建matcher。

2、通过传入的mode参数来创建history实例，我们将分析比较常用的history模式，也就是HTML5History类创建的history实例。


### 创建matcher，根据路由配置生成Record

#### createMatcher

    const { pathList, pathMap, nameMap } = createRouteMap(routes)

    function addRoutes () {...}  

    function match () {...}

    function redirect () {...}

    function alias () {...}

    function _createRoute () {...}

    return {
      match,
      addRoutes
    }

可以看到createMatcher方法首先调用createRouteMap拿到pathList, pathMap, nameMap这三个变量，然后创建了addRoutes、match、redirect、alias、_createRoute这几个方法，最后返回了一个对象包含了match、addRoutes方法，这里我们主要先分析createRouteMap。

#### createRouteMap
    
    const pathList = oldPathList || []
    const pathMap = oldPathMap || Object.create(null)
    const nameMap = oldNameMap || Object.create(null)

    routes.forEach(route => {
      addRouteRecord(pathList, pathMap, nameMap, route)
    })

createRouteMap首先创建了pathList、pathMap、nameMap三个变量，然后遍历传入的路由配置，执行addRouteRecord方法。

#### addRouteRecord

    const { path, name } = route
    const normalizedPath = normalizePath(
      path,
      parent,
      pathToRegexpOptions.strict
    )

首先拿到当前路由配置的name和path，并且执行normalizePath方法，来合并父子路由的path，合并的规则如下：

    if (path[0] === '/') return path               // 如果path的一个字符为'/'直接返回path
    if (parent == null) return path                // 如果parent为null，直接返回path
    return cleanPath(`${parent.path}/${path}`)     // 否则返回拼接 'parent.path/path',并且使用cleanPath清除路径中存在的'//'

拼接好路径之后，通过路由的定义来创建record对象，record对象是基于我们传入的路由配置而创建的，这里可以不用管里面的属性是什么意思，只需知道每个路由都会创建
一个Record对象。

    const record: RouteRecord = {
      path: normalizedPath,
      regex: compileRouteRegex(normalizedPath, pathToRegexpOptions),
      components: route.components || { default: route.component },
      instances: {},
      name,
      parent,
      matchAs,
      redirect: route.redirect,
      beforeEnter: route.beforeEnter,
      meta: route.meta || {},
      props: route.props == null
        ? {}
        : route.components
          ? route.props
          : { default: route.props }
    }

创建好Record对象之后，判断配置是否有children，如果存在则递归调用addRouteRecord来创建children的Record

    if (route.children) {
      route.children.forEach(child => {
        const childMatchAs = matchAs
          ? cleanPath(`${matchAs}/${child.path}`)
          : undefined
        addRouteRecord(pathList, pathMap, nameMap, child, record, childMatchAs)
      })
    }

递归创建完毕后，在回溯的过程中将所有路由创建的Record对象保存在pathList、pathMap、nameMap中

    if (!pathMap[record.path]) {
      pathList.push(record.path)
      pathMap[record.path] = record
    }

    if (name) {
      if (!nameMap[name]) {
        nameMap[name] = record
      } else if (process.env.NODE_ENV !== 'production' && !matchAs) {
        warn(
          false,
          `Duplicate named routes definition: ` +
          `{ name: "${name}", path: "${record.path}" }`
        )
      }
    }

如果我们传入了这样的配置：

    const routes = [
      {
        path: '/hello',
        component: Hello,
        name: 'hello',
        children: [
          {
            path: 'hello-child',
            component: HelloChild,
            name: 'hello-child'
          }
        ]
      },
      {
        path: '/hi',
        component: Hi,
        name: 'hi',
        children: [
          {
            path: 'hi-child',
            component: HiChild,
            name: 'hi-child'
          }
        ]
      }
    ]

那么通过调用addRouteRecord方法之后将产生这样的结果：

    pathList = ["/hello/hello-child", "/hello", "/hi/hi-child", "/hi"]
    pathMap = {
      '/hello/hello-child': helloChildRecord
      '/hello': helloRecord
      '/hi/hi-child': hiChildRecord
      '/hi': hiRecord
    }
    nameMap = {
      'hello': helloRecord,
      'hello-child': helloChildRecord,
      'hi': hiRecord,
      'hi-child': hiChildRecord
    }

### 创建histoty实例

分析完createMatcher的过程，接下来根据传入的mode来创建history实例，这里我们选择常用的history mode来分析。

#### new HTML5History

    class HTML5History extends History {
      constructor (router: Router, base: ?string) {
        ...
        window.addEventListener('popstate', e => {
          ...
          this.transitionTo(location, route => {
            if (supportsScroll) {
              handleScroll(router, route, current, true)
            }
          })
        })
      }

      go (n: number) {
        window.history.go(n)
      }

      push (location: RawLocation, onComplete?: Function, onAbort?: Function) {
        const { current: fromRoute } = this
        this.transitionTo(location, route => {
          pushState(cleanPath(this.base + route.fullPath))
          handleScroll(this.router, route, fromRoute, false)
          onComplete && onComplete(route)
        }, onAbort)
      }

      replace (location: RawLocation, onComplete?: Function, onAbort?: Function) {
        const { current: fromRoute } = this
        this.transitionTo(location, route => {
          replaceState(cleanPath(this.base + route.fullPath))
          handleScroll(this.router, route, fromRoute, false)
          onComplete && onComplete(route)
        }, onAbort)
      }
    }

可以看到创建HTML5History实例的过程中，监听了浏览器的popstate事件，每次当浏览器前进或者回退时，将触发popstate事件，此外Vue Router实例中的push、replace方法
都是调用history实例中的push、replace，不管是popstate事件、还是push、replace方法最终都是调用transitionTo方法来完成一次路由的切换。

### VueRouter init()

    this.apps.push(app)

    const history = this.history


init方法首先将传进来的根组件实例push到apps中，这是因为一个Vue应用可能存在多个Vue实例，不过并不常用。然后根据不同的history来调用他们的transitionTo方法，history
的创建是在new VueRouter的时候，可以看到根据我们传进来的mode选择创建不同的history实例，这里选择我们经常用的history模式来分析。

      switch (mode) {
        case 'history':
          this.history = new HTML5History(this, options.base)
          break
        case 'hash':
          this.history = new HashHistory(this, options.base, this.fallback)
          break
        case 'abstract':
          this.history = new AbstractHistory(this, options.base)
          break
        default:
          if (process.env.NODE_ENV !== 'production') {
            assert(false, `invalid mode: ${mode}`)
          }
      }

### HTML5History

    