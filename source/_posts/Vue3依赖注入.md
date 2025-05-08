---
title: Vue3依赖注入
tags: [Vue3]
categories: [vue]
date: 2025-05-08 17:51:50
---

## 依赖注入的作用

解决 Prop 逐级透传很麻烦的问题。

## Provide（提供）

要为组件后代提供数据，需要使用到 provide() 函数：

```vue
<script setup>
import { provide } from 'vue'

provide(/* 注入名 */ 'message', /* 值 */ 'hello!')
</script>
```

如果不使用 `<script setup>`，请确保 provide() 是在 setup() 同步调用的：

```vue
import { provide } from 'vue'

export default {
  setup() {
    provide(/* 注入名 */ 'message', /* 值 */ 'hello!')
  }
}
```

provide() 函数接收两个参数。第一个参数被称为注入名，可以是一个字符串或是一个 Symbol。后代组件会用注入名来查找期望注入的值。一个组件可以多次调用 provide()，使用不同的注入名，注入不同的依赖值。

第二个参数是提供的值，值可以是任意类型，包括响应式的状态，比如一个 ref：

```vue
import { ref, provide } from 'vue'

const count = ref(0)
provide('key', count)
```

## 应用层 Provide

除了在一个组件中提供依赖，我们还可以在整个应用层面提供依赖：

```vue
import { createApp } from 'vue'

const app = createApp({})

app.provide(/* 注入名 */ 'message', /* 值 */ 'hello!')
```

在应用级别提供的数据在该应用内的所有组件中都可以注入。

## Inject (注入)

要注入上层组件提供的数据，需使用 inject() 函数：

```vue
<script setup>
import { inject } from 'vue'

const message = inject('message')
</script>
```

如果提供的值是一个 ref，注入进来的也是该 ref 对象，保持了和供给方的响应性链接。

同样的，如果没有使用 `<script setup>`，inject() 需要在 setup() 内同步调用：

```vue
import { inject } from 'vue'

export default {
  setup() {
    const message = inject('message')
    return { message }
  }
}
```

## 注入默认值

如果在注入一个值时不要求必须有提供者，那么可以声明一个默认值。

```vue
// 如果没有祖先组件提供 "message"
// `value` 会是 "这是默认值"
const value = inject('message', '这是默认值')
```

默认值可能需要通过调用一个函数或初始化一个类来取得。为了避免在用不到默认值的情况下进行不必要的计算或产生副作用，我们可以使用工厂函数来创建默认值：

```vue
const value = inject('key', () => new ExpensiveClass(), true)
```

第三个参数表示默认值应该被当作一个工厂函数。

## 和响应式数据配合使用

当提供 / 注入响应式的数据时，建议尽可能将任何对响应式状态的变更都保持在供给方组件中。

如果需要在注入方组件中更改数据，我们推荐在供给方组件内声明并提供一个更改数据的方法函数：

```vue
<!-- 在供给方组件内 -->
<script setup>
import { provide, ref } from 'vue'

const location = ref('North Pole')

function updateLocation() {
  location.value = 'South Pole'
}

provide('location', {
  location,
  updateLocation
})
</script>
```

```vue
<!-- 在注入方组件 -->
<script setup>
import { inject } from 'vue'

const { location, updateLocation } = inject('location')
</script>

<template>
  <button @click="updateLocation">{{ location }}</button>
</template>
```

如果你想确保提供的数据不能被注入方的组件更改，你可以使用 readonly() 来包装提供的值。

```vue
<script setup>
import { ref, provide, readonly } from 'vue'

const count = ref(0)
provide('read-only-count', readonly(count))
</script>
```

## 使用 Symbol 作注入名

如果正在构建大型的应用，包含非常多的依赖提供，或者你正在编写提供给其他开发者使用的组件库，建议最好使用 Symbol 来作为注入名以避免潜在的冲突。

推荐在一个单独的文件中导出这些注入名 Symbol：

```vue
// keys.js
export const myInjectionKey = Symbol()
```

```vue
// 在供给方组件中
import { provide } from 'vue'
import { myInjectionKey } from './keys.js'

provide(myInjectionKey, { 
  /* 要提供的数据 */
})
```

```vue
// 注入方组件
import { inject } from 'vue'
import { myInjectionKey } from './keys.js'

const injected = inject(myInjectionKey)
```