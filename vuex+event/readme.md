## 跨组件触发事件
##### vuex很好地解决了大型跨组件通信问题，但有些情况使用起来会有些小纠结

```javascript

// button.vue
<button @click="initAllData">
 
// List.vue
<···v-for="item in list">
methods: {
    initAllData () {
        this.initData1()
        this.initData2()
        // dosomething else
    }

```

假设button.vue 和List.vue 是分属比较远的两个组件，button想要触发List的事件，且无数据交互，即initAllData方法严格属于List.vue, 这种场景在vuex内如何实现？
一种做法是利用一个状态来控制：

```javascript


// button.vue
<button @click="initAllData">
initAllData () {
    this.$store.commit('triggerInitData', true)
}
// List.vue
<···v-for="item in list">
// computed 引入triggerInitData
watch：{
    triggerInitData (val) {
        if (val) {
            this.initAllData()
            this.$store.commit('triggerInitData', false)
        }
    }
}
methods: {
    initAllData () {
        this.initData1()
        this.initData2()
        // dosomething else
    }
}

```

但这样的实现总显得有点冗余
  1. 首先，引入了无必要的状态变量triggerInitData
  2. 加长了整个链路，引进了watch

这种场景下我们更想要的其实是类似eventHub的发布订阅模式，因为我们关注的是‘事件’而不是‘数据’。
但如果因此我们就引进一个全局发布订阅的话，缺点也很明显：
  1. 同时存在vuex和一个‘eventHub’，是不是重复了两套逻辑
  2. 通过事件订阅的行为又面临不方便追踪调试的局面
  
针对第一点其实很好解，eventHub将数据操作放到新实例上，通过事件机制完成通信，但没对数据做集中管理；vuex实现了，但vuex的核心在于数据中心，并不关心数据之外的，组件间方法的相互触发。换言之事件绑定-事件触发，以及vuex，更像是两个解决方案，我们应该在数据跨组件通信时候使用vuex，在纯事件类型上探索更好的方法。
于是这个问题现在的焦点在于，如何更好地在组件间触发事件，使之对开发者透明，方便追踪调试，同时不干扰正常的数据流？

#### mutation和action，同步异步，动机拆解
思考上面问题前，我们先回过头来看vuex的设计, 以及为什么有mutation和action
vuex为了能在每次数据变化前后做跟踪，建立了mutation，约定每次数据的修改都应该通过commit方法进行，我们可以把commit理解为类似下面的实现

```javascript


state: { data: 0 },
mutations: {
    setData (state, val) {
        state.data = val
    }
}
// 组件内调用
this.$store.commit('setData', 123)
// store.commit 类似实现
function commit (evt, val) {
    store.mutations[evt](store.state, data)
    console.log('检测到commit之后变化': data)
}

```

查看vuex的api，也可以发现，vuex对插件暴露的接口subscribe，也是在每个 mutation 完成后调用，这时候我们打开vue-tool, 选择第2个tab:vuex, 可以看到，vuetool这类工具对每个mutation行为都进行了追踪。

  1. mutation 只能返回同步状态，如上述代码，如果mutations[evt]是异步函数，commit里面之后获取的data都是无意义的，此时真正的data还未返回
  2. 我们当然可以在commit里面以.then的方式书写调试逻辑，但这样就得约定所有mutation方法以promise方式书写，并且还牺牲了本身是同步状态的函数。
  3. 更好的做法是新增一个action用来处理异步，确保mutation是同步，不管action什么逻辑，只要最后触发commit，提交mutation就行了，这样数据的变化最终仍会经过mutation追踪， 在诸如vuetool工具里呈现.
  
从vuex的这些设计看来，很关注的一点是数据的可维护性，数据在进行变更时候，应该是可追踪的。结合上一段的问题，如果想在组件间触发事件，那最大的原则是不应该破坏数据在变更时候的可检测性，都应该经过mutation层，在此基础上，事件的行为本身最好也能被追踪记录。

#### 封装发布订阅，让vuetool可调试追踪

确立了需求和原则后，我们终于可以优雅地写代码了，我们整理下小目标：
  > 1. vuex项目内，引进发布订阅
  > 2. 利用mutation, 使"事件触发"这个行为被记录
  > 3. 优化封装代码，约定和确保规范，使通信过程无数据传递，以免漏测数据流
  
我们简单快速实现下第1点， 在一个vue-cli2搭建起来的项目中，我们直接在main.js中插入：

```javascript

import store from './store'
···
store.$events = {}
store.$on = function (evt, fn) {
  store.$events['$' + evt] = fn
}
store.$off = function (evt) {
  store.$events['$' + evt] = null
}
 
store.$emit = function (evt, data) {
  if (!this.$events['$' + evt]) return
  this.$events['$' + evt](data)
}
// 绑定
// this.$store.$on('test', () => {
//   console.log('test')
// })
 
// 调用
// this.$store.$emit('test')
···

```

这样就有个简单的雏形，也确定了大概的调用方式，接着我们思考第2点，如何让这个事件行为像mutation方法一样能被检测到。
一个最简单的思路是，在$emit时候,我们也提交一个mutation, 使行为本身能通过mutation被记录。结合vuex的动态加载模块功能，我们尝试一下：

```javascript


store.registerModule('myEvents', {
  mutations: {
    setEvent () {}
  }
})
store.$events = {}
store.$on = function (evt, fn) {
  store.$events['$' + evt] = fn
}
store.$off = function (evt) {
  store.$events['$' + evt] = null
}
 
store.$emit = function (evt, data) {
  if (!this.$events['$' + evt]) return
  this.$events['$' + evt](data)
  this.commit('setEvent', evt)  // 将事件evt当成payload提交给mutation
}

```

现在试下触发一个事件，我们在vuetool可以看到，事件也被记录下来了，并且payload就为触发的事件名：

 这样，我们基本实现一套功能了，在继续优化之前，唯独针对第三条小目标思考下，假若我们严格限制$emit参数的传递，防止不经过vuex的数据出现，那我们应该这样子写：
 
 ```javascript



store.$emit = function (evt) {
  if (!this.$events['$' + evt]) return
  this.$events['$' + evt]()
  this.commit('setEvent', evt)  // 将事件evt当成payload提交给mutation
}

```

假若我们传递的是组件间都公用的数据，是的，我们应当抽取到vuex，并且在维护一个单纯的事件触发。但考虑到实际场景，我们也有可能针对一个开关事件传递一个boolean, 或者根据操作类别返回一个选择0，1，2之类。为此，对emit方法的限制，更好地做法是把选择交给开发，并提出约定。

#### 完善代码，抽离逻辑，约定规范

现在我们优化封装下我们的代码，考虑这两点：
  1. 这套事件机制可以直接打到vuex对象上
  2. 发布订阅这套逻辑可以抽取出来，在任意其他对象也可以使用
