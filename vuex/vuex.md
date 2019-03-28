## Vuex源码分析

当我们开发中大型项目时，如果有一些组件的状态需要同步或者当使用Vue提供的porps、$emit来通讯，一旦父子组件嵌套了太多层，那么中间的每一层都要写很多重复的代码，虽然新版本的Vue提供了provide/inject的功能方便父子组件通讯，但是当项目比较复杂，使用provide/inject可能会使数据的流动变的不是那么明确。这时我们可能需要一个储存和管理应用的所有组件的状态的东西 --Vuex。
接下来我们就从源码分析，看看Vuex是如何实现的。

### 入口

打开package.json，可以看到Vuex执行构建的脚本是：

    node build/build.main.js

这里和Vue的源码一样，引入config文件，对不同的打包方法配置不同的参数，我们可以看到入口都是'src/index.js'，我们找到src/index.js可以看到这里暴露了一个对象:

    export default {
      Store,
      install,
      version: '__VERSION__',
      mapState,
      mapMutations,
    }

这就是import Vuex返回的东西，回忆一下平时是如何使用Vuex:

    Vue.use(Vuex)
    ...
    const store = new Vuex.Store({
      ...
    })
    new Vue ({
      ...
      store,
      ...
    })

这里首先执行了Vue.use，并传入我们import进来的对象，如果你自己写过Vue插件就知道，Vue插件都需要通过暴露一个install方法，并执行Vue.use去注册插件。我们找到Vuex提供的install方法。

    export function install (_Vue) {
      if (Vue && _Vue === Vue) {
        if (process.env.NODE_ENV !== 'production') {
          console.error(
            '[vuex] already installed. Vue.use(Vuex) should be called only once.'
          )
        }
        return
      }
      Vue = _Vue
      applyMixin(Vue)
    }

这里通过_Vue保存传入的Vue来判断是否重复注册，并且调用applyMixin

#### applyMixin

    function (Vue) {
      const version = Number(Vue.version.split('.')[0])

      if (version >= 2) {
        Vue.mixin({ beforeCreate: vuexInit })
      }
      ...
    }

    function vuexInit () {
        const options = this.$options
        // store injection
        if (options.store) {
          this.$store = typeof options.store === 'function'
            ? options.store()
            : options.store
        } else if (options.parent && options.parent.$store) {
          this.$store = options.parent.$store
        }
      }
    }

在mixin.js文件中找到applyMixin的定义，这里只看Vue2.0的分支，可以看到Vuex通过调用Vue.mixin方法往组件的beforeCreate钩子混入了vuexInit方法，也就是说每个组件实例创建时，都会执行一次vuexInit，通过vuexInit拿到options上面的store对象，如果没有传入store也就是说不是根实例，那么就拿$parent.store，通过vuexInit方法，为每个组件实例上挂载了$store属性。接下来我们通过new Vue.Store()传入我们定义的state、mutations、actions、getters、modules，来创建store实例。

#### new Store()

    this._modules = new ModuleCollection(options)

在创建Store实例的过程中，会通过我们传入的state、mutations、getters、actions、moudules来创建modules树。

#### new ModuleCollection()

    class ModuleCollection {
      constructor (rawRootModule) {
        // register root module (Vuex.Store options)
        this.register([], rawRootModule, false)
      }

      get (path) {
        return path.reduce((module, key) => {
          return module.getChild(key)
        }, this.root)
      }

      register (path, rawModule, runtime = true) {
        if (process.env.NODE_ENV !== 'production') {
          assertRawModule(path, rawModule)
        }

        const newModule = new Module(rawModule, runtime)
        if (path.length === 0) {
          this.root = newModule
        } else {
          const parent = this.get(path.slice(0, -1))
          parent.addChild(path[path.length - 1], newModule)
        }

        // register nested modules
        if (rawModule.modules) {
          forEachValue(rawModule.modules, (rawChildModule, key) => {
            this.register(path.concat(key), rawChildModule, runtime)
          })
        }
      }
    }

可以看到，在new ModuleCollection的过程中调用了register方法，通过递归调用这个方法来遍历我们传入的modules，通过Module类
来创建一个模块，并通过一个归约函数来构建父子module。当第一次调用register时，传入的path是[]，那么将创建的module赋值给this.root代表这个是根模块，接着遍历所有modules，并将模块名concat到path上，递归调用register。当再次进入register方法时，
path数组里面已经有值，则通过get方法，并截取除了path的最后一位作为传入get方法的参数，通过reduce函数一层层遍历，找到当前创建的模块的parent，并构建父子关系。

    {                                                {
      getters: {},                                     root: {
      actions: {},                                       _children: {
      ...                                                  a: {},
      modules: {      ->  path = ['a', 'b']   ->           b: {},
        a: {},                                           }
        b: {}                                           }
      }                                               }
    }

#### new Store()

    this._actions = Object.create(null)
    this._actionSubscribers = []
    this._mutations = Object.create(null)
    this._wrappedGetters = Object.create(null)
    this._modules = new ModuleCollection(options)
    this._modulesNamespaceMap = Object.create(null)
    ...
    installModule(this, state, [], this._modules.root)

我们回到Store类中，可以看到还在store实例上挂载了很多属性，当创建完modules之后，会调用installModule来遍历我们创建的所有
模块并处理定义在模块上的state、getters、actions、mutations等

#### installModule

我们知道Vuex中是有命名空间这样的概念的，[Vuex命名空间](https://vuex.vuejs.org/zh/guide/modules.html)。

    function installModule (store, rootState, path, module, hot) {
      const isRoot = !path.length
      const namespace = store._modules.getNamespace(path)

      // register in namespace map
      if (module.namespaced) {
        store._modulesNamespaceMap[namespace] = module
      }

      // set state
      if (!isRoot && !hot) {
        const parentState = getNestedState(rootState, path.slice(0, -1))
        const moduleName = path[path.length - 1]
        store._withCommit(() => {
          Vue.set(parentState, moduleName, module.state)
        })
      }

      const local = module.context = makeLocalContext(store, namespace, path)

      module.forEachMutation((mutation, key) => {
        const namespacedType = namespace + key
        registerMutation(store, namespacedType, mutation, local)
      })

      module.forEachAction((action, key) => {
        const type = action.root ? key : namespace + key
        const handler = action.handler || action
        registerAction(store, type, handler, local)
      })

      module.forEachGetter((getter, key) => {
        const namespacedType = namespace + key
        registerGetter(store, namespacedType, getter, local)
      })

      module.forEachChild((child, key) => {
        installModule(store, rootState, path.concat(key), child, hot)
      })
    }

首先installModule也是一个递归的过程，首次执行时，传入的path为空，通过ModuleCollection类上定义的getNamespace方法来拿到namespace

    getNamespace (path) {
      let module = this.root
      return path.reduce((namespace, key) => {
        module = module.getChild(key)
        return namespace + (module.namespaced ? key + '/' : '')
      }, '')
    }

可以看到namespaced就是模块的key值和'/'的拼接，如果子模块设置了namespace为true，则模块的命名空间为parant key + '/' child key, 如果没有设置，则模块的命名空间和父模块一样！

接下来通过getNestedState方法传入path，来将当前模块的state添加到父模块的state上。然后执行makeLocalContext，可以理解为创建一个模块的上下文。

#### makeLocalContext

    function makeLocalContext (store, namespace, path) {
      const noNamespace = namespace === ''

      const local = {
        dispatch: noNamespace ? store.dispatch : (_type, _payload, _options) => {
          ...
          type = namespace + type
          ...
          return store.dispatch(type, payload)
        },

        commit: noNamespace ? store.commit : (_type, _payload, _options) => {
          ...
          type = namespace + type
          ...
          store.commit(type, payload, options)
        }
      }
      Object.defineProperties(local, {
        getters: {
          get: noNamespace
            ? () => store.getters
            : () => makeLocalGetters(store, namespace)
        },
        state: {
          get: () => getNestedState(store.state, path)
        }
      })

      return local
    }

    function makeLocalGetters (store, namespace) {
      const gettersProxy = {}

      const splitPos = namespace.length
      Object.keys(store.getters).forEach(type => {
        // skip if the target getter is not match this namespace
        if (type.slice(0, splitPos) !== namespace) return

        // extract local getter type
        const localType = type.slice(splitPos)

        // Add a port to the getters proxy.
        // Define as getter property because
        // we do not want to evaluate the getters in this time.
        Object.defineProperty(gettersProxy, localType, {
          get: () => store.getters[type],
          enumerable: true
        })
      })

      return gettersProxy
    }

可以看到makeLocalContext方法返回了一个local的对象，这个对象包含了4个属性，dispatch、commit、getters、state，如果有没有命名空间，那么dispatch、和commit都是使用全局的dispatch和commit，如果有命名空间，则重写dispatch和commit方法，重写的方法其实也是调用了全局的方法，只是在传入的type前面加上了命名空间。而getters、和state则通过Object.defineProperties来定义，getters这里也根据有没有命名空间来区分，如果有，则访问的是store.getters，否则访问的是makeLocalGetters方法返回的对象，当我们通过访问这个对象上的值时，其实访问的是store.getters上添加了命名空间的type的值。state就比较简单，直接通过path拿到当前模块的state值。

    module.forEachMutation((mutation, key) => {
      const namespacedType = namespace + key
      registerMutation(store, namespacedType, mutation, local)
    })

    module.forEachAction((action, key) => {
      const type = action.root ? key : namespace + key
      const handler = action.handler || action
      registerAction(store, type, handler, local)
    })

    module.forEachGetter((getter, key) => {
      const namespacedType = namespace + key
      registerGetter(store, namespacedType, getter, local)
    })

创建localContext后，遍历模块的所有mutation、action、getter，并分别添加到this._mutations、this._actions、this._wrappedGetters上，在添加的时候全部在key前面拼接了命名空间。

    function registerMutation (store, type, handler, local) {
      const entry = store._mutations[type] || (store._mutations[type] = [])
      entry.push(function wrappedMutationHandler (payload) {
        handler.call(store, local.state, payload)
      })
    }

这里可以看到！！在call hanler也就是执行mutations时，传入的是local.state!!之前我们分析过，local.state拿到的是当前模块的state，所以这也是为什么我们在当前模块中定义mutations时，接受的state参数是当前模块的state！

    function registerAction (store, type, handler, local) {
      const entry = store._actions[type] || (store._actions[type] = [])
      entry.push(function wrappedActionHandler (payload, cb) {
        let res = handler.call(store, {
          dispatch: local.dispatch,
          commit: local.commit,
          getters: local.getters,
          state: local.state,
          rootGetters: store.getters,
          rootState: store.state
        }, payload, cb)
        if (!isPromise(res)) {
          res = Promise.resolve(res)
        }
        if (store._devtoolHook) {
          return res.catch(err => {
            store._devtoolHook.emit('vuex:error', err)
            throw err
          })
        } else {
          return res
        }
      })
    }

actions也同理，第一个参数接受的是一个对象，上面有commit、dispatch、getters，拿的都是local对象里的，这也是为什么我们在actions里使用commit、dispatch时，传入的type不需要加命名空间，因为我们在重写commit、dispatch的时候已经将命名空间拼接上去了！

    function registerGetter (store, type, rawGetter, local) {
      if (store._wrappedGetters[type]) {
        if (process.env.NODE_ENV !== 'production') {
          console.error(`[vuex] duplicate getter key: ${type}`)
        }
        return
      }
      store._wrappedGetters[type] = function wrappedGetter (store) {
        return rawGetter(
          local.state, // local state
          local.getters, // local getters
          store.state, // root state
          store.getters // root getters
        )
      }
    }

getters也同理，传入当前命名空间的state、getters还有根state、getters。

执行完installModule后，调用resetStoreVM方法来创建一个Vue实例，用来挂载state、和getters

#### resetStoreVM

    store.getters = {}
    const wrappedGetters = store._wrappedGetters
    const computed = {}
    forEachValue(wrappedGetters, (fn, key) => {
      // use computed to leverage its lazy-caching mechanism
      computed[key] = () => fn(store)
      Object.defineProperty(store.getters, key, {
        get: () => store._vm[key],
        enumerable: true // for local getters
      })
    })

    store._vm = new Vue({
      data: {
        $$state: state
      },
      computed
    })

遍历_wrappedGetters上所有的getters并添加到computed对象上，通过Object.defineProperty将store上getters的访问代理到vm实例上，并且通过new Vue传入state作为data，getters作为computed。就是在这个地方Vuex将所有的getters添加到store.getters上，
所以当我们创建localContext的getters属性时，可以从store.getters上拿到我们需要的getters。

Vuex通过Vue来挂载state和getters，也就是说state也会变成响应式对象，state、getters的更改也可以触发视图的重新渲染。