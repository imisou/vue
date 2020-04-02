# 指令处理流程


#### 一个指令定义对象可以提供如下几个钩子函数 (均为可选)：

###### 1. bind：只调用一次，指令第一次绑定到元素时调用。在这里可以进行一次性的初始化设置。

###### 2. inserted：被绑定元素插入父节点时调用 (仅保证父节点存在，但不一定已被插入文档中)。

###### 3. update：所在组件的 VNode 更新时调用，但是可能发生在其子 VNode 更新之前。指令的值可能发生了改变，也可能没有。但是你可以通过比较更新前后的值来忽略不必要的模板更新 (详细的钩子函数参数见下)。

###### 4. componentUpdated：指令所在组件的 VNode 及其子 VNode 全部更新后调用。

###### 5. unbind：只调用一次，指令与元素解绑时调用。



## Vue对象初始化流程

###### core\instance\index.js

```js
function Vue (options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  this._init(options)
}

```
创建一个Vue的构造函数，然后

###### core\index.js

```
initGlobalAPI(Vue)
```

###### platforms\web\runtime\index.js
```js
// install platform runtime directives & components
extend(Vue.options.directives, platformDirectives)
```
然后在 各平台下拓展了配置options.directives对象，使得Vue.options.directives变成了一个

![image](https://note.youdao.com/yws/public/resource/fa4a717e0bafc76404a2b7658a9371c6/xmlnote/7396A03B081E40F49799FB638869D40B/8660)

## Vue 创建实例对象过程

```js
vm.$options = mergeOptions(
    resolveConstructorOptions(vm.constructor),
    options || {},
    vm
)
```
合并Vue构造函数上的Vue.options对象 从而使得用户定义的 directives、 Vue.mixin合并的、Vue内置的directives合并在vm.$options.directives 中
```js
vm.$options = {
    directives:{
        __proto__ : {
            show : {},
            model : {}
        }
    }
}
```

## patch 过程

在patch 将vnode生成DOM的过程中，如果发现vnode 为节点类型，其会调用createElm()方法去创建节点，此时判断如果不是组件占位符节点类型(就是普通的元素节点类型)其会调用元素创建时候的钩子函数

```js
if (isDef(data)) {
    invokeCreateHooks(vnode, insertedVnodeQueue)
}

---------------
if (isDef(i.create)) i.create(emptyNode, vnode)

```
发现i.create中存在 一个 updateDirectives()钩子函数
![image](https://note.youdao.com/yws/public/resource/fa4a717e0bafc76404a2b7658a9371c6/xmlnote/3523BCC2C4244ED3A7223D372D700F06/8531)

其不在 platforms\web\runtime\modules\index.js 定义的各平台的元素各生命周期的钩子函数，而是Vue的baseModules。

###### core\vdom\modules\directives.js

```js
export default {
    create: updateDirectives,
    update: updateDirectives,
    destroy: function unbindDirectives(vnode: VNodeWithData) {
        updateDirectives(vnode, emptyNode)
    }
}

function updateDirectives(oldVnode: VNodeWithData, vnode: VNodeWithData) {
    if (oldVnode.data.directives || vnode.data.directives) {
        _update(oldVnode, vnode)
    }
}
```

说明在vnode的create update destroy3个生命周期中都会触发updateDirectives()方法。
我们看updateDirectives发现如果新旧vnode中发现存在data.directives属性就进行_update() 方法。

下面是_update() 的方法

其实其步骤是这样的

1. 通过normalizeDirectives(Vnode.data.directives, vm) 去以属性中的指令为基础合并vm.$options.directives中的同名的指令执行对象，最后形成一个新的对象

```js

/**
 * 指令属性 处理方法

    这里面涉及到指令的 bind  inserted update componentUpdated unbind 5个生命周期

    涉及到钩子函数的回调函数的参数
    vnode.elm,
    dir : {
        name : '' , // 指令的名称
        value : ,   // 指令的值
        oldValue :    , // 指令绑定的前一个值，仅在 update 和 componentUpdated 钩子中可用。无论值是否改变都可用
        expression  :     //
        arg
        modifiers   :
    },
    vnode,
    oldVnode,
    isDestroy       


 * @param {*} oldVnode
 * @param {*} vnode
 */
function _update(oldVnode, vnode) {
    const isCreate = oldVnode === emptyNode
    const isDestroy = vnode === emptyNode
    const oldDirs = normalizeDirectives(oldVnode.data.directives, oldVnode.context)
    // 生成当前vnode的指令处理对象
    const newDirs = normalizeDirectives(vnode.data.directives, vnode.context)

}

```

```js
vnode.directives 中的v-show : {
    name : '' ,       // 指令的名称
    value : ,       // 指令的值
    expression  :     //
    arg         :  ''     //入参
    modifiers   :  
},

vm.$options.directives 中的
show: {
    bind : () => {},
    inserted : () => {}...
}

最后合并成
dir = {
    'show.xx.xxx' : {
        name : '' ,       // 指令的名称
        value : ,         // 指令的值
        oldValue : ''
        expression  :     //
        arg         :  ''     //入参
        modifiers   :  
        def : {
            bind : () => {},
            inserted : () => {}...
        }
    }
}
```

2. 变量新的vnode对象中的所有dir属性，发现
    1. 如果旧vnode中不存在 那么就是创建，执行bind生命周期的回调钩子函数。如果还定义了inserted生命周期的回调钩子函数，那么就通过dirsWithInsert() 在 vnode.data.hook.insert的生命周期钩子函数中创建一个invoker回调函数。
    2. 如果旧vnode存在此指令属性，那么就是更新 触发update生命周期的回调钩子函数,同样如果定义了componentUpdated生命周期的回调钩子函数，那么就通过dirsWithPostpatch() 在vnode.data.hook.postpatch的生命周期钩子函数中创建一个invoker回调函数。

```js
let key, oldDir, dir
for (key in newDirs) {
    oldDir = oldDirs[key]
    dir = newDirs[key]
    // 如果旧vnode与新vnode相同的指令属性 不存在，那么说明这是一个新的指令
    if (!oldDir) {
        // new directive, bind
        // 回调指令的 bind 方法
        callHook(dir, 'bind', vnode, oldVnode)
        // 如果定义了inserted 时的回调函数 那么将其存入回调队列
        if (dir.def && dir.def.inserted) {
            dirsWithInsert.push(dir)
        }
    } else {
        // existing directive, update
        // 如果存在相同的回调周期，那么说明需要更新
        dir.oldValue = oldDir.value
        // 回调指令的 update 方法，并在之前修改了保存了oldValue
        callHook(dir, 'update', vnode, oldVnode)
        // 如果定义了componentUpdated 时的回调函数 那么将其存入回调队列
        if (dir.def && dir.def.componentUpdated) {
            dirsWithPostpatch.push(dir)
        }
    }
}
// 如果是一个新的指令属性
if (dirsWithInsert.length) {
    // 创建所有新的指令属性的 inserted周期的回调函数方法
    const callInsert = () => {
        for (let i = 0; i < dirsWithInsert.length; i++) {
            callHook(dirsWithInsert[i], 'inserted', vnode, oldVnode)
        }
    }
    // 说明vnode是 刚创建
    if (isCreate) {
        mergeVNodeHook(vnode, 'insert', callInsert)
    } else {
        // 说明是vnode 更新期间添加的新的指令属性
        callInsert()
    }
}
// 如果更新vnode期间 原来也存在此指令属性 那么就触发update方法，
//  然后在vnode转dom完成触发vnode.data.hook.postpatch钩子函数 是回调此指令属性的 componentUpdated生命周期钩子函数
if (dirsWithPostpatch.length) {
    // 跟bind的时候一样 在vnode.data.hook中添加一个钩子函数
    mergeVNodeHook(vnode, 'postpatch', () => {
        for (let i = 0; i < dirsWithPostpatch.length; i++) {
            callHook(dirsWithPostpatch[i], 'componentUpdated', vnode, oldVnode)
        }
    })
}
```


3. 如果旧vnode存在，然后遍历旧vnode的dir，如果发现新vnode中不存在此指令属性，那么说明就是销毁，那么就触发unbind生命周期的回调钩子函数

```js
// 如果存在旧的vnode 说明这是更新阶段
if (!isCreate) {
    // 遍历旧的指令属性
    for (key in oldDirs) {
        // 如果在新的vnode中不存在说明 需要调用解绑的钩子函数
        if (!newDirs[key]) {
            // no longer present, unbind
            callHook(oldDirs[key], 'unbind', oldVnode, oldVnode, isDestroy)
        }
    }
}
```

### <font color=red>重点：</font>

##### 1. bind 与 inserted生命周期的区别？如何实现的？同理还有 update 与 componentUpdated

回看上面指令定义的钩子函数说明中

- bind：只调用一次，指令第一次绑定到元素时调用。在这里可以进行一次性的初始化设置。
- inserted：被绑定元素插入父节点时调用 (仅保证父节点存在，但不一定已被插入文档中)。

第一：

他们在元素存在DOM中都只执行一次 不会因为vnode的update而从新触发。

第二：

- bind 的时候 vnode.el还是 undefined 以为这时候vnode还没有转换成dom。
- inserted触发的时候vnode.el是有节点的元素的，此时vnode也转换成dom，但是不一定挂载到真实DOM树中 所以此时 document.getElementById(xxx)是不一定找到此节点元素的。

###### 如何实现的 ？

上面inserted 和 componentUpdated 都是通过 mergeVNodeHook() 去实现的。

我们看   
###### core\vdom\helpers\merge-hook.js
```js
/**

  vnode.data.hook[声明周期(insert)] = invoker(构造函数)
  invoker = function(){}
  invoker.fns 保存了所有的 回调处理方法
  inveker.merged 表明合并成功

 * @param {*} def          如果def = vnode 那么  def = vnode.data.hook
 * @param {*} hookKey      vnode 生命周期名称  insert postpatch
 * @param {*} hook         回调函数
 */
export function mergeVNodeHook(def: Object, hookKey: string, hook: Function) {
    if (def instanceof VNode) {
        def = def.data.hook || (def.data.hook = {})
    }
    let invoker
    const oldHook = def[hookKey]

    function wrappedHook() {
        hook.apply(this, arguments)
        // important: remove merged hook to ensure it's called only once
        // and prevent memory leak
        // 钩子函数执行完成 后就删除当前钩子函数的回调方法，防止重复触发
        remove(invoker.fns, wrappedHook)
    }

    // 如果没有定义当前生命周期的钩子函数 hook
    if (isUndef(oldHook)) {
        // no existing hook
        // 那么就创建一个新的 回调函数
        invoker = createFnInvoker([wrappedHook])
    } else {
        /* istanbul ignore if */
        // 如果定义过 invoker 那么就直接往 invoker.fns 队列中添加此回调 方法
        if (isDef(oldHook.fns) && isTrue(oldHook.merged)) {
            // already a merged invoker
            invoker = oldHook
            invoker.fns.push(wrappedHook)
        } else {
            // existing plain hook
            // 没有fns 说明还是咩有创建钩子函数 那么还是创建新的 invoker回调函数
            invoker = createFnInvoker([oldHook, wrappedHook])
        }
    }
    // 设置回调函数 合并成功
    invoker.merged = true
    // 放在钩子函数中
    def[hookKey] = invoker
}
```
其主要还是使用了vnode.data.hook这是属性对象，并在其中存储了vnode各个生命周期的钩子函数，如此时的 insert 和 postpatch两个周期
然后我们看我们执行directives的时候，即patch的过程
```js
if (isDef(i.create)) i.create(emptyNode, vnode)
if (isDef(i.insert)) insertedVnodeQueue.push(vnode)
```
是否发现我们在将回调的invoker回调者存放在 vnode.data.hook.insert的时候，在下一步就 if (isDef(i.insert)) 然后将此vnode push到插入队列。

然后我们在patch的最后 执行了invokeInsertHook()方法。发现在最后 queue[i].data.hook.insert(queue[i]) 回调队列中vnode的insert钩子函数。在执行了invokeInsertHook()之前vnode 已经渲染成DOM数，但是还没有插入到浏览器中
```js
invokeInsertHook(vnode, insertedVnodeQueue, isInitialPatch)

function invokeInsertHook(vnode, queue, initial) {
    // delay insert hooks for component root nodes, invoke them after the
    // element is really inserted
    if (isTrue(initial) && isDef(vnode.parent)) {
        vnode.parent.data.pendingInsert = queue
    } else {
        for (let i = 0; i < queue.length; ++i) {
            queue[i].data.hook.insert(queue[i])
        }
    }
}
```
而是在

```js
Vue.prototype._update = function(vnode: VNode, hydrating ? : boolean) {

    if (!prevVnode) {
        // initial render
        vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */ )
    } else {
        // 当组件更新的时候触发 preVnode 为旧的组件VNode vnode为新render生成的VNode
        // updates
        vm.$el = vm.__patch__(prevVnode, vnode)
    }

    // if parent is an HOC, update its $el as well  插入到父组件的DOM数
    if (vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode) {
        vm.$parent.$el = vm.$el
    }
}
```

##### 2. isDestroy 这个参数的作用？

我们可以想象什么时候应该触发指令的 unbind钩子函数？

1. 旧vnode中存在此指令属性而新vnode中不存在此指令属性，那么此指令属性就需要调用unbind钩子函数。
2. 旧vnode存在 而新vnode已经不存在了，就是节点移除了，那么也要触发unbind钩子函数。

isDestroy就是用来区别这两个触发时机的。看 const isDestroy = vnode === emptyNode

```js
function _update(oldVnode, vnode) {
    const isCreate = oldVnode === emptyNode
    const isDestroy = vnode === emptyNode
}
```

##### 3. 各生命周期的钩子函数的参数？

```js
/**
 * 指令生命周期的回调函数方法
 *
 * @param {*} dir   指令的属性即处理对象
 * @param {*} hook  当前回调的生命周期字符串  'bind' 'insert'
 * @param {*} vnode  新的vnode
 * @param {*} oldVnode  原来的vnode
 * @param {*} isDestroy  vnode是否是销毁，还是只是此指令属性销毁了 。 true: vnode销毁 false：指令属性销毁
 */
function callHook(dir, hook, vnode, oldVnode, isDestroy) {
    const fn = dir.def && dir.def[hook]
    if (fn) {
        try {
            fn(vnode.elm, dir, vnode, oldVnode, isDestroy)
        } catch (e) {
            handleError(e, vnode.context, `directive ${dir.name} ${hook} hook`)
        }
    }
}
```

可以发现我们各个生命周期钩子函数都有5个参数

```html
 <div v-clickoutside:arg1.mod1.mod2="value">xxxxx</div>
```

```js
vnode.elm ：
    bind的时候 vnode还没有转换为 dom所以 vnode.elm 是 undefined
    inserted , update ,componentUpdated 期间都是存在的
    unbind : vnode.elm指向的是 oldVnode.elm

dir : {
    name : 'show' ,                    // 指令的名称

    value : "当前value的新值  如 1",   // 指令的值

    oldValue :  "当前value以前的值"  , // 指令绑定的前一个值，仅在 update 和 componentUpdated 钩子中可用。无论值是否改变都可用

    expression  :  "value"            // 指令属性值的表达式

    arg  : 'arg1'  ,                  // 指令传入的入参

    modifiers  : {                    // 指令属性 的修饰符
        mod1 : true,
        mod2 : true
    }

    dep : {                           // 指令执行对象
        bind : () => {}
        inserted : () => {}
    }
}

vnode :                               // 新的vnode

oldVnode :                            // 旧的vnode

isDestroy :                           // 重点2说明的问题 表明解绑的时机
```
