7.1 数据初始化
回顾一下之前的内容，我们对Vue源码的分析是从初始化开始，初始化_init会执行一系列的过程，这个过程包括了配置选项的合并，数据的监测代理，最后才是实例的挂载。而在实例挂载前还有意忽略了一个重要的过程，数据的初始化(即initState(vm))。initState的过程，是对数据进行响应式设计的过程，过程会针对props,methods,data,computed和watch做数据的初始化处理，并将他们转换为响应式对象，接下来我们会逐步分析每一个过程。

function initState (vm) {
  vm._watchers = [];
  var opts = vm.$options;
  // 初始化props
  if (opts.props) { initProps(vm, opts.props); }
  // 初始化methods
  if (opts.methods) { initMethods(vm, opts.methods); }
  // 初始化data
  if (opts.data) {
    initData(vm);
  } else {
    // 如果没有定义data，则创建一个空对象，并设置为响应式
    observe(vm._data = {}, true /* asRootData */);
  }
  // 初始化computed
  if (opts.computed) { initComputed(vm, opts.computed); }
  // 初始化watch
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch);
  }
}
Copy
7.2 initProps
简单回顾一下props的用法，父组件通过属性的形式将数据传递给子组件，子组件通过props属性接收父组件传递的值。

// 父组件
<child :test="test"></child>
var vm = new Vue({
  el: '#app',
  data() {
    return {
      test: 'child'
    }
  }
})
// 子组件
Vue.component('child', {
  template: '<div>{{test}}</div>',
  props: ['test']
})
Copy
因此分析props需要分析父组件和子组件的两个过程，我们先看父组件对传递值的处理。按照以往文章介绍的那样，父组件优先进行模板编译得到一个render函数，在解析过程中遇到子组件的属性，:test=test会被解析成{ attrs: {test： test}}并作为子组件的render函数存在，如下所示:

with(){..._c('child',{attrs:{"test":test}})}
Copy
render解析Vnode的过程遇到child这个子占位符节点，因此会进入创建子组件Vnode的过程，创建子Vnode过程是调用createComponent,这个阶段我们在组件章节有分析过，在组件的高级用法也有分析过，最终会调用new Vnode去创建子Vnode。而对于props的处理，extractPropsFromVNodeData会对attrs属性进行规范校验后，最后会把校验后的结果以propsData属性的形式传入Vnode构造器中。总结来说，props传递给占位符组件的写法，会以propsData的形式作为子组件Vnode的属性存在。下面会分析具体的细节。

// 创建子组件过程
function createComponent() {
  // props校验
  var propsData = extractPropsFromVNodeData(data, Ctor, tag);
  ···
  // 创建子组件vnode
  var vnode = new VNode(
    ("vue-component-" + (Ctor.cid) + (name ? ("-" + name) : '')),
    data, undefined, undefined, undefined, context,
    { Ctor: Ctor, propsData: propsData, listeners: listeners, tag: tag, children: children },
    asyncFactory
  );
}
Copy
7.2.1 props的命名规范
先看检测props规范性的过程。props编译后的结果有两种，其中attrs前面分析过，是编译生成render函数针对属性的处理，而props是针对用户自写render函数的属性值。因此需要同时对这两种方式进行校验。

function extractPropsFromVNodeData (data,Ctor,tag) {
  // Ctor为子类构造器
  ···
  var res = {};
  // 子组件props选项
  var propOptions = Ctor.options.props;
  // data.attrs针对编译生成的render函数，data.props针对用户自定义的render函数
  var attrs = data.attrs;
  var props = data.props;
  if (isDef(attrs) || isDef(props)) {
    for (var key in propOptions) {
      // aB 形式转成 a-b
      var altKey = hyphenate(key);
      {
          var keyInLowerCase = key.toLowerCase();
          if (
            key !== keyInLowerCase &&
            attrs && hasOwn(attrs, keyInLowerCase)
          ) {
            // 警告
          }
        }
    }
  }
}
Copy
重点说一下源码在这一部分的处理，HTML对大小写是不敏感的，所有的浏览器会把大写字符解释为小写字符，因此我们在使用DOM中的模板时，cameCase(驼峰命名法)的props名需要使用其等价的 kebab-case (短横线分隔命名) 命代替。 即： <child :aB="test"></child>需要写成<child :a-b="test"></child>

7.2.2 响应式数据props
刚才说到分析props需要两个过程，前面已经针对父组件对props的处理做了描述，而对于子组件而言，我们是通过props选项去接收父组件传递的值。我们再看看子组件对props的处理：

子组件处理props的过程，是发生在父组件_update阶段，这个阶段是Vnode生成真实节点的过程，期间会遇到子Vnode,这时会调用createComponent去实例化子组件。而实例化子组件的过程又回到了_init初始化，此时又会经历选项的合并，针对props选项，最终会统一成{props: { test: { type: null }}}的写法。接着会调用initProps, initProps做的事情，简单概括一句话就是，将组件的props数据设置为响应式数据。

function initProps (vm, propsOptions) {
  var propsData = vm.$options.propsData || {};
  var loop = function(key) {
    ···
    defineReactive(props,key,value,cb)；
    if (!(key in vm)) {
      proxy(vm, "_props", key);
    }
  }
  // 遍历props，执行loop设置为响应式数据。
  for (var key in propsOptions) loop( key );
}
Copy
其中proxy(vm, "_props", key);为props做了一层代理，用户通过vm.XXX可以代理访问到vm._props上的值。针对defineReactive,本质上是利用Object.defineProperty对数据的getter,setter方法进行重写，具体的原理可以参考数据代理章节的内容，在这小节后半段也会有一个基本的实现。

7.3 initMethods
initMethod方法和这一节介绍的响应式没有任何的关系，他的实现也相对简单，主要是保证methods方法定义必须是函数，且命名不能和props重复，最终会将定义的方法都挂载到根实例上。

function initMethods (vm, methods) {
    var props = vm.$options.props;
    for (var key in methods) {
      {
        // method必须为函数形式
        if (typeof methods[key] !== 'function') {
          warn(
            "Method \"" + key + "\" has type \"" + (typeof methods[key]) + "\" in the component definition. " +
            "Did you reference the function correctly?",
            vm
          );
        }
        // methods方法名不能和props重复
        if (props && hasOwn(props, key)) {
          warn(
            ("Method \"" + key + "\" has already been defined as a prop."),
            vm
          );
        }
        //  不能以_ or $.这些Vue保留标志开头
        if ((key in vm) && isReserved(key)) {
          warn(
            "Method \"" + key + "\" conflicts with an existing Vue instance method. " +
            "Avoid defining component methods that start with _ or $."
          );
        }
      }
      // 直接挂载到实例的属性上,可以通过vm[method]访问。
      vm[key] = typeof methods[key] !== 'function' ? noop : bind(methods[key], vm);
    }
  }
Copy
7.4 initData
data在初始化选项合并时会生成一个函数，只有在执行函数时才会返回真正的数据，所以initData方法会先执行拿到组件的data数据，并且会对对象每个属性的命名进行校验，保证不能和props，methods重复。最后的核心方法是observe,observe方法是将数据对象标记为响应式对象，并对对象的每个属性进行响应式处理。与此同时，和props的代理处理方式一样，proxy会对data做一层代理，直接通过vm.XXX可以代理访问到vm._data上挂载的对象属性。

function initData(vm) {
  var data = vm.$options.data;
  // 根实例时，data是一个对象，子组件的data是一个函数，其中getData会调用函数返回data对象
  data = vm._data = typeof data === 'function'? getData(data, vm): data || {};
  var keys = Object.keys(data);
  var props = vm.$options.props;
  var methods = vm.$options.methods;
  var i = keys.length;
  while (i--) {
    var key = keys[i];
    {
      // 命名不能和方法重复
      if (methods && hasOwn(methods, key)) {
        warn(("Method \"" + key + "\" has already been defined as a data property."),vm);
      }
    }
    // 命名不能和props重复
    if (props && hasOwn(props, key)) {
      warn("The data property \"" + key + "\" is already declared as a prop. " + "Use prop default value instead.",vm);
    } else if (!isReserved(key)) {
      // 数据代理，用户可直接通过vm实例返回data数据
      proxy(vm, "_data", key);
    }
  }
  // observe data
  observe(data, true /* asRootData */);
}
Copy
最后讲讲observe,observe具体的行为是将数据对象添加一个不可枚举的属性__ob__，标志对象是一个响应式对象，并且拿到每个对象的属性值，重写getter,setter方法，使得每个属性值都是响应式数据。详细的代码我们后面分析。

7.5 initComputed
和上面的分析方法一样，initComputed是computed数据的初始化,不同之处在于以下几点：

computed可以是对象，也可以是函数，但是对象必须有getter方法,因此如果computed中的属性值是对象时需要进行验证。
针对computed的每个属性，要创建一个监听的依赖，也就是实例化一个watcher,watcher的定义，可以暂时理解为数据使用的依赖本身，一个watcher实例代表多了一个需要被监听的数据依赖。
除了不同点，initComputed也会将每个属性设置成响应式的数据，同样的，也会对computed的命名做检测，防止与props,data冲突。

function initComputed (vm, computed) {
  ···
  for (var key in computed) {
      var userDef = computed[key];
      var getter = typeof userDef === 'function' ? userDef : userDef.get;
      // computed属性为对象时，要保证有getter方法
      if (getter == null) {
        warn(("Getter is missing for computed property \"" + key + "\"."),vm);
      }
      if (!isSSR) {
        // 创建computed watcher
        watchers[key] = new Watcher(vm,getter || noop,noop,computedWatcherOptions);
      }
      if (!(key in vm)) {
        // 设置为响应式数据
        defineComputed(vm, key, userDef);
      } else {
        // 不能和props，data命名冲突
        if (key in vm.$data) {
          warn(("The computed property \"" + key + "\" is already defined in data."), vm);
        } else if (vm.$options.props && key in vm.$options.props) {
          warn(("The computed property \"" + key + "\" is already defined as a prop."), vm);
        }
      }
    }
}
Copy
显然Vue提供了很多种数据供开发者使用，但是分析完后发现每个处理的核心都是将数据转化成响应式数据，有了响应式数据，如何构建一个响应式系统呢？前面提到的watcher又是什么东西？构建响应式系统还需要其他的东西吗？接下来我们尝试着去实现一个极简风的响应式系统。

7.6 极简风的响应式系统
Vue的响应式系统构建是比较复杂的，直接进入源码分析构建的每一个流程会让理解变得困难，因此我觉得在尽可能保留源码的设计逻辑下,用最小的代码构建一个最基础的响应式系统是有必要的。对Dep,Watcher,Observer概念的初步认识，也有助于下一篇对响应式系统设计细节的分析。

7.6.1 框架搭建
我们以MyVue作为类响应式框架，框架的搭建不做赘述。我们模拟Vue源码的实现思路，实例化MyVue时会传递一个选项配置，精简的代码只有一个id挂载元素和一个数据对象data。模拟源码的思路，我们在实例化时会先进行数据的初始化，这一步就是响应式的构建，我们稍后分析。数据初始化后开始进行真实DOM的挂载。

var vm = new MyVue({
  id: '#app',
  data: {
    test: 12
  }
})
// myVue.js
(function(global) {
  class MyVue {
      constructor(options) {
        this.options = options;
        // 数据的初始化
        this.initData(options);
        let el = this.options.id;
        // 实例的挂载
        this.$mount(el);
      }
      initData(options) {
      }
      $mount(el) {
      }
    }
}(window))
Copy
7.6.2 设置响应式对象 - Observer
首先引入一个类Observer,这个类的目的是将数据变成响应式对象，利用Object.defineProperty对数据的getter,setter方法进行改写。在数据读取getter阶段我们会进行依赖的收集，在数据的修改setter阶段，我们会进行依赖的更新(这两个概念的介绍放在后面)。因此在数据初始化阶段，我们会利用Observer这个类将数据对象修改为相应式对象，而这是所有流程的基础。

class MyVue {
  initData(options) {
    if(!options.data) return;
    this.data = options.data;
    // 将数据重置getter，setter方法
    new Observer(options.data);
  }
}
// Observer类的定义
class Observer {
  constructor(data) {
    // 实例化时执行walk方法对每个数据属性重写getter，setter方法
    this.walk(data)
  }

  walk(obj) {
    const keys = Object.keys(obj);
    for(let i = 0;i< keys.length; i++) {
      // Object.defineProperty的处理逻辑
      defineReactive(obj, keys[i])
    }
  }
}
Copy
7.6.3 依赖本身 - Watcher
我们可以这样理解，一个Watcher实例就是一个依赖，数据不管是在渲染模板时使用还是在用户计算时使用，都可以算做一个需要监听的依赖，watcher中记录着这个依赖监听的状态，以及如何更新操作的方法。

// 监听的依赖
class Watcher {
  constructor(expOrFn, isRenderWatcher) {
    this.getter = expOrFn;
    // Watcher.prototype.get的调用会进行状态的更新。
    this.get();
  }

  get() {}
}
Copy
那么哪个时间点会实例化watcher并更新数据状态呢？显然在渲染数据到真实DOM时可以创建watcher。$mount流程前面章节介绍过，会经历模板生成render函数和render函数渲染真实DOM的过程。我们对代码做了精简，updateView浓缩了这一过程。

class MyVue {
  $mount(el) {
    // 直接改写innerHTML
    const updateView = _ => {
      let innerHtml = document.querySelector(el).innerHTML;
      let key = innerHtml.match(/{(\w+)}/)[1];
      document.querySelector(el).innerHTML = this.options.data[key]
    }
    // 创建一个渲染的依赖。
    new Watcher(updateView, true)
  }
}
Copy
7.6.4 依赖管理 - Dep
watcher如果理解为每个数据需要监听的依赖，那么Dep 可以理解为对依赖的一种管理。数据可以在渲染中使用，也可以在计算属性中使用。相应的每个数据对应的watcher也有很多。而我们在更新数据时，如何通知到数据相关的每一个依赖，这就需要Dep进行通知管理了。并且浏览器同一时间只能更新一个watcher,所以也需要一个属性去记录当前更新的watcher。而Dep这个类只需要做两件事情，将依赖进行收集，派发依赖进行更新。

let uid = 0;
class Dep {
  constructor() {
    this.id = uid++;
    this.subs = []
  }
  // 依赖收集
  depend() {
    if(Dep.target) {
      // Dep.target是当前的watcher,将当前的依赖推到subs中
      this.subs.push(Dep.target)
    }
  }
  // 派发更新
  notify() {
    const subs = this.subs.slice();
    for (var i = 0, l = subs.length; i < l; i++) { 
      // 遍历dep中的依赖，对每个依赖执行更新操作
      subs[i].update();
    }
  }
}

Dep.target = null;
Copy
7.6.5 依赖管理过程 - defineReactive
我们看看数据拦截的过程。前面的Observer实例化最终会调用defineReactive重写getter,setter方法。这个方法开始会实例化一个Dep,也就是创建一个数据的依赖管理。在重写的getter方法中会进行依赖的收集，也就是调用dep.depend的方法。在setter阶段，比较两个数不同后，会调用依赖的派发更新。即dep.notify

const defineReactive = (obj, key) => {
  const dep = new Dep();
  const property = Object.getOwnPropertyDescriptor(obj);
  let val = obj[key]
  if(property && property.configurable === false) return;
  Object.defineProperty(obj, key, {
    configurable: true,
    enumerable: true,
    get() {
      // 做依赖的收集
      if(Dep.target) {
        dep.depend()
      }
      return val
    },
    set(nval) {
      if(nval === val) return
      // 派发更新
      val = nval
      dep.notify();
    }
  })
}
Copy
回过头来看watcher,实例化watcher时会将Dep.target设置为当前的watcher,执行完状态更新函数之后，再将Dep.target置空。这样在收集依赖时只要将Dep.target当前的watcher push到Dep的subs数组即可。而在派发更新阶段也只需要重新更新状态即可。

class Watcher {
  constructor(expOrFn, isRenderWatcher) {
    this.getter = expOrFn;
    // Watcher.prototype.get的调用会进行状态的更新。
    this.get();
  }

  get() {
    // 当前执行的watcher
    Dep.target = this
    this.getter()
    Dep.target = null;
  }
  update() {
    this.get()
  }
}
Copy
7.6.6 结果
一个极简的响应式系统搭建完成。在精简代码的同时，保持了源码设计的思想和逻辑。有了这一步的基础，接下来深入分析源码中每个环节的实现细节会更加简单。

7.7 小结
这一节内容，我们正式进入响应式系统的介绍，前面在数据代理章节，我们学过Object.defineProperty,这是一个用来进行数据拦截的方法，而响应式系统构建的基础就是数据的拦截。我们先介绍了Vue内部在初始化数据的过程，最终得出的结论是，不管是data,computed,还是其他的用户定义数据，最终都是调用Object.defineProperty进行数据拦截。而文章的最后，我们在保留源码设计思想和逻辑的前提下，构建出了一个简化版的响应式系统。完整的功能有助于我们下一节对源码具体实现细节的分析和思考。