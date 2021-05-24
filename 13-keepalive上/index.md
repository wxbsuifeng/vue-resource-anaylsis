13.1 基本用法
keep-alive的使用只需要在动态组件的最外层添加标签即可。

<div id="app">
    <button @click="changeTabs('child1')">child1</button>
    <button @click="changeTabs('child2')">child2</button>
    <keep-alive>
        <component :is="chooseTabs">
        </component>
    </keep-alive>
</div>
Copy

var child1 = {
    template: '<div><button @click="add">add</button><p>{{num}}</p></div>',
    data() {
        return {
            num: 1
        }
    },
    methods: {
        add() {
            this.num++
        }
    },
}
var child2 = {
    template: '<div>child2</div>'
}
var vm = new Vue({
    el: '#app',
    components: {
        child1,
        child2,
    },
    data() {
        return {
            chooseTabs: 'child1',
        }
    },
    methods: {
        changeTabs(tab) {
            this.chooseTabs = tab;
        }
    }
})
Copy
简单的结果如下，动态组件在child1,child2之间来回切换，当第二次切到child1时，child1保留着原来的数据状态，num = 5。



13.2 从模板编译到生成vnode
按照以往分析的经验，我们会从模板的解析开始说起，第一个疑问便是：内置组件和普通组件在编译过程有区别吗？答案是没有的，不管是内置的还是用户定义组件，本质上组件在模板编译成render函数的处理方式是一致的，这里的细节不展开分析，有疑惑的可以参考前几节的原理分析。最终针对keep-alive的render函数的结果如下：

with(this){···_c('keep-alive',{attrs:{"include":"child2"}},[_c(chooseTabs,{tag:"component"})],1)}
Copy
有了render函数，接下来从子开始到父会执行生成Vnode对象的过程，_c('keep-alive'···)的处理，会执行createElement生成组件Vnode,其中由于keep-alive是组件，所以会调用createComponent函数去创建子组件Vnode,createComponent之前也有分析过，这个环节和创建普通组件Vnode不同之处在于，keep-alive的Vnode会剔除多余的属性内容，由于keep-alive除了slot属性之外，其他属性在组件内部并没有意义，例如class样式，<keep-alive clas="test"></keep-alive>等，所以在Vnode层剔除掉多余的属性是有意义的。而<keep-alive slot="test">的写法在2.6以上的版本也已经被废弃。(其中abstract作为抽象组件的标志，以及其作用我们后面会讲到)

// 创建子组件Vnode过程
function createComponent(Ctordata,context,children,tag) {
    // abstract是内置组件(抽象组件)的标志
    if (isTrue(Ctor.options.abstract)) {
        // 只保留slot属性，其他标签属性都被移除，在vnode对象上不再存在
        var slot = data.slot;
        data = {};
        if (slot) {
            data.slot = slot;
        }
    }
}
Copy
13.3 初次渲染
keep-alive之所以特别，是因为它不会重复渲染相同的组件，只会利用初次渲染保留的缓存去更新节点。所以为了全面了解它的实现原理，我们需要从keep-alive的首次渲染开始说起。

13.3.1 流程图
为了理清楚流程，我大致画了一个流程图，流程图大致覆盖了初始渲染keep-alive所执行的过程，接下来会照着这个过程进行源码分析。



和渲染普通组件相同的是，Vue会拿到前面生成的Vnode对象执行真实节点创建的过程，也就是熟悉的patch过程,patch执行阶段会调用createElm创建真实dom，在创建节点途中，keep-alive的vnode对象会被认定是一个组件Vnode,因此针对组件Vnode又会执行createComponent函数，它会对keep-alive组件进行初始化和实例化。

function createComponent (vnode, insertedVnodeQueue, parentElm, refElm) {
      var i = vnode.data;
      if (isDef(i)) {
        // isReactivated用来判断组件是否缓存。
        var isReactivated = isDef(vnode.componentInstance) && i.keepAlive;
        if (isDef(i = i.hook) && isDef(i = i.init)) {
            // 执行组件初始化的内部钩子 init
          i(vnode, false /* hydrating */);
        }
        if (isDef(vnode.componentInstance)) {
          // 其中一个作用是保留真实dom到vnode中
          initComponent(vnode, insertedVnodeQueue);
          insert(parentElm, vnode.elm, refElm);
          if (isTrue(isReactivated)) {
            reactivateComponent(vnode, insertedVnodeQueue, parentElm, refElm);
          }
          return true
        }
      }
    }
Copy
keep-alive组件会先调用内部钩子init方法进行初始化操作，我们先看看init过程做了什么操作。

// 组件内部钩子
var componentVNodeHooks = {
    init: function init (vnode, hydrating) {
      if (
        vnode.componentInstance &&
        !vnode.componentInstance._isDestroyed &&
        vnode.data.keepAlive
      ) {
        // kept-alive components, treat as a patch
        var mountedNode = vnode; // work around flow
        componentVNodeHooks.prepatch(mountedNode, mountedNode);
      } else {
          // 将组件实例赋值给vnode的componentInstance属性
        var child = vnode.componentInstance = createComponentInstanceForVnode(
          vnode,
          activeInstance
        );
        child.$mount(hydrating ? vnode.elm : undefined, hydrating);
      }
    },
    // 后面分析
    prepatch： function() {}
}
Copy
第一次执行，很明显组件vnode没有componentInstance属性，vnode.data.keepAlive也没有值，所以会调用createComponentInstanceForVnode方法进行组件实例化并将组件实例赋值给vnode的componentInstance属性， 最终执行组件实例的$mount方法进行实例挂载。

createComponentInstanceForVnode就是组件实例化的过程，而组件实例化从系列的第一篇就开始说了，无非就是一系列选项合并，初始化事件，生命周期等初始化操作。

function createComponentInstanceForVnode (vnode, parent) {
    var options = {
      _isComponent: true,
      _parentVnode: vnode,
      parent: parent
    };
    // 内联模板的处理，忽略这部分代码
    ···
    // 执行vue子组件实例化
    return new vnode.componentOptions.Ctor(options)
  }
Copy
13.3.2 内置组件选项
我们在使用组件的时候经常利用对象的形式定义组件选项，包括data,method,computed等，并在父组件或根组件中注册。keep-alive同样遵循这个道理，内置两字也说明了keep-alive是在Vue源码中内置好的选项配置，并且也已经注册到全局，这一部分的源码可以参考组态组件小节末尾对内置组件构造器和注册过程的介绍。这一部分我们重点关注一下keep-alive的具体选项。

// keepalive组件选项
  var KeepAlive = {
    name: 'keep-alive',
    // 抽象组件的标志
    abstract: true,
    // keep-alive允许使用的props
    props: {
      include: patternTypes,
      exclude: patternTypes,
      max: [String, Number]
    },

    created: function created () {
      // 缓存组件vnode
      this.cache = Object.create(null);
      // 缓存组件名
      this.keys = [];
    },

    destroyed: function destroyed () {
      for (var key in this.cache) {
        pruneCacheEntry(this.cache, key, this.keys);
      }
    },

    mounted: function mounted () {
      var this$1 = this;
      // 动态include和exclude
      // 对include exclue的监听
      this.$watch('include', function (val) {
        pruneCache(this$1, function (name) { return matches(val, name); });
      });
      this.$watch('exclude', function (val) {
        pruneCache(this$1, function (name) { return !matches(val, name); });
      });
    },
    // keep-alive的渲染函数
    render: function render () {
      // 拿到keep-alive下插槽的值
      var slot = this.$slots.default;
      // 第一个vnode节点
      var vnode = getFirstComponentChild(slot);
      // 拿到第一个组件实例
      var componentOptions = vnode && vnode.componentOptions;
      // keep-alive的第一个子组件实例存在
      if (componentOptions) {
        // check pattern
        //拿到第一个vnode节点的name
        var name = getComponentName(componentOptions);
        var ref = this;
        var include = ref.include;
        var exclude = ref.exclude;
        // 通过判断子组件是否满足缓存匹配
        if (
          // not included
          (include && (!name || !matches(include, name))) ||
          // excluded
          (exclude && name && matches(exclude, name))
        ) {
          return vnode
        }

        var ref$1 = this;
        var cache = ref$1.cache;
        var keys = ref$1.keys;
        var key = vnode.key == null
          ? componentOptions.Ctor.cid + (componentOptions.tag ? ("::" + (componentOptions.tag)) : '')
          : vnode.key;
          // 再次命中缓存
        if (cache[key]) {
          vnode.componentInstance = cache[key].componentInstance;
          // make current key freshest
          remove(keys, key);
          keys.push(key);
        } else {
        // 初次渲染时，将vnode缓存
          cache[key] = vnode;
          keys.push(key);
          // prune oldest entry
          if (this.max && keys.length > parseInt(this.max)) {
            pruneCacheEntry(cache, keys[0], keys, this._vnode);
          }
        }
        // 为缓存组件打上标志
        vnode.data.keepAlive = true;
      }
      // 将渲染的vnode返回
      return vnode || (slot && slot[0])
    }
  };
Copy
keep-alive选项跟我们平时写的组件选项还是基本类似的，唯一的不同是keep-ailve组件没有用template而是使用render函数。keep-alive本质上只是存缓存和拿缓存的过程，并没有实际的节点渲染，所以使用render处理是最优的选择。

13.3.3 缓存vnode
还是先回到流程图的分析。上面说到keep-alive在执行组件实例化之后会进行组件的挂载。而挂载$mount又回到vm._render(),vm._update()的过程。由于keep-alive拥有render函数，所以我们可以直接将焦点放在render函数的实现上。

首先是获取keep-alive下插槽的内容，也就是keep-alive需要渲染的子组件,例子中是chil1 Vnode对象，源码中对应getFirstComponentChild函数。
  function getFirstComponentChild (children) {
    if (Array.isArray(children)) {
      for (var i = 0; i < children.length; i++) {
        var c = children[i];
        // 组件实例存在，则返回，理论上返回第一个组件vnode
        if (isDef(c) && (isDef(c.componentOptions) || isAsyncPlaceholder(c))) {
          return c
        }
      }
    }
  }
Copy
判断组件满足缓存的匹配条件，在keep-alive组件的使用过程中，Vue源码允许我们是用include, exclude来定义匹配条件，include规定了只有名称匹配的组件才会被缓存，exclude规定了任何名称匹配的组件都不会被缓存。更者，我们可以使用max来限制可以缓存多少匹配实例，而为什么要做数量的限制呢？我们后文会提到。
拿到子组件的实例后，我们需要先进行是否满足匹配条件的判断,其中匹配的规则允许使用数组，字符串，正则的形式。

var include = ref.include;
var exclude = ref.exclude;
// 通过判断子组件是否满足缓存匹配
if (
    // not included
    (include && (!name || !matches(include, name))) ||
    // excluded
    (exclude && name && matches(exclude, name))
) {
    return vnode
}

// matches
function matches (pattern, name) {
    // 允许使用数组['child1', 'child2']
    if (Array.isArray(pattern)) {
        return pattern.indexOf(name) > -1
    } else if (typeof pattern === 'string') {
        // 允许使用字符串 child1,child2
        return pattern.split(',').indexOf(name) > -1
    } else if (isRegExp(pattern)) {
        // 允许使用正则 /^child{1,2}$/g
        return pattern.test(name)
    }
    /* istanbul ignore next */
    return false
}
Copy
如果组件不满足缓存的要求，则直接返回组件的vnode,不做任何处理,此时组件会进入正常的挂载环节。

render函数执行的关键一步是缓存vnode,由于是第一次执行render函数，选项中的cache和keys数据都没有值，其中cache是一个空对象，我们将用它来缓存{ name: vnode }枚举，而keys我们用来缓存组件名。 因此我们在第一次渲染keep-alive时，会将需要渲染的子组件vnode进行缓存。

 cache[key] = vnode;
 keys.push(key);
Copy
将已经缓存的vnode打上标记, 并将子组件的Vnode返回。 vnode.data.keepAlive = true

13.3.4 真实节点的保存
我们再回到createComponent的逻辑，之前提到createComponent会先执行keep-alive组件的初始化流程，也包括了子组件的挂载。并且我们通过componentInstance拿到了keep-alive组件的实例，而接下来重要的一步是将真实的dom保存再vnode中。

function createComponent(vnode, insertedVnodeQueue) {
    ···
    if (isDef(vnode.componentInstance)) {
        // 其中一个作用是保留真实dom到vnode中
        initComponent(vnode, insertedVnodeQueue);
        // 将真实节点添加到父节点中
        insert(parentElm, vnode.elm, refElm);
        if (isTrue(isReactivated)) {
            reactivateComponent(vnode, insertedVnodeQueue, parentElm, refElm);
        }
        return true
    }
}
Copy
insert的源码不列举出来，它只是简单的调用操作dom的api,将子节点插入到父节点中，我们可以重点看看initComponent关键步骤的逻辑。

function initComponent() {
    ···
    // vnode保留真实节点
    vnode.elm = vnode.componentInstance.$el;
    ···
}
Copy
因此，我们很清晰的回到之前遗留下来的问题，为什么keep-alive需要一个max来限制缓存组件的数量。原因就是keep-alive缓存的组件数据除了包括vnode这一描述对象外，还保留着真实的dom节点,而我们知道真实节点对象是庞大的，所以大量保留缓存组件是耗费性能的。因此我们需要严格控制缓存的组件数量，而在缓存策略上也需要做优化，这点我们在下一篇文章也继续提到。

由于isReactivated为false,reactivateComponent函数也不会执行。至此keep-alive的初次渲染流程分析完毕。

如果忽略步骤的分析，只对初次渲染流程做一个总结：内置的keep-alive组件，让子组件在第一次渲染的时候将vnode和真实的elm进行了缓存。

13.4 抽象组件
这一节的最后顺便提一下上文提到的抽象组件的概念。Vue提供的内置组件都有一个描述组件类型的选项，这个选项就是{ astract: true },它表明了该组件是抽象组件。什么是抽象组件，为什么要有这一类型的区别呢？我觉得归根究底有两个方面的原因。

抽象组件没有真实的节点，它在组件渲染阶段不会去解析渲染成真实的dom节点，而只是作为中间的数据过渡层处理，在keep-alive中是对组件缓存的处理。
在我们介绍组件初始化的时候曾经说到父子组件会显式的建立一层关系，这层关系奠定了父子组件之间通信的基础。我们可以再次回顾一下initLifecycle的代码。
Vue.prototype._init = function() {
    ···
    var vm = this;
    initLifecycle(vm)
}

function initLifecycle (vm) {
    var options = vm.$options;

    var parent = options.parent;
    if (parent && !options.abstract) {
        // 如果有abstract属性，一直往上层寻找，直到不是抽象组件
      while (parent.$options.abstract && parent.$parent) {
        parent = parent.$parent;
      }
      parent.$children.push(vm);
    }
    ···
  }
Copy
子组件在注册阶段会把父实例挂载到自身选项的parent属性上，在initLifecycle过程中，会反向拿到parent上的父组件vnode,并为其$children属性添加该子组件vnode,如果在反向找父组件的过程中，父组件拥有abstract属性，即可判定该组件为抽象组件，此时利用parent的链条往上寻找，直到组件不是抽象组件为止。initLifecycle的处理，让每个组件都能找到上层的父组件以及下层的子组件，使得组件之间形成一个紧密的关系树。

13.5 小结
这一节介绍了Vue内置组件中一个最重要的，也是最常用的组件keep-alive，在日常开发中，我们经常将keep-alive配合动态组件is使用，达到切换组件的同时，将旧的组件缓存。最终达到保留初始状态的目的。在第一次组件渲染时，keep-alive会将组件Vnode以及对应的真实节点进行缓存。而当再次渲染组件时，keep-alive是如何利用这些缓存的？源码又对缓存进行了优化？并且keep-alive组件的生命周期又包括哪些，这些疑问我们将在下一节一一展开。