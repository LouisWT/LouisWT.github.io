# 一. 组件注册

## 组件名
组件注册的时候，第一个参数就是组件名，之后就可以利用这个组件名来重用组件了。
```js
Vue.component('my-component', ...)
```
- DOM 中直接使用的组件，组件名必须是小写，**并且必须包含一个短横线**，比如`my-component`，这是由于 HTML 对自定义组件名的规范
- 字符串模板或者单文件组件中，组件名还可以使用 首字母大写命名，比如 `MyComponent`

## 全局注册与局部注册
使用 webpack 编译，全局注册的组件的组件永远会在构建结果中，即便这个组件已经没用到了

### 全局注册
```js
Vue.component('MyComponent', ...)
```
### 局部注册
```js
let componentA = {
  // 使用普通对象定义组件
}

let componentB = {
  // ...
  // 给组件B注册组件A，如果没有这步的话，它无法使用组件A
  components: {
    componentA,
  }
}

new Vue({
  el: '#app',
  // 给应用注册组件A 和 组件B
  components: {
    componentA,
    componentB,
  }
})
```
### 最佳实践

- 正常情况下，对大部分组件使用局部注册，也就是说当前组件用到了哪些别的组件就在 components 中声明这些组件
- 如果有一些基础组件被许多组件公用，那么可以对这些 **基础组件** 进行全局注册

自动化注册 components 文件夹下的 Base 开头的组件：
```js
import Vue from 'vue'
import upperFirst from 'lodash/upperFirst'
import camelCase from 'lodash/camelCase'

const requireComponent = require.context(
  // 其组件目录的相对路径
  './components',
  // 是否查询其子目录
  false,
  // 匹配基础组件文件名的正则表达式
  /Base[A-Z]\w+\.(vue|js)$/
)

requireComponent.keys().forEach(fileName => {
  // 获取组件配置
  const componentConfig = requireComponent(fileName)

  // 获取组件的 PascalCase 命名
  const componentName = upperFirst(
    camelCase(
      // 获取和目录深度无关的文件名
      fileName
        .split('/')
        .pop()
        .replace(/\.\w+$/, '')
    )
  )

  // 全局注册组件
  Vue.component(
    componentName,
    // 如果这个组件选项是通过 `export default` 导出的，
    // 那么就会优先使用 `.default`，
    // 否则回退到使用模块的根。
    componentConfig.default || componentConfig
  )
})
```
# 二. Prop

## Prop 名
与组件名一样，分为 DOM 和 template 两种情况。
- DOM 中，向元素传递 prop，需要使用 小写加短横线 的命名方式
- 字符串模板或者单文件组件中，可以使用驼峰命名法

```js
Vue.component('blog-post', {
  // 在 JavaScript 中是 camelCase 的
  props: ['postTitle'],
  template: '<h3>{{ postTitle }}</h3>'
})

// 如果在 DOM 中使用
<blog-post post-title="hello!"></blog-post>

// 如果在 template 中使用
<blog-post postTitle="hello!"></blog-post>
```
## Prop 类型
传入 Prop 时可以传入数组或对象。
数组形式的话，每个值就是变量名称；
对象形式的话，每个键值对时变量的名称和变量的类型，从而进行参数验证。

### 类型检测
prop 类型检测需要传入对象，对象的键是 prop 名，值可以是**字符串/数组/对象**
- 字符串：prop 的单一类型，比如 Number
- 数组：prop 的多个可能类型，比如 [Number, String]
- 对象：
  - type 传入 props 的类型
  - required：为 true 表示 prop 必须
  - default：表示 prop 为空时的默认值，如果要返回对象的话，要使用函数返回新对象，否则所有组件会共享这个初始默认对象
  - validator：自定义类型检测函数
```js
Vue.component('my-component', {
  props: {
    // 基础的类型检查 (`null` 和 `undefined` 会通过任何类型验证)
    propA: Number,
    // 多个可能的类型
    propB: [String, Number],
    // 必填的字符串
    propC: {
      type: String,
      required: true
    },
    // 带有默认值的数字
    propD: {
      type: Number,
      default: 100
    },
    // 带有默认值的对象
    propE: {
      type: Object,
      // 对象或数组默认值必须从一个工厂函数获取
      default: function () {
        return { message: 'hello' }
      }
    },
    // 自定义验证函数
    propF: {
      validator: function (value) {
        // 这个值必须匹配下列字符串中的一个
        return ['success', 'warning', 'danger'].indexOf(value) !== -1
      }
    }
  }
})
```

### type 类型
- Number
- Boolean
- String
- Symbol
- Object
- Array
- Date
- Function
- 某个构造函数，比如 Promise，Vue 会用 instanceof 来检查


## Prop 传递
```html
<!-- 传递静态字符串 -->
<blog-post title="My journey with Vue"></blog-post>

<!-- 传递变量 -->
<blog-post v-bind:title="post.title"></blog-post>

<!-- 传递数字，尽管是静态的值，但是也需要 v-bind 来说明这是JS表达式不是字符串 -->
<blog-post v-bind:likes="42"></blog-post>

<!-- 传递 Boolean -->
<blog-post v-bind:isPublished="true"></blog-post>

<!-- 传递数组 -->
<blog-post v-bind:commentIds="[123, 456, 789]"></blog-post>

<!-- 传递对象 -->
<blog-post v-bind:author="{name: 'louiswt', age: 24}"></blog-post>

<!-- 将对象的所有属性作为 props 传入 -->
<!--
post: {
  id: 1,
  title: 'My Journey with Vue'
}
-->
<blog-post v-bind="post"></blog-post>
<!-- 相当于 -->
<blog-post v-bind:id="post.id" v-bind:title="post.title"></blog-post>
```

## 特性继承
组件可以接受任意的特性，这些特性会被添加到这个组件的根元素上。
```html
<!-- 给 bootstrap-date-input 添加一个 data-date-picker 的 attr -->
<bootstrap-date-input data-date-picker="activated"></bootstrap-date-input>

<!-- 假设 bootstrap-date-input 的模板如下  -->
<input type="date" class="form-control">

<!-- 那么 data-date-picker 属性最终会自动添加到 bootstrap-date-input 的根元素，也就是 input 元素，相当于 -->
<input type="date" class="form-control" data-date-picker="activated">
```

- 禁用特性继承：在组件选项中设置 `inheritAttrs: false`
- 将特性指定给特定元素，而不是根元素：`v-bind='$attrs'`

手动决定非 prop 的特性会被赋予给哪个元素：
```html
<script>
// 利用 $attrs 将非 label 和 value 的 attr 都绑定给 input 元素
// $attrs 其实就是一个对象，对象的键是 attr name ，值是 attr value
// 绑定的语法是绑定对象的所有属性语法
Vue.component('base-input', {
  inheritAttrs: false,
  props: ['label', 'value'],
  template: `
    <label>
      {{ label }}
      <input
        v-bind="$attrs"
        v-bind:value="value"
        v-on:input="$emit('input', $event.target.value)"
      >
    </label>
  `
})
</script>

<!-- 使用方式 -->
<!-- $attrs = { required: true, placeholder: 'Enter your username' } -->
<!-- required 和 placeholder 属性就会被传给 组件内部的 input 元素 -->
<base-input
  v-model="username"
  required
  placeholder="Enter your username"
></base-input>
```
# 三. 自定义事件

Vue支持自定义事件。**事件名尽量使用 kebab-case （小写加短横线）的事件名**。

## 自定义组件的 v-model

**v-model 绑定的变量只能属于当前组件的 data，不能是父组件传过来的 prop**

### 1. 组件的 v-model 原理
**父组件使用子组件时，如果在 v-model 上绑定了变量 data，那么这个变量 data 默认会利用 `value` prop 传入子组件，
并且父组件默认会监听子组件的 `input` 事件，从而在变量 data 被修改后改变变量 data 的值**。

子组件需要完成对 value prop 的绑定并且在监听到 input 事件后，将事件和 新的 value 抛出。

子组件在整个过程中并没有修改父组件给的值。
```html
<!-- base-input 组件 -->
<template>
  <input
    type="text"
    v-bind="$attrs"
    v-bind:value="value"
    @input="handleOnInput"
  >
</template>

<script>
export default {
  name: 'BaseInput',
  // // v-model 绑定的变量会默认使用 value 作为变量名传入
  props: {
    value: {
      type: String,
      required: true,
    }
  },
  data() {
    return {
      text: this.value,
    }
  },
  methods: {
    handleOnInput(event) {
      // 同步组件内部的状态
      this.text = event.target.value
      // 抛出事件和新的值，从而更新父组件的状态
      this.$emit('input', event.target.value)
    }
  }
}
</script>

<!-- BaseInput 使用方法 -->
<!-- 将 textValue 作为 value props 传入 -->
<base-input
  v-model="textValue"
  required
  placeholder="Enter your username"
></base-input>
```
### 2. v-model 修改传入子组件中的变量名或父组件监听的事件
子组件默认会将得到的 v-model 变量作为 value prop 传入，如果想作为其他 prop 名传入，可以修改子组件的 model 属性；

父组件默认会监听子组件的 input 事件，如果需要监听其他事件，也可以修改 子组件的 model 属性。

```html
<!-- base-box 组件 -->
<template>
  <label for="checkbox">
    <!-- 监听 change 事件并抛出事件和新的值，这是和 model 属性中的设置对应的 -->
    <input
    id="checkbox"
    type="checkbox"
    v-bind:checked="check"
    v-on:change="$emit('onChange', $event.target.checked)"
    >
    这是单选框
  </label>
</template>

<script>
export default {
  name: 'base-box',
  props: {
    // 无论如何必须声明 prop
    check: Boolean
  },
  model: {
    // 将传入的 v-model 作为 check prop 而不是 value prop
    prop: 'check',
    // 父组件监听子组件的 onChange 事件而不是 input 事件
    // 这个 onChange 事件是自定义的，只要和子组件 $emit 的事件名对应就行
    event: 'onChange'
  },
}
</script>

<!-- base-box 使用方式 -->
<!-- 将 lovingVue 作为 v-model 绑定到 base-box 组件 -->
<base-box v-model="lovingVue"></base-box>
```

1.  子组件在 props 和 model 的 prop 属性中，指定传入的 v-model 的变量名
2.  子组件在 model 的 event 属性中，指定父组件监听自己的什么事件
3.  子组件绑定父组件给的 prop
4.  子组件在特定情况下发出事件，并给出改变的值
5.  父组件监听事件并得到新的值

## 对 prop 进行双向绑定
双向绑定就是说，子组件修改父组件的 prop 之后，父组件的状态也会被修改，单纯的双向绑定不符合单向数据流的理念，所以可以使用 `update:propName`自定义事件来实现
```html
<!-- Title 组件，在点击文本后， 会 $emit update:title 事件和新的 title -->
<template>
  <div @click="handleOnClick">Click Me</div>
</template>

<script>
export default {
  name: "tittle",
  props: {
    title: {
      type: String,
      required: true,
    }
  },
  data() {
    return {
      count: 0,
    }
  },
  methods: {
    handleOnClick() {
      this.count++;
      this.$emit('update:title', this.title + this.count)
    }
  }
}
</script>

<!-- 使用方式 -->
<Title
  :title="myTitle"
  @update:title="myTitle = $event"
></Title>
<!-- 或 -->
<Title :title.sync="myTitle"></Title>
```
事件名可以是任意事件名，并不需要一定是 `update:propName` 形式，只要子组件 $emit 的事件名和 使用时监听的事件名相同即可。
但是如果要使用 .sync 修饰符来自动绑定的话，那么就需要使用 `update:propName` 形式。

综合来说，用 `update:propName` 形式更规范一点。

## 监听子组件的原生事件
从上面可以看到，父组件实际监听的 input 事件实际上是子组件再次 $emit 而来的事件，得到的是子组件传过来的参数，而不是原生的事件。

如果想监听原生事件，那么可以使用 .native 修饰符，加上这个修饰符后，父组件会监听子组件根元素的对应事件，并且触发后会得到原生的 event 对象

如果原生事件不是发生在子组件的根元素，那么子组件可以使用 `this.$listeners` 来获得父组件传入的事件处理函数与事件的对象，然后将它绑定到发生事件的元素。

```html
<template>
  <label for="text">
    <input
      id="text"
      type="text"
      v-bind="$attrs"
      v-bind:value="value"
      v-on="inputListeners"
    >
  </label>
</template>

<script>
export default {
  name: 'BaseInput',
  // v-model 绑定的变量会默认使用 value 作为变量名传入
  props: {
    value: {
      type: String,
      required: true,
    }
  },
  data() {
    return {
      text: this.value,
    }
  },
  computed: {
    // 这样写是为了兼容 v-model 的用法
    inputListeners() {
      return {
        ...this.$listeners,
        input: this.handleOnInput,
      }
    }
  },
  methods: {
    handleOnInput(event) {
      this.text = event.target.value
      this.$emit('input', event.target.value)
    }
  }
}
</script>

<!-- 用法 -->
<BaseInput
    @focus.native="onFoucs"
    @input.native="onInput"
    required
    placeholder="Enter your username"
></BaseInput>
```
