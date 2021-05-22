7.8 相关概念
在构建简易式响应式系统的时候，我们引出了几个重要的概念，他们都是响应式原理设计的核心，我们先简单回顾一下：

Observer类，实例化一个Observer类会通过Object.defineProperty对数据的getter,setter方法进行改写，在getter阶段进行依赖的收集,在数据发生更新阶段，触发setter方法进行依赖的更新
watcher类，实例化watcher类相当于创建一个依赖，简单的理解是数据在哪里被使用就需要产生了一个依赖。当数据发生改变时，会通知到每个依赖进行更新，前面提到的渲染wathcer便是渲染dom时使用数据产生的依赖。
Dep类，既然watcher理解为每个数据需要监听的依赖，那么对这些依赖的收集和通知则需要另一个类来管理，这个类便是Dep,Dep需要做的只有两件事，收集依赖和派发更新依赖。
这是响应式系统构建的三个基本核心概念，也是这一节的基础，如果还没有印象，请先回顾上一节对极简风响应式系统的构建。

7.9 data
7.9.1 问题思考
在开始分析data之前，我们先抛出几个问题让读者思考，而答案都包含在接下来内容分析中。

前面已经知道，Dep是作为管理依赖的容器，那么这个容器在什么时候产生？也就是实例化Dep发生在什么时候？

Dep收集了什么类型的依赖？即watcher作为依赖的分类有哪些，分别是什么场景，以及区别在哪里？

Observer这个类具体对getter,setter方法做了哪些事情？
手写的watcher和页面数据渲染监听的watch如果同时监听到数据的变化，优先级怎么排？
有了依赖的收集是不是还有依赖的解除，依赖解除的意义在哪里？
带着这几个问题，我们开始对data的响应式细节展开分析。

7.9.2 依赖收集
data在初始化阶段会实例化一个Observer类，这个类的定义如下(忽略数组类型的data):

// initData 
function initData(data) {
  ···
  observe(data, true)
}
// observe
function observe(value, asRootData) {
  ···
  ob = new Observer(value);
  return ob
}

// 观察者类，对象只要设置成拥有观察属性，则对象下的所有属性都会重写getter和setter方法，而getter，setting方法会进行依赖的收集和派发更新
var Observer = function Observer (value) {
    ···
    // 将__ob__属性设置成不可枚举属性。外部无法通过遍历获取。
    def(value, '__ob__', this);
    // 数组处理
    if (Array.isArray(value)) {
        ···
    } else {
      // 对象处理
      this.walk(value);
    }
  };

function def (obj, key, val, enumerable) {
  Object.defineProperty(obj, key, {
    value: val,
    enumerable: !!enumerable, // 是否可枚举
    writable: true,
    configurable: true
  });
}
Copy
Observer会为data添加一个__ob__属性， __ob__属性是作为响应式对象的标志，同时def方法确保了该属性是不可枚举属性，即外界无法通过遍历获取该属性值。除了标志响应式对象外，Observer类还调用了原型上的walk方法，遍历对象上每个属性进行getter,setter的改写。

Observer.prototype.walk = function walk (obj) {
    // 获取对象所有属性，遍历调用defineReactive###1进行改写
    var keys = Object.keys(obj);
    for (var i = 0; i < keys.length; i++) {
        defineReactive###1(obj, keys[i]);
    }
};
Copy
defineReactive###1是响应式构建的核心，它会先实例化一个Dep类，即为每个数据都创建一个依赖的管理，之后利用Object.defineProperty重写getter,setter方法。这里我们只分析依赖收集的代码。

function defineReactive###1 (obj,key,val,customSetter,shallow) {
    // 每个数据实例化一个Dep类，创建一个依赖的管理
    var dep = new Dep();

    var property = Object.getOwnPropertyDescriptor(obj, key);
    // 属性必须满足可配置
    if (property && property.configurable === false) {
      return
    }
    // cater for pre-defined getter/setters
    var getter = property && property.get;
    var setter = property && property.set;
    // 这一部分的逻辑是针对深层次的对象，如果对象的属性是一个对象，则会递归调用实例化Observe类，让其属性值也转换为响应式对象
    var childOb = !shallow && observe(val);
    Object.defineProperty(obj, key, {
      enumerable: true,
      configurable: true,s
      get: function reactiveGetter () {
        var value = getter ? getter.call(obj) : val;
        if (Dep.target) {
          // 为当前watcher添加dep数据
          dep.depend();
          if (childOb) {
            childOb.dep.depend();
            if (Array.isArray(value)) {
              dependArray(value);
            }
          }
        }
        return value
      },
      set: function reactiveSetter (newVal) {}
    });
  }
Copy
主要看getter的逻辑，我们知道当data中属性值被访问时，会被getter函数拦截，根据我们旧有的知识体系可以知道，实例挂载前会创建一个渲染watcher。

new Watcher(vm, updateComponent, noop, {
  before: function before () {
    if (vm._isMounted && !vm._isDestroyed) {
      callHook(vm, 'beforeUpdate');
    }
  }
}, true /* isRenderWatcher */);
Copy
与此同时，updateComponent的逻辑会执行实例的挂载，在这个过程中，模板会被优先解析为render函数，而render函数转换成Vnode时，会访问到定义的data数据，这个时候会触发gettter进行依赖收集。而此时数据收集的依赖就是这个渲染watcher本身。

代码中依赖收集阶段会做下面几件事：

为当前的watcher(该场景下是渲染watcher)添加拥有的数据。
为当前的数据收集需要监听的依赖
如何理解这两点？我们先看代码中的实现。getter阶段会执行dep.depend(),这是Dep这个类定义在原型上的方法。

dep.depend();


Dep.prototype.depend = function depend () {
    if (Dep.target) {
      Dep.target.addDep(this);
    }
  };
Copy
Dep.target为当前执行的watcher,在渲染阶段，Dep.target为组件挂载时实例化的渲染watcher,因此depend方法又会调用当前watcher的addDep方法为watcher添加依赖的数据。

Watcher.prototype.addDep = function addDep (dep) {
    var id = dep.id;
    if (!this.newDepIds.has(id)) {
      // newDepIds和newDeps记录watcher拥有的数据
      this.newDepIds.add(id);
      this.newDeps.push(dep);
      // 避免重复添加同一个data收集器
      if (!this.depIds.has(id)) {
        dep.addSub(this);
      }
    }
  };
Copy
其中newDepIds是具有唯一成员是Set数据结构，newDeps是数组，他们用来记录当前watcher所拥有的数据，这一过程会进行逻辑判断，避免同一数据添加多次。

addSub为每个数据依赖收集器添加需要被监听的watcher。

Dep.prototype.addSub = function addSub (sub) {
  //将当前watcher添加到数据依赖收集器中
    this.subs.push(sub);
};
Copy
getter如果遇到属性值为对象时，会为该对象的每个值收集依赖
这句话也很好理解，如果我们将一个值为基本类型的响应式数据改变成一个对象，此时新增对象里的属性，也需要设置成响应式数据。

遇到属性值为数组时，进行特殊处理，这点放到后面讲。
通俗的总结一下依赖收集的过程，每个数据就是一个依赖管理器，而每个使用数据的地方就是一个依赖。当访问到数据时，会将当前访问的场景作为一个依赖收集到依赖管理器中，同时也会为这个场景的依赖收集拥有的数据。

7.9.3 派发更新
在分析依赖收集的过程中，可能会有不少困惑，为什么要维护这么多的关系？在数据更新时，这些关系会起到什么作用？带着疑惑，我们来看看派发更新的过程。 在数据发生改变时，会执行定义好的setter方法，我们先看源码。

Object.defineProperty(obj,key, {
  ···
  set: function reactiveSetter (newVal) {
      var value = getter ? getter.call(obj) : val;
      // 新值和旧值相等时，跳出操作
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return
      }
      ···
      // 新值为对象时，会为新对象进行依赖收集过程
      childOb = !shallow && observe(newVal);
      dep.notify();
    }
})
Copy
派发更新阶段会做以下几件事：

判断数据更改前后是否一致，如果数据相等则不进行任何派发更新操作。
新值为对象时，会对该值的属性进行依赖收集过程。
通知该数据收集的watcher依赖,遍历每个watcher进行数据更新,这个阶段是调用该数据依赖收集器的dep.notify方法进行更新的派发。
Dep.prototype.notify = function notify () {
  var subs = this.subs.slice();
  if (!config.async) {
    // 根据依赖的id进行排序
    subs.sort(function (a, b) { return a.id - b.id; });
  }
  for (var i = 0, l = subs.length; i < l; i++) {
    // 遍历每个依赖，进行更新数据操作。
    subs[i].update();
  }
};
Copy
更新时会将每个watcher推到队列中，等待下一个tick到来时取出每个watcher进行run操作
Watcher.prototype.update = function update () {
  ···
  queueWatcher(this);
};
Copy
queueWatcher方法的调用，会将数据所收集的依赖依次推到queue数组中,数组会在下一个事件循环'tick'中根据缓冲结果进行视图更新。而在执行视图更新过程中，难免会因为数据的改变而在渲染模板上添加新的依赖，这样又会执行queueWatcher的过程。所以需要有一个标志位来记录是否处于异步更新过程的队列中。这个标志位为flushing,当处于异步更新过程时，新增的watcher会插入到queue中。
function queueWatcher (watcher) {
  var id = watcher.id;
  // 保证同一个watcher只执行一次
  if (has[id] == null) {
    has[id] = true;
    if (!flushing) {
      queue.push(watcher);
    } else {
      var i = queue.length - 1;
      while (i > index && queue[i].id > watcher.id) {
        i--;
      }
      queue.splice(i + 1, 0, watcher);
    }
    ···
    nextTick(flushSchedulerQueue);
  }
}
Copy
nextTick的原理和实现先不讲，概括来说，nextTick会缓冲多个数据处理过程，等到下一个事件循环tick中再去执行DOM操作，它的原理，本质是利用事件循环的微任务队列实现异步更新。
当下一个tick到来时，会执行flushSchedulerQueue方法，它会拿到收集的queue数组(这是一个watcher的集合),并对数组依赖进行排序。为什么进行排序呢？源码中解释了三点：

组件创建是先父后子，所以组件的更新也是先父后子，因此需要保证父的渲染watcher优先于子的渲染watcher更新。
用户自定义的watcher,称为user watcher。 user watcher和render watcher执行也有先后，由于user watchers比render watcher要先创建，所以user watcher要优先执行。
如果一个组件在父组件的 watcher 执行阶段被销毁，那么它对应的 watcher 执行都可以被跳过。
function flushSchedulerQueue () {
    currentFlushTimestamp = getNow();
    flushing = true;
    var watcher, id;
    // 对queue的watcher进行排序
    queue.sort(function (a, b) { return a.id - b.id; });
    // 循环执行queue.length，为了确保由于渲染时添加新的依赖导致queue的长度不断改变。
    for (index = 0; index < queue.length; index++) {
      watcher = queue[index];
      // 如果watcher定义了before的配置，则优先执行before方法
      if (watcher.before) {
        watcher.before();
      }
      id = watcher.id;
      has[id] = null;
      watcher.run();
      // in dev build, check and stop circular updates.
      if (has[id] != null) {
        circular[id] = (circular[id] || 0) + 1;
        if (circular[id] > MAX_UPDATE_COUNT) {
          warn(
            'You may have an infinite update loop ' + (
              watcher.user
                ? ("in watcher with expression \"" + (watcher.expression) + "\"")
                : "in a component render function."
            ),
            watcher.vm
          );
          break
        }
      }
    }

    // keep copies of post queues before resetting state
    var activatedQueue = activatedChildren.slice();
    var updatedQueue = queue.slice();
    // 重置恢复状态，清空队列
    resetSchedulerState();

    // 视图改变后，调用其他钩子
    callActivatedHooks(activatedQueue);
    callUpdatedHooks(updatedQueue);

    // devtool hook
    /* istanbul ignore if */
    if (devtools && config.devtools) {
      devtools.emit('flush');
    }
  }
Copy
flushSchedulerQueue阶段，重要的过程可以总结为四点：

对queue中的watcher进行排序，原因上面已经总结。
遍历watcher,如果当前watcher有before配置，则执行before方法，对应前面的渲染watcher:在渲染watcher实例化时，我们传递了before函数，即在下个tick更新视图前，会调用beforeUpdate生命周期钩子。
执行watcher.run进行修改的操作。
重置恢复状态，这个阶段会将一些流程控制的状态变量恢复为初始值，并清空记录watcher的队列。
new Watcher(vm, updateComponent, noop, {
  before: function before () {
    if (vm._isMounted && !vm._isDestroyed) {
      callHook(vm, 'beforeUpdate');
    }
  }
}, true /* isRenderWatcher */);
Copy
重点看看watcher.run()的操作。

Watcher.prototype.run = function run () {
    if (this.active) {
      var value = this.get();
      if ( value !== this.value || isObject(value) || this.deep ) {
        // 设置新值
        var oldValue = this.value;
        this.value = value;
        // 针对user watcher，暂时不分析
        if (this.user) {
          try {
            this.cb.call(this.vm, value, oldValue);
          } catch (e) {
            handleError(e, this.vm, ("callback for watcher \"" + (this.expression) + "\""));
          }
        } else {
          this.cb.call(this.vm, value, oldValue);
        }
      }
    }
  };
Copy
首先会执行watcher.prototype.get的方法，得到数据变化后的当前值，之后会对新值做判断，如果判断满足条件，则执行cb,cb为实例化watcher时传入的回调。

在分析get方法前，回头看看watcher构造函数的几个属性定义

var watcher = function Watcher(
  vm, // 组件实例
  expOrFn, // 执行函数
  cb, // 回调
  options, // 配置
  isRenderWatcher // 是否为渲染watcher
) {
  this.vm = vm;
    if (isRenderWatcher) {
      vm._watcher = this;
    }
    vm._watchers.push(this);
    // options
    if (options) {
      this.deep = !!options.deep;
      this.user = !!options.user;
      this.lazy = !!options.lazy;
      this.sync = !!options.sync;
      this.before = options.before;
    } else {
      this.deep = this.user = this.lazy = this.sync = false;
    }
    this.cb = cb;
    this.id = ++uid$2; // uid for batching
    this.active = true;
    this.dirty = this.lazy; // for lazy watchers
    this.deps = [];
    this.newDeps = [];
    this.depIds = new _Set();
    this.newDepIds = new _Set();
    this.expression = expOrFn.toString();
    // parse expression for getter
    if (typeof expOrFn === 'function') {
      this.getter = expOrFn;
    } else {
      this.getter = parsePath(expOrFn);
      if (!this.getter) {
        this.getter = noop;
        warn(
          "Failed watching path: \"" + expOrFn + "\" " +
          'Watcher only accepts simple dot-delimited paths. ' +
          'For full control, use a function instead.',
          vm
        );
      }
    }
    // lazy为计算属性标志，当watcher为计算watcher时，不会理解执行get方法进行求值
    this.value = this.lazy
      ? undefined
      : this.get();

}
Copy
方法get的定义如下：

Watcher.prototype.get = function get () {
    pushTarget(this);
    var value;
    var vm = this.vm;
    try {
      value = this.getter.call(vm, vm);
    } catch (e) {
     ···
    } finally {
      ···
      // 把Dep.target恢复到上一个状态，依赖收集过程完成
      popTarget();
      this.cleanupDeps();
    }
    return value
  };
Copy
get方法会执行this.getter进行求值，在当前渲染watcher的条件下,getter会执行视图更新的操作。这一阶段会重新渲染页面组件

new Watcher(vm, updateComponent, noop, { before: () => {} }, true);

updateComponent = function () {
  vm._update(vm._render(), hydrating);
};
Copy
执行完getter方法后，最后一步会进行依赖的清除，也就是cleanupDeps的过程。

关于依赖清除的作用，我们列举一个场景： 我们经常会使用v-if来进行模板的切换，切换过程中会执行不同的模板渲染，如果A模板监听a数据，B模板监听b数据，当渲染模板B时，如果不进行旧依赖的清除，在B模板的场景下，a数据的变化同样会引起依赖的重新渲染更新，这会造成性能的浪费。因此旧依赖的清除在优化阶段是有必要。

// 依赖清除的过程
  Watcher.prototype.cleanupDeps = function cleanupDeps () {
    var i = this.deps.length;
    while (i--) {
      var dep = this.deps[i];
      if (!this.newDepIds.has(dep.id)) {
        dep.removeSub(this);
      }
    }
    var tmp = this.depIds;
    this.depIds = this.newDepIds;
    this.newDepIds = tmp;
    this.newDepIds.clear();
    tmp = this.deps;
    this.deps = this.newDeps;
    this.newDeps = tmp;
    this.newDeps.length = 0;
  };
Copy
把上面分析的总结成依赖派发更新的最后两个点

执行run操作会执行getter方法,也就是重新计算新值，针对渲染watcher而言，会重新执行updateComponent进行视图更新
重新计算getter后，会进行依赖的清除
7.10 computed
计算属性设计的初衷是用于简单运算的，毕竟在模板中放入太多的逻辑会让模板过重且难以维护。在分析computed时，我们依旧遵循依赖收集和派发更新两个过程进行分析。

7.10.1 依赖收集
computed的初始化过程，会遍历computed的每一个属性值，并为每一个属性实例化一个computed watcher，其中{ lazy: true}是computed watcher的标志，最终会调用defineComputed将数据设置为响应式数据，对应源码如下：

function initComputed() {
  ···
  for(var key in computed) {
    watchers[key] = new Watcher(
        vm,
        getter || noop,
        noop,
        computedWatcherOptions
      );
  }
  if (!(key in vm)) {
    defineComputed(vm, key, userDef);
  }
}

// computed watcher的标志，lazy属性为true
var computedWatcherOptions = { lazy: true };
Copy
defineComputed的逻辑和分析data的逻辑相似，最终调用Object.defineProperty进行数据拦截。具体的定义如下：

function defineComputed (target,key,userDef) {
  // 非服务端渲染会对getter进行缓存
  var shouldCache = !isServerRendering();
  if (typeof userDef === 'function') {
    // 
    sharedPropertyDefinition.get = shouldCache
      ? createComputedGetter(key)
      : createGetterInvoker(userDef);
    sharedPropertyDefinition.set = noop;
  } else {
    sharedPropertyDefinition.get = userDef.get
      ? shouldCache && userDef.cache !== false
        ? createComputedGetter(key)
        : createGetterInvoker(userDef.get)
      : noop;
    sharedPropertyDefinition.set = userDef.set || noop;
  }
  if (sharedPropertyDefinition.set === noop) {
    sharedPropertyDefinition.set = function () {
      warn(
        ("Computed property \"" + key + "\" was assigned to but it has no setter."),
        this
      );
    };
  }
  Object.defineProperty(target, key, sharedPropertyDefinition);
}
Copy
在非服务端渲染的情形，计算属性的计算结果会被缓存，缓存的意义在于，只有在相关响应式数据发生变化时，computed才会重新求值，其余情况多次访问计算属性的值都会返回之前计算的结果，这就是缓存的优化，computed属性有两种写法，一种是函数，另一种是对象，其中对象的写法需要提供getter和setter方法。

当访问到computed属性时，会触发getter方法进行依赖收集，看看createComputedGetter的实现。

function createComputedGetter (key) {
    return function computedGetter () {
      var watcher = this._computedWatchers && this._computedWatchers[key];
      if (watcher) {
        if (watcher.dirty) {
          watcher.evaluate();
        }
        if (Dep.target) {
          watcher.depend();
        }
        return watcher.value
      }
    }
  }
Copy
createComputedGetter返回的函数在执行过程中会先拿到属性的computed watcher,dirty是标志是否已经执行过计算结果，如果执行过则不会执行watcher.evaluate重复计算，这也是缓存的原理。

Watcher.prototype.evaluate = function evaluate () {
    // 对于计算属性而言 evaluate的作用是执行计算回调
    this.value = this.get();
    this.dirty = false;
  };
Copy
get方法前面介绍过，会调用实例化watcher时传递的执行函数，在computer watcher的场景下，执行函数是计算属性的计算函数，他可以是一个函数，也可以是对象的getter方法。

列举一个场景避免和data的处理脱节，computed在计算阶段，如果访问到data数据的属性值，会触发data数据的getter方法进行依赖收集，根据前面分析，data的Dep收集器会将当前watcher作为依赖进行收集，而这个watcher就是computed watcher，并且会为当前的watcher添加访问的数据Dep

回到计算执行函数的this.get()方法，getter执行完成后同样会进行依赖的清除，原理和目的参考data阶段的分析。get执行完毕后会进入watcher.depend进行依赖的收集。收集过程和data一致,将当前的computed watcher作为依赖收集到数据的依赖收集器Dep中。

这就是computed依赖收集的完整过程，对比data的依赖收集，computed会对运算的结果进行缓存，避免重复执行运算过程。

7.10.2 派发更新
派发更新的条件是data中数据发生改变，所以大部分的逻辑和分析data时一致，我们做一个总结。

当计算属性依赖的数据发生更新时，由于数据的Dep收集过computed watch这个依赖，所以会调用dep的notify方法，对依赖进行状态更新。
此时computed watcher和之前介绍的watcher不同，它不会立刻执行依赖的更新操作，而是通过一个dirty进行标记。我们再回头看依赖更新的代码。
Dep.prototype.notify = function() {
  ···
   for (var i = 0, l = subs.length; i < l; i++) {
      subs[i].update();
    }
}

Watcher.prototype.update = function update () {
  // 计算属性分支  
  if (this.lazy) {
    this.dirty = true;
  } else if (this.sync) {
    this.run();
  } else {
    queueWatcher(this);
  }
};
Copy
由于lazy属性的存在，update过程不会执行状态更新的操作，只会将dirty标记为true。

由于data数据拥有渲染watcher这个依赖，所以同时会执行updateComponent进行视图重新渲染,而render过程中会访问到计算属性,此时由于this.dirty值为true,又会对计算属性重新求值。
7.11 小结
我们在上一节的理论基础上深入分析了Vue如何利用data,computed构建响应式系统。响应式系统的核心是利用Object.defineProperty对数据的getter,setter进行拦截处理，处理的核心是在访问数据时对数据所在场景的依赖进行收集，在数据发生更改时，通知收集过的依赖进行更新。这一节我们详细的介绍了data,computed对响应式的处理，两者处理逻辑存在很大的相似性但却各有的特性。源码中会computed的计算结果进行缓存，避免了在多个地方使用时频繁重复计算的问题。由于篇幅有限，对于用户自定义的watcher我们会放到下一小节分析。文章还留有一个疑惑，依赖收集时如果遇到的数据是数组时应该怎么处理，这些疑惑都会在之后的文章一一解开。