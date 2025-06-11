---
title: Vue3响应式基础
tags: [vue3]
categories: [vue]
date: 2025-04-24 14:49:42
---

## 声明响应式状态

在组合式 API 中，推荐使用 ref() 函数来声明响应式状态。**ref() 接收参数，并将其包裹在一个带有 .value 属性的 ref 对象中返回，所以在访问ref对象的值时需要通过 `.value` 属性访问**。要在组件模板中访问 ref，需要从组件的 setup() 函数中声明并返回它们。**注意，在模板中使用 ref 时，不需要附加 .value。这是因为为了方便起见，当在模板中使用时，ref 会自动解包** 。

```js
import { ref } from 'vue'

export default {
  setup() {
    const count = ref(0)

    function increment() {
      // 在 JavaScript 中需要 .value
      count.value++
    }

    // 不要忘记同时暴露 increment 函数
    return {
      count,
      increment
    }
  }
}
```

```vue
<button @click="increment">
  {{ count }}
</button>
```

### `<script setup>`
在 setup() 函数中手动暴露大量的状态和方法非常繁琐。所以框架提供了 `<script setup>` 来简化代码：

```vue
<script setup>
import { ref } from 'vue'

const count = ref(0)

function increment() {
  count.value++
}
</script>

<template>
  <button @click="increment">
    {{ count }}
  </button>
</template>
```

`<script setup>` 中的顶层的导入、声明的变量和函数可在同一组件的模板中直接使用。可以简单理解为它们是在模板同一作用域内声明的。

### 为什么需要使用带有 .value 的 ref？

为了实现响应式数据发生变化时，能自动检测到这个变化，并且相应地更新 DOM。vue要在组件首次渲染时追踪渲染过程中使用的每一个ref。然后，当一个 ref 被修改时，触发追踪它的组件的一次重新渲染。

在标准的 JavaScript 中，检测普通变量的访问或修改是行不通的。不过，可以通过 ref 的 getter 和 setter 方法来拦截对象属性的 get 和 set 操作。

这就给了 Vue 一个机会来检测 ref 何时被访问或修改。在其内部，Vue 在它的 getter 中执行追踪，在它的 setter 中执行触发。从概念上讲，可以将 ref 看作是一个像这样的对象：

```js
// 伪代码，不是真正的实现
const myRef = {
  _value: 0,
  get value() {
    track()
    return this._value
  },
  set value(newValue) {
    this._value = newValue
    trigger()
  }
}
```

### 深层响应性

Ref 可以持有任何类型的值，包括深层嵌套的对象、数组或者 JavaScript 内置的数据结构。

Ref 会使它的值具有深层响应性。这意味着即使改变嵌套对象或数组时，变化也会被检测到：

```js
import { ref } from 'vue'

const obj = ref({
  nested: { count: 0 },
  arr: ['foo', 'bar']
})

function mutateDeeply() {
  // 以下都会按照期望工作
  obj.value.nested.count++
  obj.value.arr.push('baz')
}
```

也可以通过 shallowRef() 来放弃深层响应性。对于浅层 ref，只有 .value 的访问会被追踪。**浅层 ref 可以用于避免对大型数据的响应性开销来优化性能、或者有外部库管理其内部状态的情况**。

### DOM 更新时机

修改了响应式状态时，DOM 会被自动更新。但是**DOM 更新不是同步的**。Vue 会在“next tick”更新周期中缓冲所有状态的修改，以确保不管进行了多少次状态修改，每个组件都只会被更新一次。

如果要等待 DOM 更新完成后再执行额外的代码，可以使用 nextTick() 全局 API：

```js
import { nextTick } from 'vue'

async function increment() {
  count.value++
  await nextTick()
  // 现在 DOM 已经更新了
}
```

## `reactive()`

另一种声明响应式状态的方式，即使用 `reactive()` API。与将内部值包装在特殊对象中的 ref 不同，`reactive()` 将使对象本身具有响应性。

```js
import { reactive } from 'vue'

const state = reactive({ count: 0 })
```

在模板中使用：

```vue
<button @click="state.count++">
  {{ state.count }}
</button>
```

响应式对象是 **JavaScript 代理**，其行为就和普通对象一样。不同的是，Vue 能够拦截对响应式对象所有属性的访问和修改，以便进行依赖追踪和触发更新。

reactive() 将深层地转换对象：当访问嵌套对象时，它们也会被 reactive() 包装。当 ref 的值是一个对象时，ref() 也会在内部调用它。**与浅层 ref 类似，这里也有一个 shallowReactive() API 可以选择退出深层响应性。**

### 响应式代理对象 vs. 原始对象

reactive() 返回的是一个原始对象的 Proxy，它和原始对象是不相等的。只有代理对象是响应式的，更改原始对象不会触发更新。因此，使用 Vue 的响应式系统的**最佳实践是仅使用你声明对象的代理版本**。

**为保证访问代理的一致性，对同一个原始对象调用 reactive() 会总是返回同样的代理对象，而对一个已存在的代理对象调用 reactive() 会返回其本身**：

```js
const raw = {}
const proxy = reactive(raw)

// 代理对象和原始对象不是全等的
console.log(proxy === raw) // false

// 在同一个对象上调用 reactive() 会返回相同的代理
console.log(reactive(raw) === proxy) // true

// 在一个代理上调用 reactive() 会返回它自己
console.log(reactive(proxy) === proxy) // true
```

依靠深层响应性，响应式对象内的嵌套对象依然是代理，因此响应式对象的嵌套对象和嵌套对象的原始对象也不相等。

```js
const proxy = reactive({})

const raw = {}
proxy.nested = raw

console.log(proxy.nested === raw) // false
```

### reactive() 的局限性

1. **有限的值类型**：它只能用于对象类型 (对象、数组和如 Map、Set 这样的集合类型)。

2. **不能替换整个对象**：由于 Vue 的响应式跟踪是通过属性访问实现的，因此我们必须始终保持对响应式对象的相同引用。这意味着我们不能地“替换”响应式对象，因为这样的话与第一个引用的响应性连接将丢失。

3. **对解构操作不友好**：当我们将响应式对象的原始类型属性解构为本地变量时，或者将该属性传递给函数时，我们将丢失响应性连接。

由于这些限制，**建议使用 ref() 作为声明响应式状态的主要 API**。

## 额外的 ref 解包细节

### ref 会在作为响应式对象的属性被访问或修改时自动解包

```js
const count = ref(0)
const state = reactive({
  count
})

console.log(state.count) // 0

state.count = 1
console.log(count.value) // 1
```

如果将一个新的 ref 赋值给一个关联了已有 ref 的属性，那么它会替换掉旧的 ref：

```js
const otherCount = ref(2)

state.count = otherCount
console.log(state.count) // 2
// 原始 ref 现在已经和 state.count 失去联系
console.log(count.value) // 1
```

**只有当嵌套在一个深层响应式对象内时，才会发生 ref 解包**。当其作为浅层响应式对象的属性被访问时不会解包。

## 当 ref 作为响应式数组或原生集合类型 (如 Map) 中的元素被访问时，它不会被解包

```js
const books = reactive([ref('Vue 3 Guide')])
// 这里需要 .value
console.log(books[0].value)

const map = reactive(new Map([['count', ref(0)]]))
// 这里需要 .value
console.log(map.get('count').value)
```

### 在模板中解包的注意事项

#### 在模板渲染上下文中，只有顶级的 ref 属性才会被解包

在下面的例子中，count 和 object 是顶级属性，但 object.id 不是：

```js
const count = ref(0)
const object = { id: ref(1) }
```

因此，这个表达式将不会按预期工作：

```vue
{{ object.id + 1 }}
```

渲染的结果将是 [object Object]1，因为在计算表达式时 object.id 没有被解包，仍然是一个 ref 对象。为了解决这个问题，我们可以将 id 解构为一个顶级属性：

```js
const { id } = object
```

#### 如果 ref 是文本插值的最终计算值，那么它将被解包

```vue
<!-- 以下内容将渲染为 1： -->
{{ object.id }}
```