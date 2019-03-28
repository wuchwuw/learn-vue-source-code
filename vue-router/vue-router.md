## Vue Router

Vue Router是Vue.js官方的路由管理器，通过Vue Router我们可以很方便的构建单页应用，接下来我们将通过分析Vue Router的主要源码，了解其实现过程。

### 入口

通过[Vue Router](https://github.com/vuejs/vue-router)的git仓库拉取源码，打开package.json可以看到，Vue Router构建的时候执行的是:

    node build/build.js

我们找到build目录下的build.js文件，可以看到这里引入了configs.js作为配置来打包，在configs.js下可以看到:

    const config = {
      input: {
        input: resolve('src/index.js'),
        ....
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
    const routes = [
      { path: '/foo', component: Foo },
      { path: '/bar', component: Bar }
    ]

    // 4. 创建 router 实例，然后传 `routes` 配置
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
都是调用history实例中的push、replace，而这些方法最终都是执行浏览器原生的history.pushState和history.replaceState方法。不过无论是popstate事件、还是push、
replace方法最终都是调用transitionTo方法来完成一次路由的切换。transitionTo方法定义在HTML5History的父类History中，这里我们先不分析transitionTo这个方法，因
为我们主要的目的还是了解HTML5History实例创建的过程。

分析完Vue Router实例的主要创建过程之后，我们回到之前在install方法中通过Vue.mixin传入的beforeCreate钩子中，当我们创建好Vue Router实例后，就会调用new Vue来
传入我们生成的Vue Router实例，在new Vue的过程中，根组件将被实例化，这时候就会调用根组件的beforeCreate钩子，也就是会调用Vue Router实例上的init方法。

#### VueRouter init()

    this.apps.push(app)

    const history = this.history
    ...
    history.transitionTo(history.getCurrentLocation())
    ...
    history.listen(route => {
      this.apps.forEach((app) => {
        app._route = route
      })
    })

init方法首先将传进来的根组件实例push到apps中，这是因为一个Vue应用可能存在多个Vue实例，不过不太常用。然后拿到之前创建的HTML5History的实例并且调用一次transitionTo
方法来完成第一次路由切换，传入的参数是通过history.getCurrentLocation方法返回的当前的location，简单来说就是返回URL除去域名的部分，例如:

    如果第一次进来的url是: https://github.com  那么传入的location就是 '/'
    如果第一次进来的url是: https://github.com/wuch1995?xx=xx  那么传入的location就是 '/wuch1995?xx=xx'

具体的getCurrentLocation方法你可以自己去分析，其实就是简单的字符串拼接。

<!-- 最后通过history.listen来监听一个函数，可以看到这个函数接收了一个route参数，并把它赋值到根组件实例上的_route属性，是不是有点眼熟？我们之前
分析install方法时候有提到根组件在调用beforeCreate的时候会通过Vue.util.defineReactive(this, '_route', this._router.history.current)，往根组件实例上添加
一个响应式属性_route，而这个_route的初始值是history实例上的current，我们先不管这个值是什么，只需要知道 -->

#### transitionTo

transitionTo方法定义在history/base.js中

    const route = this.router.match(location, this.current)

    this.confirmTransition(route, () => { ... })

首先，通过传入的location和this.current拿到下一个路由的route对象，那么this.router.match调用的就是我们createMathcer返回的对象中的match方法，我们可以回到这个方法看一下。

### match

    const location = normalizeLocation(raw, currentRoute, false, router)
    const { name } = location

    const record = nameMap[name]

    if (record) {
      return _createRoute(record, location, redirectedFrom)
    } else {
      if (matchRoute(record.regex, location.path, location.params)) {
        return _createRoute(record, location, redirectedFrom)
      }
    }

首先将我们传入的location进行格式化，如果传入的location是字符串的话，会被转化成一个location对象，接着通过location的name来拿到我们之前生成的record，如果路由配置里没有写name这个属性，那么就通过正则表达式来匹配record，拿到recored后通过_createRoute来生成下一个路由的route对象。

#### createRoute

    const route: Route = {
      name: location.name || (record && record.name),
      meta: (record && record.meta) || {},
      path: location.path || '/',
      hash: location.hash || '',
      query,
      params: location.params || {},
      fullPath: getFullPath(location, stringifyQuery),
      matched: record ? formatMatch(record) : []
    }
    return Object.freeze(route)

createRoute通过我们传入的location和record来创建route对象，并调用Object.freeze禁止修改route，这里我们可以总结一下创建route的过程：

    location  ->  record
    record + location + currentRoute -> new route

创建完route之后，我们回到transitionTo中，接下来就是执行confirmTransition这个方法，并传入我们创建好的route。


#### confirmTransition

confirmTransition逻辑比较多，但是我们可以分成以下几部分来分析：

1、拿到将要执行的所有钩子并push到队列中

2、依次执行所有钩子

3、完成一次路由的切换，更新currentRoute，并且触发视图的重新渲染。


首先我们来看在一次路由的切换中，Vue Router将会执行什么钩子，并且他们的执行顺序是怎样的？以下是从Vue Router的官网中拷贝来的：

    导航被触发。
    在失活的组件里调用离开守卫。
    调用全局的 beforeEach 守卫。
    在重用的组件里调用 beforeRouteUpdate 守卫 (2.2+)。
    在路由配置里调用 beforeEnter。
    解析异步路由组件。
    在被激活的组件里调用 beforeRouteEnter。
    调用全局的 beforeResolve 守卫 (2.5+)。
    导航被确认。
    调用全局的 afterEach 钩子。
    触发 DOM 更新。
    用创建好的实例调用 beforeRouteEnter 守卫中传给 next 的回调函数。

可以看到这里面有全局的钩子，也包含组件内的钩子。而这里面存在这几个定义：失活的组件、重用的组件、被激活的组件，接下来我们就将分析这几个定义。

    const {
      updated,
      deactivated,
      activated
    } = resolveQueue(this.current.matched, route.matched)

可以看到在confirmTransition方法中先会调用resolveQueue来拿到deactivated、updated、activated，这也就是失活的组件、重用的组件、被激活的组件。我们可以通过一个
例子来说明这三个变量到底包含了什么，比如当前path是'/bar/foo',而将要跳转path是'/bar',那么：

    this.current.matched = [
      rootRecord, barRecord, fooRecord
    ]

    this.route = [
      rootRecord, barRecord
    ]

    deactivated = [fooRecord]

    updated = [rootRecord, barRecord]

    activated = []

也就是说，route中的matched属性包含了当前path中的record以及它的所有父record，deactivated、activated就是他们的差集，updated就是他们的交集。并且拿到record上
的组件实例，返回实例上相应的钩子：

    const queue: Array<?NavigationGuard> = [].concat(
      // in-component leave guards
      extractLeaveGuards(deactivated),       // 失活组件的beforeRouteLeave
      // global before hooks
      this.router.beforeHooks,               // 全局beforeEach钩子
      // in-component update hooks
      extractUpdateHooks(updated),           // 重用的组件beforeRouteUpdate钩子
      // in-config enter guards
      activated.map(m => m.beforeEnter),     // 被激活组件的路由配置里的beforeEnter钩子
      // async components
      resolveAsyncComponents(activated)      // 解析异步路由组件
    )

接下来我们分析一下这个队列的执行：

    const iterator = (hook: NavigationGuard, next) => {
      hook(route, current, () => {
        next()
      })
    }

    runQueue(queue, iterator, () => {
      const enterGuards = extractEnterGuards(activated, postEnterCbs, isValid)
      const queue = enterGuards.concat(this.router.resolveHooks)
      runQueue(queuem iterator, () => {})
    })

    function runQueue (queue: Array<?NavigationGuard>, fn: Function, cb: Function) {
      const step = index => {
        if (index >= queue.length) {
          cb()
        } else {
          if (queue[index]) {
            fn(queue[index], () => {
              step(index + 1)
            })
          } else {
            step(index + 1)
          }
        }
      }
      step(0)
    }

简化执行的代码后可以看到，runQueue传入了将要执行的队列，一个迭代的函数iterator，还有队列执行完之后执行的回调函数，
在runQueue中定义了step方法并传入index，通过index来判断队列是否执行完毕，如果index大于或等于队列的长度，则执行回调函数，
否则就调用iterator并传入当前要执行的函数，以及stpe(index + 1)，当传入iterator的函数也就是hook执行完毕时，调用next()
也就是step(index + 1)顺序执行队列中的下一个函数。在这里我们也可以回想一下平时定义的路由钩子，每个路由钩子都可以接收3个参数，
to、from、next，当我们执行完一个钩子之后，都会调用next来执行下一个钩子，当我们执行钩子内的next时，就会执行iterator中hook传
入的回调函数从而执行下一个钩子。

当整个queue执行完毕时，在回调函数中又执行了以下的逻辑：

    // 被激活的组件里的beforeRouteEnter钩子
    const enterGuards = extractEnterGuards(activated, postEnterCbs, isValid)
    // 调用全局的 beforeResolve钩子
    const queue = enterGuards.concat(this.router.resolveHooks)
    runQueue(queuem iterator, () => {
      onComplete(route)
    })

因为这里必须等到异步组件加载完毕才可以拿到异步组件的实例，也就是说要等到queue执行完毕之后才可以拿到激活组件的钩子。拿到激活组件的钩子
之后，又继续执行一遍runQueue，把剩余的钩子执行完毕。当所有钩子执行完毕之后，在第二个runQueue的回调函数中会执行onComplete方法，此时
路由的切换就完成了，同时就会触发视图的重新渲染。

### 组件重新渲染

onComplete是在调用confirmTransition中传入的，而触发重新渲染的关键逻辑就是：

    this.updateRoute(route)

    // this.updateRoute
    updateRoute (route: Route) {
      const prev = this.current
      this.current = route
      this.cb && this.cb(route)
      this.router.afterHooks.forEach(hook => {
        hook && hook(route, prev)
      })
    }

updateRoute将替换当前的currentRoute，并且调用this.cb，而this.cb这个函数的赋值发生在我们调用Vue Router实例的init方法中：

    history.listen(route => {
      this.apps.forEach((app) => {
        app._route = route
      })
    })

    // history.listen
    listen (cb: Function) {
      this.cb = cb
    }

也就是说当this.cb执行时，根组件上的_route属性将会被替换成当前的currentRoute，而_route属性是在执行install方法时，通过defineReactive
挂载到根组件实例上的，也就是说它是一个响应式的属性：

    Vue.util.defineReactive(this, '_route', this._router.history.current)

所以当_route属性被重新赋值时，就会触发视图的重新渲染。一旦视图重新渲染，router-view组件就可以重新拿到组件的定义来渲染对应的组件。
那么router-view是如何知道它将要渲染什么组件，以及对于多个router-view组件嵌套的情况，它又是如何来处理的呢？接下来我们就来分析
router-view组件的源码，了解它的实现原理。

下一篇:[router-view](./router-view.md)