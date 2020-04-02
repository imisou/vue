# 组件化流程(生命周期流程)

首先我们已组建生命周期图示镇图：

![image](https://note.youdao.com/yws/public/resource/60116e0de35cc4930a98ea2291671610/xmlnote/EAC1B8AD10634C3AB6E6549BD6DE8E3A/11699)


### 第一步： new Vue();

在实际中我们都会通过new Vue()的方式去绑定一个根组件

```js
var vue = new Vue({
    el: "#app",
    components:{
        buttonCounter
    },
    data: function() {
        return {
            name: ""
        }
    }
})
```
那么这时候其调用的是
```js
/**
 * 初始化生成 Vue 全局函数
 * @author guzhanghua
 * @param {*} options
 */
function Vue(options) {
    if (process.env.NODE_ENV !== 'production' &&
        !(this instanceof Vue)
    ) {
        warn('Vue is a constructor and should be called with the `new` keyword')
    }
    this._init(options)
}
```
然后执行 this._init()方法，这时候就到了第二步了

### 第二步：Init(Events & Lifecycle)

在第二步的时候执行 _init()方法。我们看_init() 方法
```js
Vue.prototype._init = function(options ? : Object) {
    ...
    // expose real self
    vm._self = vm
    initLifecycle(vm)
    initEvents(vm)
    initRender(vm)
    // 触发beforeCreate回调
    callHook(vm, 'beforeCreate')
    ...
}
```
这时候执行了 initLifecycle(vm)、 initEvents(vm) 就像生命周期中的(Events & Lifecycle)，但是这时候还进行了initRender(vm)，为什么没有提及？

因为initRender(vm)主要是初始化 $mount的时候回调render函数的一些属性和方法 如_vnode、_c()。

然后触发了  callHook(vm, 'beforeCreate') beforeCreate生命周期函数。

因为这时候没有进行 initState 所以这时候访问不了组件响应式属性 this.xxx

### 第三步： Init(injection & reactiivity)

这个过程主要是初始化 inject属性 和响应式数据，即在下一个生命周期的时候可以访问组件的响应式数据属性。

```js
Vue.prototype._init = function(options ? : Object) {
    ...
    // 初始化高阶属性inject
    initInjections(vm) // resolve injections before data/props
    initState(vm)
    // 初始化高阶属性 provide
    initProvide(vm) // resolve provide after data/props
    callHook(vm, 'created')
    ...
}
```

### 第四步：Has 'el' option

```js
Vue.prototype._init = function(options ? : Object) {
    ...
    // 对于子组件类型 其都不会通过el去绑定 模板DOM。而对于跟组件其会el:'#app' 所以需要触发 vm.$mount()方法。
    // 子组件在什么时候触发 $mount方法？  在init 钩子函数 最后调用 child.$mount
    if (vm.$options.el) {
        vm.$mount(vm.$options.el)
    }
}
```

这一步主要是做什么？

我们Vue组件初始化完成了那么就应该 安装组件了。这一步是一个条件语句,我们在new Vue的时候可以

```js
var vue = new Vue({
    el: "#app",
})
```
当然也可以这样
```js
var vue = new Vue({ el: "#app" }).$mount('#app')
```
可见这一步主要是可以实例化的时候就执行$mount，也可以之后在vm.$mount()手动去挂载。

### 第五步： Compile template into render function

其实这一步就是执行 $mount函数。

对于带编译的版本其 $mount方法的定义在

###### src\platforms\web\entry-runtime-with-compiler.js

```js
const mount = Vue.prototype.$mount
Vue.prototype.$mount = function(
    el ? : string | Element,
    hydrating ? : boolean
): Component {
    ...   // 编译 el 元素
    return mount.call(this, el, hydrating)
}
```
然后调用 mount.call()

###### src\platforms\web\runtime\index.js

```js
// public mount method
Vue.prototype.$mount = function(
    el ? : string | Element,
    hydrating ? : boolean
): Component {
    el = el && inBrowser ? query(el) : undefined
    return mountComponent(this, el, hydrating)
}
```
真正关键的是 mountComponent(this, el, hydrating)

###### src\core\instance\lifecycle.js

```js
export function mountComponent(
    vm: Component,
    el: ? Element,
    hydrating ? : boolean
): Component {
    vm.$el = el

    // 判断此时是否存在render 函数，
    // Vue中不管是通过el,template,render() 3种方式中的一种去获取模板的 都在最后将其转换成render函数，
    if (!vm.$options.render) {
        vm.$options.render = createEmptyVNode
        ...
    }
    // 触发钩子函数 看生命周期  之前 Compile template into render function or Compile el's outerHTML as template
    callHook(vm, 'beforeMount')

    return vm
}
```

所以在 callHook(vm, 'beforeMount') 触发 beforeMount生命周期函数的之前主要就是将template el转换成<font color=red> render函数</font>

### 第六步： <font color=red> Create vm.$el and replace 'el' with it </font>

这是最重要的一步也是最复杂的一步。

首先我们需要了解一个概念，即Vue不是直接将 我们的template 或者el直接转换成DOM插入DOM树中。而是 先编译成可执行的代码字符串(HTML -> JS),然后执行此代码装换成vnode(JS -> VNode)，然后patch将vnode变成真正的DOM插入DOM树(VNode -> DOM)。

而这一步就是执行render 将JS转成vnode、然后patch将vnode变成真正的DOM插入DOM树。

下面我们看代码

###### src\core\instance\lifecycle.js

```js
export function mountComponent(
    vm: Component,
    el: ? Element,
    hydrating ? : boolean
): Component {
    ...
    let updateComponent
        /* istanbul ignore if */
    if (process.env.NODE_ENV !== 'production' && config.performance && mark) {

    } else {
        // 创建一个更新组件方法
        updateComponent = () => {
            vm._update(vm._render(), hydrating)
        }
    }
    new Watcher(vm, updateComponent, noop, {
        // 在我们的更新队列中 其更新方法 是sort排列 使得 子组件在父组件之后更新
        // 先调用before 然后调用 watcher.run()方法
        before() {
            if (vm._isMounted) {
                callHook(vm, 'beforeUpdate')
            }
        }
    }, true /* isRenderWatcher */ )
    hydrating = false
    if (vm.$vnode == null) {
        vm._isMounted = true
        callHook(vm, 'mounted')
    }
    return vm
}
```
其先创建一个函数updateComponent作为watcher的 expOrFns参数。然后在watcher的get的时候回调执行此方法。
然后触发 vm._render()

```js
/**
 * 作用  就是 执行组件上定义的 render函数  生成 一个vnode
 * @return {vnode} [组件vnode]
 */
Vue.prototype._render = function(): VNode {
    // 第一次 vm = new Vue()
    const vm: Component = this
    const { render, _parentVnode } = vm.$options
    ... // 处理slot
    vm.$vnode = _parentVnode
    // render self
    let vnode
    try {
        // 调用 组件定义的render函数
        vnode = render.call(vm._renderProxy, vm.$createElement)
    } catch (e) {

    }
    // return empty vnode in case the render function errored out
    vnode.parent = _parentVnode
    return vnode
}
```
可见这一步最重要的时候 通过render.call(xxx)去回调执行我们的render函数。在render函数中通过 vm._createElement()方法返回一个vnode的VirtualDOM 树。

然后触发 _update()方法

```js
/*
    可见 _update()方法触发的时机有两种。
    1、 当组件初始化渲染的时候 此时组件从AST -> VNode 但是没有生成DOM元素 此时触发_update 进行 VNode -> DOM的过程
    2、 当组件发生更新的时候  此时响应式数据触发 set方法 然后 dep.notify() 去通知渲染Watcher进行重新getter方法
    此时也会触发 _update() 方法
 */
Vue.prototype._update = function(vnode: VNode, hydrating ? : boolean) {
    ...
    if (!prevVnode) {
        // initial render
        vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */ );
    } else {
        // 当组件更新的时候触发 preVnode 为旧的组件VNode vnode为新render生成的VNode
        // updates
        vm.$el = vm.__patch__(prevVnode, vnode)
    }

    if (vm.$el) {
        vm.$el.__vue__ = vm
    }
}
```
调用 __patch__() 方法将VNode转成DOM。  

#### VNode转DOM过程中，元素创建的过程、插入的次序、父子组件DOM元素如何插入
首先我们还是看 patch 方法

```js
return function patch(oldVnode, vnode, hydrating, removeOnly) {
    ... // Destory
    let isInitialPatch = false
    const insertedVnodeQueue = []
    // 如果没有真实的 DOM 那么 就可能是 一开始创建的时候  或者 懒加载的组件类型
    // 那么 直接调用createEle 生成DOM
    if (isUndef(oldVnode)) {
        // empty mount (likely as component), create new root element
        isInitialPatch = true
        createElm(vnode, insertedVnodeQueue)
    } else {
        // 第一步 oldVode = #app  所以 oldVnode.nodeType = 1；
        const isRealElement = isDef(oldVnode.nodeType)
        if (!isRealElement && sameVnode(oldVnode, vnode)) {
            // patch existing root node
            patchVnode(oldVnode, vnode, insertedVnodeQueue, removeOnly)
        } else {

            // replacing existing element
            const oldElm = oldVnode.elm
            const parentElm = nodeOps.parentNode(oldElm)
            // create new node
            createElm(
                vnode, // 当前的组件vnode
                insertedVnodeQueue,
                // extremely rare edge case: do not insert if old element is in a
                // leaving transition. Only happens when combining transition +
                // keep-alive + HOCs. (#4590)
                oldElm._leaveCb ? null : parentElm, // 父元素
                nodeOps.nextSibling(oldElm)
            )
        }
    }
    // 将vnode转成dom树后，调用 insertedVnodeQueue队列中的 insert的钩子函数
    invokeInsertHook(vnode, insertedVnodeQueue, isInitialPatch)
    return vnode.elm
}
```
如对于 跟组件 我们oldVnode 就是我们 div#app元素，且 isRealElement === true; 然后执行
 createElm(vnode, insertedVnodeQueue,oldElm._leaveCb ? null : parentElm, nodeOps.nextSibling(oldElm) )这时候 parentElm 为 <body>元素

```js
function createElm(
    vnode,
    insertedVnodeQueue,
    parentElm,
    refElm,
    nested,
    ownerArray,
    index
) {

    // 是否是嵌套的内部组件
    vnode.isRootInsert = !nested // for transition enter check
    // 如果为true 说明 此当前处理的vnode是一个组件
    // 如果是undefined 说明当前处理的vnode为元素节点
    if (createComponent(vnode, insertedVnodeQueue, parentElm, refElm)) {
        return
    }
    // 元素节点 保存其data数据
    const data = vnode.data
    // 获取其子vnode
    const children = vnode.children
    const tag = vnode.tag

    if (isDef(tag)) {

        // 创建当前 节点元素
        vnode.elm = vnode.ns ?
            nodeOps.createElementNS(vnode.ns, tag) :
            nodeOps.createElement(tag, vnode)
        setScope(vnode)
            /* istanbul ignore if */
        if (__WEEX__) {

        } else {
            // 处理子节点
            createChildren(vnode, children, insertedVnodeQueue)
            if (isDef(data)) {
                invokeCreateHooks(vnode, insertedVnodeQueue)
            }
            // 在父节点上 插入处理好的此节点
            insert(parentElm, vnode.elm, refElm)
        }
        if (process.env.NODE_ENV !== 'production' && data && data.pre) {
            creatingElmInVPre--
        }
        // 如果节点是注释节点
    } else if (isTrue(vnode.isComment)) {
        vnode.elm = nodeOps.createComment(vnode.text)
        insert(parentElm, vnode.elm, refElm)
    } else {
        // 其他说明这是一个文本节点
        vnode.elm = nodeOps.createTextNode(vnode.text)
        insert(parentElm, vnode.elm, refElm)
    }
}
```

对于 div#app 元素其肯定不是组件 所以createComponent(vnode, insertedVnodeQueue, parentElm, refElm)为false。然后执行到  vnode.elm = vnode.ns ? nodeOps.createElementNS(vnode.ns, tag) : nodeOps.createElement(tag, vnode)。
通过函数柯里化方法定义了各平台的nodeOps(DOM操作方法)，去创建当前 div#app元素。
然后 createChildren(vnode, children, insertedVnodeQueue)去深度处理子元素，然后insert(parentElm, vnode.elm, refElm)。

可以组件中 元素的创建是 由上而下。然后通过深度遍历子节点，将各个子节点插入到其父节点的node.elm上

如：

![image](https://note.youdao.com/yws/public/resource/60116e0de35cc4930a98ea2291671610/xmlnote/BC9A877DCBDB4C0FAE20F650F8F26320/11893)

其先创建 div#app 元素，

然后createChildren() 处理子节点。

###### 第一个分支

然后创建 H3节点

然后createChildren() 处理 H3 的子节点；

然后创建文本节点 TEXT(APP);

然后createChildren() 没有获取到子节点，执行insert将文本节点插入到父vnode.elm上，即使得 VNode(H3).elm = H3>APP;

然后执行H3的插入操作 使得 VNode(app).elm = div>h3>app;

###### 第二个分支

一样像第一个分支一样创建父节点，然后处理子节点，然后将子节点插入到父节点的vnode.elm上，这样由上而下创建，处理；自下而上插入生成树。

##### 第三个分支

这时候遇到组件那么 createComponent()将为true；在处理子组件的时候其也是这样。

```js
function createComponent(vnode, insertedVnodeQueue, parentElm, refElm) {
    let i = vnode.data
    if (isDef(i)) {
        const isReactivated = isDef(vnode.componentInstance) && i.keepAlive
        // 在create-component.js 中
        if (isDef(i = i.hook) && isDef(i = i.init)) {
            i(vnode, false /* hydrating */ )
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
}
```
通过  i(vnode, false /* hydrating */ ) 生成子组件的组件vnode.elm;然后在  insert(parentElm, vnode.elm, refElm)将子组件vnode.elm 插入到 parentElm下。

通过这个流程我们就知道vnode -> DOM 的过程是通过深度遍历的方式。将各个级别的元素转成DOM元素，然后由上而下插入生成一个DOM树，并保存在vnode.elm上。

在patch的最后 invokeInsertHook(vnode, insertedVnodeQueue, isInitialPatch) 调用组件的 insert钩子函数并在钩子函数中 <font color=red>callHook(componentInstance, 'mounted')</font>

```js
/**
 * 当组件 从vnode -> 真实的DOM 并且插入到 DOM上的时候  
 * 回调 mounted()  钩子函数
 */
insert(vnode: MountedComponentVNode) {
    const { context, componentInstance } = vnode
    if (!componentInstance._isMounted) {
        componentInstance._isMounted = true
        callHook(componentInstance, 'mounted')
    }
    ...
},
```

#### <font color=red>注意:</font>

##### 1. 由此可见我们注意到一个问题：

> mounted 不会承诺所有的子组件也都一起被挂载。如果你希望等到整个视图都渲染完毕，可以用 vm.$nextTick 替换掉 mounted

即在回调 子组件的 mounted的时候，不能确认子组件已经在浏览器的DOM树上。而需要等到根组件 mounted 的时候才能确认所有的DOM节点都渲染到DOM树上。在 子组件的 mounted的时候 只能确认的是 可以通过 ref 访问的元素节点(注意： 这时候元素可能只是在内存中)。

##### 2. 组件的 mounted 是先子后父，由下而上触发的。


## 分支流程(组件卸载 when vm.$destory() is called)


```html
<body>
    <div id="app">
        <h3>APP</h3>
        <button @click="handleClickShowSecond">showSecond</button>
        <yz-second v-if="isShowSecond"></yz-second>
    </div>
</body>
<script type="text/x-template" id="second">
    <div id="second-wrapper">
      <button @click="handleClickShowThree">showThree</button>
      <yz-three v-if="isShowThree">
        <span>放在组件slot内的</span>
        <yz-children></yz-children>
      </yz-three>
  </div>
</script>
<script type="text/x-template" id="three">
    <div id="three-wrapper">
      this is three
      <slot></slot>
      <yz-four>
          <yz-fourchildren></yz-fourchildren>
      </yz-four>
  </div>
</script>
<script type="text/x-template" id="children">
    <div id="children-wrapper">
      this is children
  </div>
</script>
<script type="text/x-template" id="four">
<div id="children-wrapper">
    this is children
    <slot></slot>
</div>
</script>
<script type="text/javascript" src="./vue.js"></script>
<script type="text/javascript">
var yzFour = Vue.component('yz-four', {
    name: 'yzFour',
    template: '#four',
    beforeDestroy: function() {
        console.log('yzFour beforeDestroy')
    },
    destroyed: function() {
        console.log('yzFour destroyed')
    }
})
var yzFourchildren = Vue.component('yz-fourchildren', {
    name: 'yzFourchildren',
    template: '#children',
    beforeDestroy: function() {
        console.log('yzFourchildren beforeDestroy')
    },
    destroyed: function() {
        console.log('yzFourchildren destroyed')
    }
})
var yzThree = Vue.component('yz-three', {
    name: 'yzThree',
    components: {
        yzFour,
        yzFourchildren
    },
    template: '#three',
    beforeDestroy: function() {
        console.log('yzThree beforeDestroy')
    },
    destroyed: function() {
        console.log('yzThree destroyed')
    }
})
var yzChildren = Vue.component('yz-children', {
    name: 'yzChildren',
    template: '#children',
    beforeDestroy: function() {
        console.log('yzChildren beforeDestroy')
    },
    destroyed: function() {
        console.log('yzChildren destroyed')
    }

})
var yzSecond = Vue.component('yz-second', {
    name: 'yzSecond',
    components: {
        yzThree,
        yzChildren
    },
    data: function() {
        return {
            isShowThree: true
        }
    },
    template: '#second',
    methods: {
        handleClickShowThree() {
            this.isShowThree = !this.isShowThree;
        }
    },
    beforeDestroy: function() {
        console.log('yzSecond beforeDestroy')
    },
    destroyed: function() {
        console.log('yzSecond destroyed')
    }
})

var vue = new Vue({
    el: "#app",
    components: {
        yzSecond
    },
    data: function() {
        return {
            isShowSecond: true
        }
    },
    methods: {
        handleClickShowSecond() {
            this.isShowSecond = !this.isShowSecond;
        }
    },
})
</script>
```
![image](https://note.youdao.com/yws/public/resource/60116e0de35cc4930a98ea2291671610/xmlnote/D7B828F8028A4BA39297216318825D73/11978)

当我们点击 showThree是的yzThree 隐藏。这时候就会触发yzSecond的update

然后触发 yzSecond 的patch

```js
return function patch(oldVnode, vnode, hydrating, removeOnly) {
    if (isUndef(oldVnode)) {

    } else {
        // 第一步 oldVode = #app  所以 oldVnode.nodeType = 1；
        const isRealElement = isDef(oldVnode.nodeType)
        if (!isRealElement && sameVnode(oldVnode, vnode)) {
            // patch existing root node
            patchVnode(oldVnode, vnode, insertedVnodeQueue, removeOnly)
        } else {}
    }
}
```
这时候 oldVnode.nodeType 为undefined 且 div#second-wrapper 相同所以触发 yzSecond的 patchVnode(oldVnode, vnode, insertedVnodeQueue, removeOnly)

在patchVnode 的时候 isDef(oldCh) && isDef(ch) 为true(即存在旧vnode且存在子节点)。所以updateChildren() 处理 <button @click="handleClickShowThree">showThree</button> 和  <yz-three v-if="isShowThree"> 两个子节点

在updateChildren() 的时候通过diff 判断

```js
//  oldStartIdx === 1  newStartIdx === 1 oldStartVnode === button newStartVnode === button
if (sameVnode(oldStartVnode, newStartVnode)) {    
    patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue)
    oldStartVnode = oldCh[++oldStartIdx]
    newStartVnode = newCh[++newStartIdx]
}
```
然后比较新旧两个button，然后又updateChildren()处理button的文本节点 showThree 相同

这一步具体请看patch的过程

###### updateChildren() 处理好button，然后  oldStartIdx === 2  newStartIdx === 2

这时候在 while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx)  的时候所有的条件都触发不上 导致 newStartIdx = 3;

然后触发 removeVnodes 去除子节点 （yz-three 节点）
```js
if (newStartIdx > newEndIdx) {
    removeVnodes(parentElm, oldCh, oldStartIdx, oldEndIdx)
}

function removeVnodes(parentElm, vnodes, startIdx, endIdx) {
    for (; startIdx <= endIdx; ++startIdx) {
        const ch = vnodes[startIdx]
        if (isDef(ch)) {
            //  如果 存在 tag元素 说明这是不是一个文本vnode
            if (isDef(ch.tag)) {
                // 移除vnode上定义的 事件、cbs
                removeAndInvokeRemoveHook(ch)
                // 调用组件vnode 的 data.hook.destory 钩子函数进行组件的卸载
                invokeDestroyHook(ch)
            } else { // Text node
                removeNode(ch.elm)
            }
        }
    }
}
```
我们看 removeVnodes() 的逻辑，发现其先移除占位符vnode上的事件、cbs等；然后调用 invokeDestroyHook(ch)

```js
/**
 * 调用组件卸载的钩子函数  data.hook.destroy
 * @param {*} vnode
 */
function invokeDestroyHook(vnode) {
    let i, j
    const data = vnode.data

    if (isDef(data)) {
        // 调用占位符vnode的 data.hook的destory钩子函数
        if (isDef(i = data.hook) && isDef(i = i.destroy)) i(vnode)
        // 调用cbs.destory 的属性处理函数 如： destroy(),unbindDirectives()
        for (i = 0; i < cbs.destroy.length; ++i) cbs.destroy[i](vnode)
    }
    // 深度遍历处理占位符vnode中的 子组件的卸载
    if (isDef(i = vnode.children)) {
        for (j = 0; j < vnode.children.length; ++j) {
            invokeDestroyHook(vnode.children[j])
        }
    }
}

//-------------------------
/**
 * 组件卸载时调用的钩子函数
 * @param {MountedComponentVNode} vnode
 */
destroy(vnode: MountedComponentVNode) {
    const { componentInstance } = vnode
    if (!componentInstance._isDestroyed) {
        if (!vnode.data.keepAlive) {
            // 调用组件实例方法 $destory
            componentInstance.$destroy()
        } else {
            deactivateChildComponent(componentInstance, true /* direct */ )
        }
    }
}
```

在invokeDestroyHook中通过调用data.hook.destory钩子函数，触发子组件的 $destroy()方法。然后再深度遍历vnode.children

那么我们就可以想象对于组件的卸载的流程是这样的：(由下至上依次触发)

当触发某一组件的更新的时候其先patch处理当前组件，然后通过diff去比较子组件，如果遇到节点的卸载，那么就调用removeVnode方法去移除此节点。

如果这个节点是一个组件的占位符节点，那么其先调用组件的data.hook.destory钩子函数 -> 调用组件的$destory实例方法 -> 然后触发子组件的vm.__patch__(vm._vnode, null) -> 然后深度遍历invokeDestroyHook()卸载子组件的所有节点 ....

###### 所以上面栗子中当点击showThree去隐藏yz-three节点的时候其输出的结果为

```js
yzThree beforeDestroy
yzChildren beforeDestroy
yzChildren destroyed
yzFour beforeDestroy
yzFourchildren beforeDestroy
yzFourchildren destroyed
yzFour destroyed
yzThree destroyed
```
