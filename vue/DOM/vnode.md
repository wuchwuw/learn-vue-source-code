## vnode以及Vue中一些关于vnode的概念

Vue2.x中引入了虚拟DOM的概念，虚拟DOM就是一些用来描述真实DOM节点的对象，它包含的信息会告诉Vue页面上需要渲染
什么样的节点，Vue把这样的节点描述为"虚拟节点"，也常简写它为"VNode"。

其实Vue中vnode以及vnode的diff的实现都是参考了一个开源的库[snabbdom](https://github.com/snabbdom/snabbdom)，有兴趣的也可以去看一下它的实现。

接下来我们主要介绍一下Vue中一些关于Vnode的属性，了解这些属性的含义可以帮助我们更好的阅读Vue的源码。

### 组件vnode(placeholder vnode)

在Vue中，模板中的每个节点都会生成一个vnode，vnode的children属性包含着子vnode，而组件的节点也不例外，不过它生成的叫做组件vnode，如：

    <template>
      <App></App>
    </template>

App节点生成的vnode就是组件vnode，也称为App组件的placeholder vnode(占位vnode)，它和普通的vnode不同，它存在着componentOptions和componentInstance这两个属性：

    export interface VNodeComponentOptions {
      Ctor: typeof Vue;            // 组件定义
      propsData?: object;          // 组件上的属性
      listeners?: object;          // 组件上的自定义事件和原生事件
      children?: VNodeChildren;    // slot相关
      tag?: string;
    }

componentOptions包含着对即将要渲染的组件的一些描述，而componentInstance保存着将要渲染的组件的实例。

### 组件的根vnode

如果一个组件的模板是这样的：

    <template>
      <div>
        <span></span>
      </div>
    </template>

那么div节点生成的vnode就是整个组件的根vnode，它的children属性包含着它所有的子节点，形成一个树结构，所以根vnode包含了整个组件的所有节点的信息。

### 组件vnode和根vnode的关系

当一个子组件创建的过程中，组件vnode会作为_parentVnode传递到子组件的options中，提供子组件创建的信息。当子组件实例创建完成时，它将保存在实例
的$vnode上,也就是说：

    vm.$vnode === _parentVnode // 组件的占位vnode

当组件调用render函数来创建vnode时，会将创建完毕的根vnode保存到实例上的_vnode中，也就是说：

    vm._vnode // 组件的根vnode

并且，组件vnode会保存在根vnode的parent属性上：

    vm.$vnode === vm._vnode.parnet


在这里先介绍一些vnode的概念，主要是因为我自己刚开始看源码的时候也经常对这几个vnode的概念很模糊，导致看源码的过程中很不顺利，在接下来源码的分析中会经常看到这几个属性，希望先了解这几个概念可以帮助我们更好的分析源码。