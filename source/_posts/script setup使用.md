---
title: script setup
tags: ["vue3"]
categories: ["vue"]
date: 2025-04-21 18:24:45
---

`<script setup>` 是在单文件组件中使用组合式 API 的编译时语法糖。相比于普通的` <script> `语法，它具有更多优势：

- 更少的样板内容，更简洁的代码。
- 能够使用纯 TypeScript 声明 props 和自定义事件。
- 更好的运行时性能 (其模板会被编译成同一作用域内的渲染函数，避免了渲染上下文代理对象)。
- 更好的 IDE 类型推导性能 (减少了语言服务器从代码中抽取类型的工作)。

## 基本语法：

要启用setup语法，只需要在` <script> `代码块上添加 `setup attribute`。顶层的变量和函数等会被暴露给模版，import 导入的内容也会暴露。

```vue
<script setup>
import { capitalize } from './helpers'

// 变量
const msg = 'Hello!'

// 函数
function log() {
  console.log(msg)
}
</script>

<template>
  <div>{{ capitalize('hello') }}</div>
  <button @click="log">{{ msg }}</button>
</template>
```

## 使用组件

`<script setup> `里的导入的组件可以直接作为自定义组件的标签名使用：

```vue
<script setup>
import MyComponent from './MyComponent.vue'
</script>

<template>
  <MyComponent />
</template>
```

其 kebab-case 格式的 <my-component> 同样能在模板中使用。不过，官方强烈建议使用 PascalCase 格式以保持一致性。同时这也有助于区分原生的自定义元素。

### 动态组件

由于组件是通过变量引用而不是基于字符串组件名注册的，在 `<script setup>` 中要使用动态组件的时候，应该使用动态的 :is 来绑定：

```vue
<script setup>
import Foo from './Foo.vue'
import Bar from './Bar.vue'
</script>

<template>
  <component :is="Foo" />
  <component :is="someCondition ? Foo : Bar" />
</template>
```

### 递归组件

一个单文件组件可以通过它的文件名被其自己所引用。例如：名为 FooBar.vue 的组件可以在其模板中用 <FooBar/> 引用它自己。

*这种方式相比于导入的组件优先级更低。*如果有具名的导入和组件自身推导的名字冲突了，可以为导入的组件添加别名

```vue
import { FooBar as FooBarChild } from './components'
```

## 使用自定义指令

本地的自定义指令在 `<script setup>` 中不需要显式注册，但他们必须遵循 vNameOfDirective 这样的命名规范：

```vue
<script setup>
const vMyDirective = {
  beforeMount: (el) => {
    // 在元素上做些操作
  }
}
</script>
<template>
  <h1 v-my-directive>This is a Heading</h1>
</template>
```

## defineProps() 和 defineEmits()

为了在声明 props 和 emits 选项时获得完整的类型推导支持，我们可以使用 defineProps 和 defineEmits API，它们自动地在 `<script setup>` 中可用：

```vue
<script setup>
const props = defineProps({
  foo: String
})

const emit = defineEmits(['change', 'delete'])
// setup 代码
</script>
```

- `defineProps` 和 `defineEmits` 都是只能在 `<script setup>` 中使用的编译器宏。他们不需要导入，且会随着 `<script setup>` 的处理过程一同被编译掉。

- `defineProps` 和 `defineEmits` 在选项传入后，会提供恰当的类型推导。

- **注意：传入的选项不能引用在 setup 作用域中声明的局部变量。因为传入到 `defineProps` 和 `defineEmits` 的选项会在编译阶段从 setup 中提升到模块的作用域（整个文件的顶层作用域），这样做会引起编译错误。** 但是，它可以引用导入的绑定，因为导入的绑定也在模块作用域内。

```vue

<script setup>
const localValue = "hello";
import { someConstant } from './constants';

defineProps({
  msg: {
    type: String,
    // 错误！因为选项被提升到了模块作用域，无法访问 setup 内的局部变量
    default: localValue
  },
  otherProp: {
    type: String,
    // 正确！导入的绑定位于模块作用域，可以被引用
    default: someConstant
  }

});
</script>
```

## 参考

- [vue3官方文档——单文件组件<script setup>](https://cn.vuejs.org/api/sfc-script-setup.html)

