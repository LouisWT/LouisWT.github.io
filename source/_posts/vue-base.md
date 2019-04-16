---
layout: simple-article
title: Vue 学习笔记一
categories:
    - Vue
tags:
    - Vue
---
最近开始学习 Vue 框架，发现 Vue 的文档写的是真的好，想要学习的小伙伴建议读文档就够了，这篇文章仅作为我自己梳理使用，毕竟好记性不如烂笔头 :D
<!-- more -->

# 一. 模板语法
## 插值
- EXP 表示data中的变量名的字符串 或者 可以返回字符串的单个 JS 表达式
- ATTR 表示元素的属性名 或者 组件的 props 属性名
- ENT 表示事件名称,比如 click

| 类型                 | 语法              |
| -------------------- | ----------------- |
| 元素内容插值         | {{ EXP }}         |
| 元素属性插值         | v-bind:ATTR="EXP" 或 :ATTR="EXP" |
| 监听DOM事件          | v-on:ENT="EXP" 或 @ENT="EXP" |

## 计算参数
计算参数主要用于动态指定 v-bind 和 v-on 中的 ATTR 和 ENT

**动态参数中不能有空格引号还有大写键名**
```html
<a v-bind:[attributeName]="url"> ... </a>
<!-- data 中的 attributeName 等于 herf -->
<a v-bind:href="url"> ... </a>

<a v-on:[eventName]="doSomething"> ... </a>
<!-- data 中的 eventName 等于 click -->
<a v-on:click="doSomething"> ... </a>

<!-- 不能有引号和空格,这会触发一个编译警告 -->
<a v-bind:['foo' + bar]="value"> ... </a>
<!-- 在 DOM 中使用模板时这段代码会被转换为 `v-bind:[someattr]` -->
<a v-bind:[someAttr]="value"> ... </a>
```

## 条件渲染
| 类型                 | 语法              |
| -------------------- | ----------------- |
| 决定插入或者移除元素 | v-if="EXP"  v-else-if="EXP1"（可以加可以不加） v-else（可以加也可以不加） |
| 是否显示元素(元素只是会不显示，但是不会移除) | v-show="EXP" |

- 如果元素需要频繁切换是否显示，那么 v-show 更好；
- 如果运行时条件很少改变，那么 v-if 更好
- 不要在同一元素上使用 v-if 和 v-for

## 列表渲染
| 类型                 | 语法              |
| -------------------- | ----------------- |
| 根据源数据数组进行渲染 | v-for='item in items' (items 是源数据数组的变量名)
| 根据源数据对象进行渲染 | v-for='(value, key) in obj' (obj 是源数据对象的变量名)
| 列表渲染时需要使用 v-bind 绑定 key 来方便重用元素 | v-for='item in items' :key="item.id" |

```html
<ul id="example-2">
  <li v-for="(item, index) in items">
    {{ parentMessage }} - {{ index }} - {{ item.message }}
  </li>
</ul>
<script>
var example2 = new Vue({
  el: '#example-2',
  data: {
    parentMessage: 'Parent',
    items: [
      { message: 'Foo' },
      { message: 'Bar' }
    ]
  }
})
</script>
```

### 列表数据修改
```js
// 错误，视图不能响应变化
example2.data.items[1] = { message: 'Hi' };

// 使用 splice 来代替数组中的某个元素
example2.data.items.splice(1, 1, { message: 'Hi' })
// Vue.set 来替换元素
Vue.set(example2.data.items, 1, { message: 'Hi' })
```

### 对象属性添加
```js
var vm = new Vue({
  data: {
    // `vm.a` 现在是响应式的
    a: 1
  }
})
// `vm.b` 不是响应式的
vm.b = 2

// 使用 vm.$set 或者 Vue.set
vm.$set(vm, 'b', 2);
Vue.set(vm, 'b', 2);
```
**添加多个属性时，可以使用 Object.assign，但是不能直接在原对象的基础上复制，而应该返回一个新对象**
```js
// 错误，视图不会响应
Object.assign(vm.userProfile, {
  age: 27,
  favoriteColor: 'Vue Green'
})

// 给属性指定一个新对象
vm.userProfile = Object.assign({}, vm.userProfile, {
  age: 27,
  favoriteColor: 'Vue Green'
})
```

### v-for 与 v-if 同时使用
v-for 比 v-if 的优先级高，因此如果在同一个元素上使用，就表示对元素使用 v-for，然后对每个元素再判断 v-if 来决定这个元素是否显示。然而 v-if 的过滤过程完全可以通过方法或者计算属性实现，因此这个用法是不必要的。

如果想要通过判断 v-if 来决定是否进入 v-for，那么可以在上层元素使用 v-if

## 事件处理
`v-on:ENT="EXP"`，EXP 可以是一段代码 或者 一个函数名 或者 一个带参数的函数调用
- 代码: `<button v-on:click="counter += 1">Add 1</button>` 点击之后 counter 变量加1
- 函数名：`<button v-on:click="greet">Greet</button>` 点击之后，调用 methods 中定义的 greet 方法，greet 方法只有一个参数，就是 event
- 函数调用：`<button v-on:click="warn('No.', $event)">Submit</button>` warn 方法有两个参数，原生的event对象通过 $event 传入

### 事件修饰符
| 功能 | 语法 |
| --- | --- |
| 阻止事件继续传播 | v.on:ENT.stop="EXP"
| 阻止事件的默认处理程序(preventDefault()) | v-on:ENT.prevent="EXP"
| 使用事件捕获而不是事件冒泡 | v-on:ENT.capture="EXP"
| 只在事件是在当前元素自身触发（不是内部元素触发）的，才处理事件 | v-on:ENT.self="EXP"
| 事件只会处理一次 | v-on:ENT.once="EXP"
| 事件会直接完成默认行为而不等待事件处理程序完成 | v-on:ENT.passive="EXP"
| 事件修饰符可以串联使用（顺序对效果有影响） | v-on:ENT.stop.prevent="EXP"
| 可以只使用事件修饰符而不指定事件处理程序 | v-on:ENT.prevent

>  .passive 选项相当于默认事件处理程序不会调用 preventDefault，所以直接调用默认行为，这对提升移动端性能非常有用，比如说滚动的时候，直接进行滚动，而不会等待滚动的事件处理程序执行完成

### 按键修饰符
可以在监听**键盘事件**时添加按键修饰符：
```html
<!-- enter 是按键别名，表示 enter 键，如果是 enter 键的 keyup 事件，那么就调用事件处理程序 -->
<input v-on:keyup.enter="submit">
```
#### 正常按键：
- .enter
- .tab
- .delete (捕获“删除”和“退格”键)
- .esc
- .space
- .up
- .down
- .left
- .right

#### 系统修饰键：
- .ctrl
- .alt
- .shift
- .meta（window 键或者 command 键）
> 修饰键与常规按键不同，在和 keyup 事件一起用时，事件触发时修饰键必须处于按下状态。换句话说，只有在按住 ctrl 的情况下释放其它按键，才能触发 keyup.ctrl
```html
<!-- 67 表示 c，所以是 alt + c 会触发事件处理函数 -->
<input @keyup.alt.67="clear">
```

exact 修饰符:
```html
<!-- 这样的话如果按下的键是 ctrl + alt 再点击，也会触发事件 -->
<button @click.ctrl="onClick">A</button>

<!-- 这样的话只能是按下 ctrl 再点击才会触发，如果还同时按下了别的修饰键，是没有效果的 -->
<button @click.ctrl.exact="onClick">A</button>

<!-- 没有按下任何系统修饰键的情况下，点击才触发事件 -->
<button @click.exact="onClick">A</button>
```
### 鼠标修饰符
可以跟在**鼠标点击事件**之后
- .left
- .right
- .middle

## 表单输入绑定

### 单行文本
```html
<input v-model="message" placeholder="edit me">
<p>Message is: {{ message }}</p>

<!-- 等价于 -->
<input
  v-bind:value="message"
  v-on:input="message = $event.target.value"
>
```

### 多行文本
```html
<textarea v-model="message" placeholder="add multiple lines"></textarea>
<p style="white-space: pre-line;">{{ message }}</p>
```
### 复选框
单个复选框：
```html
<!-- 单个复选框，绑定的值是布尔值，表示是否选中 -->
<input type="checkbox" id="checkbox" v-model="checked">
<!-- checked 是 true/false -->
<label for="checkbox">{{ checked }}</label>
```
多个复选框：
```html
<!-- 多个复选框，绑定的值是一个数组，数组的每一项是被选中的值 -->
<div id='example-3'>
  <input type="checkbox" id="jack" value="Jack" v-model="checkedNames">
  <label for="jack">Jack</label>
  <input type="checkbox" id="john" value="John" v-model="checkedNames">
  <label for="john">John</label>
  <input type="checkbox" id="mike" value="Mike" v-model="checkedNames">
  <label for="mike">Mike</label>
  <br>
  <span>Checked names: {{ checkedNames }}</span>
</div>
<script>
new Vue({
  el: '#example-3',
  data: {
    // checkedNames 的值是 Jack John Mike 的数组
    checkedNames: []
  }
})
</script>
```
### 单选按钮
```html
<div id="example-4">
  <input type="radio" id="one" value="One" v-model="picked">
  <label for="one">One</label>
  <input type="radio" id="two" value="Two" v-model="picked">
  <label for="two">Two</label>
  <span>Picked: {{ picked }}</span>
</div>
<script>
new Vue({
  el: '#example-4',
  // picked 的值会是选中的值，One / Two
  data: {
    picked: ''
  }
})
</script>
```
### 下拉选择框
```html
<div id="example-5">
  <select v-model="selected">
    <option disabled value="">请选择</option>
    <option>A</option>
    <option>B</option>
    <option>C</option>
  </select>
  <span>Selected: {{ selected }}</span>
</div>
<script>
new Vue({
  el: '...',
  data: {
    // 选中的值的 value，A / B / C
    selected: ''
  }
})
</script>
```

### 表单修饰符
- .lazy: 默认的时候 v-model 在每次 input 事件后，将输入框内容与数据进行同步，可以添加 .lazy 修饰符，使得与 change 事件同步 `<input v-model.lazy="msg">`
- .number: 自动将用户的输入转为数字。`<input v-model.number="num" type="number">`
- .trim: 自动过滤首尾的空白字符。`<input v-model.trim="msg">`

# 二. 计算属性和侦听器

## 1. 计算属性 computed
```js
let vm = new Vue({
  el: '#app',
  data: {
    message: 'hello',
    firstName: 'wt',
    lastName: 'liu'
  },
  computed: {
    reversedMessage: function () {
      return this.message.split('').reverse().join('')
    },
    fullName: {
      get: function() {
        return this.firstName + ' ' +this.lastName;
      },
      set: function(val) {
        [this.firstName, this.lastName] = val.split(' ');
      }
    }
  }
})
```
这里的 computed 中的 reversedMessage 就是利用 message 属性计算而来的属性,可以直接在组件中访问到。

**如果 message 属性不变，那么是不会重新计算 reversedMessage 属性，如果使用 methods 来计算 reversedMessage，那么每次都会调用方法，不论 message 是否改变**

计算属性默认传递的是 getter 函数，但是也可以指定 setter 函数，就像 fullName 属性一样

## 2. 侦听属性 watch
```js
let vm = new Vue({
  el: '#app',
  data: {
    message: 'hello',
    reversedMessage: 'olleh',
  },
  watch: {
    message: function (newVal, oldVal) {
      this.reversedMessage = val.split('').reverse().join('')
    }
  }
})
```
watch 属性中定义的就是监听属性，这里监听了 message 属性，当 message 属性改变，那么就执行监听函数(接受新旧属性，两个参数)，从而计算新的 reversedMessage

**能使用计算属性就尽量使用计算属性，计算属性不满足需求，再使用侦听属性**

### 侦听属性的正确应用场景
**当数据变化时需要执行异步操作或者开销较大的操作，那么应当使用 watch 侦听属性**
```js
let watchExampleVM = new Vue({
  el: '#watch-example',
  data: {
    question: '',
    answer: 'I cannot give you an answer until you ask a question!'
  },
  watch: {
    question: function (newQuestion, oldQuestion) {
      this.answer = 'Waiting for you to stop typing...'
      this.debouncedGetAnswer()
    }
  },
  created: function () {
    this.debouncedGetAnswer = _.debounce(this.getAnswer, 500)
  },
  methods: {
    getAnswer: function () {
      this.answer = 'Thinking...'
      let vm = this
      axios.get('https://yesno.wtf/api')
        .then(function (response) {
          vm.answer = _.capitalize(response.data.answer)
        })
        .catch(function (error) {
          vm.answer = 'Error! Could not reach the API. ' + error
        })
    }
  }
})
```

# 三. 绑定 class 和 style

- 如果 class 和 style 是不依赖变量的定值，那么不需要 v-bind，定义时直接指定就好了。
- 对于依赖变量的 class 和 style 值，需要使用 v-bind:class v-bind:style，简写的话就是 :class :style
```html
<div class="container"  style="color: red;"></div>
```

## 1. 绑定 class
绑定 class 主要通过 v-bind:class=CLASS 实现。CLASS 是字符串，这个字符串可以是 类名 也可以是 对象形式 或者数组形式

- 类名：`<div v-bind:class="className"></div>` 表示 div 的类名由 className 变量决定
- 对象：`<div v-bind:class="{ active: isActive }"></div>` 表示 div 的 class 是否是 active 取决于变量 isActive 是否为真值（除了 +0 -0 0 NaN false '' null undefined，剩下都是 true）
- 数组：`<div v-bind:class="[{ active: isActive }, errorClass]"></div>` 表示 div 的 class 由 errorClass 变量决定，并且是否包含 active 类，由 isActive 变量是否为真值决定

## 2. 绑定 style
绑定 style 主要通过 v-bind:style=STYLE 实现。STYLE 是可以是 对象形式 或者数组形式。
Vue 会自动添加浏览器前缀。

- 对象形式：
```html
<div v-bind:style="{color: activeColor, fontSize: fontSize + 'px'}"></div>

<!-- 直接传入代表样式对象的变量名 -->
<div v-bind:style="styleObject"></div>
data:{
  styleObject: {
    color: 'red',
    fontSize: '13px'
  }
}
```
- 数组：
```html
<!-- 表示将多个样式对象应用于这个元素 -->
<div v-bind:style="[baseStyles, overridingStyles]"></div>
```

# 四. 组件

## 注册组件
组件分为全局组件和局部组件。

### 全局组件：
```js
Vue.component('counter', {
  data: function() {
    return {
      count: 0,
    }
  },
  template: '<button @click="count++">{{ count }}</button>'
})

// 使用组件
<counter></counter>
```
- 使用组件之前必须先注册组件
- component 和 new Vue 接受的属性基本一致，主要区别在于，component 没有 el 根元素；component 的 data 属性必须是函数，并在函数中返回状态对象，避免多个组件共享一个状态对象。

## 向组件传递数据
**注册 component 时传递 props 数组**
传递数据：
- 对于静态数据，使用时直接指定 ATTR
- 对于变化的数据，使用 v-bind 将数据绑定到组件上

```html
<!-- 方式1 -->
<blog-post title="My journey with Vue"></blog-post>
<blog-post title="Blogging with Vue"></blog-post>
<blog-post title="Why Vue is so fun"></blog-post>

<!-- 方式二 -->
<blog-post
  v-for="post in posts"
  v-bind:key="post.id"
  v-bind:title="post.title"
></blog-post>

<script>
Vue.component('blog-post', {
  props: ['title'],
  template: '<h3>{{ title }}</h3>'
})
</script>
```

## 监听子组件事件
- 通过 v-on:ENT="EXP" 来监听自定义事件 ENT；
- 通过 $emit("ENT", param) 方法来触发事件，并传递参数 param