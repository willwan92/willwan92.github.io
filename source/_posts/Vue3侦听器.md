---
title: Vue3侦听器
tags: [Vue3]
categories: [vue]
date: 2025-04-25 15:37:41
---

在组合式 API 中，可以使用 watch 函数在每次响应式状态发生变化时触发回调函数。

## 侦听数据源类型

watch 的第一个参数可以是不同形式的“数据源”：它可以是一个 ref (包括计算属性)、一个响应式对象、一个 getter 函数、或多个数据源组成的数组：

```js
const x = ref(0)
const y = ref(0)

// 单个 ref
watch(x, (newX) => {
  console.log(`x is ${newX}`)
})

// getter 函数
watch(
  () => x.value + y.value,
  (sum) => {
    console.log(`sum of x + y is: ${sum}`)
  }
)

// 多个来源组成的数组
watch([x, () => y.value], ([newX, newY]) => {
  console.log(`x is ${newX} and y is ${newY}`)
})
```

**注意，你不能直接侦听响应式对象的属性值，需要用一个返回该属性的 getter 函数**

```js
const obj = reactive({ count: 0 })

// 错误，因为 watch() 得到的参数是一个 number
watch(obj.count, (count) => {
  console.log(`Count is: ${count}`)
})

// 提供一个 getter 函数
watch(
  () => obj.count,
  (count) => {
    console.log(`Count is: ${count}`)
  }
)
```

## 深层侦听器

直接给 watch() 传入一个响应式对象，会隐式地创建一个深层侦听器——该回调函数在所有嵌套的变更时都会被触发：

```js
const obj = reactive({ count: 0 })

watch(obj, (newValue, oldValue) => {
  // 在嵌套的属性变更时触发
  // 注意：`newValue` 此处和 `oldValue` 是相等的
  // 因为它们是同一个对象！
})

obj.count++
```

监听一个返回响应式对象的 getter 函数，只有在返回不同的对象时，才会触发回调：

```js
watch(
  () => state.someObject,
  () => {
    // 仅当 state.someObject 被替换时触发
  }
)
```

也可以给上面这个例子显式地加上 `deep` 选项，强制转成深层侦听器：

```js
watch(
  () => state.someObject,
  (newValue, oldValue) => {
    // 注意：`newValue` 此处和 `oldValue` 是相等的
    // *除非* state.someObject 被整个替换了
  },
  { deep: true }
)
```

在 Vue 3.5+ 中，deep 选项还可以是一个数字，表示最大遍历深度。

## 即时回调的侦听器

watch 默认是懒执行的：仅当数据源变化时，才会执行回调。可以通过传入 `immediate: true` 选项来强制侦听器的回调立即执行：

```js
watch(
  source,
  (newValue, oldValue) => {
    // 立即执行，且当 `source` 改变时再次执行
  },
  { immediate: true }
)
```

## 一次性侦听器

如果希望回调只在源变化时触发一次，请使用 `once: true` 选项。

```js
watch(
  source,
  (newValue, oldValue) => {
    // 当 `source` 变化时，仅触发一次
  },
  { once: true }
)
```

## watchEffect()

`watchEffect` 和 `watch` 都能响应式地执行有副作用的回调。

例如下面的代码，在每当 todoId 的引用发生变化时使用侦听器来加载一个远程资源：

```js
const todoId = ref(1)
const data = ref(null)

watch(
  todoId,
  async () => {
    const response = await fetch(
      `https://jsonplaceholder.typicode.com/todos/${todoId.value}`
    )
    data.value = await response.json()
  },
  { immediate: true }
)
```

像上面例子中这样如果侦听器的回调中使用与源完全相同的响应式状态，就可以用 watchEffect 函数 来简化上面的代码。

```js
watchEffect(async () => {
  const response = await fetch(
    `https://jsonplaceholder.typicode.com/todos/${todoId.value}`
  )
  data.value = await response.json()
})
```

### watch vs. watchEffect

watch 和 watchEffect 之间的主要区别是追踪响应式依赖的方式：

- watch 只追踪明确侦听的数据源。它不会追踪任何在回调中访问到的东西。另外，仅在数据源确实改变时才会触发回调。watch 会避免在发生副作用时追踪依赖，因此，我们能更加精确地控制回调函数的触发时机。

- watchEffect，则会在副作用发生期间追踪依赖。它会在同步执行过程中，自动追踪所有能访问到的响应式属性。这更方便，而且代码往往更简洁，但有时其响应性依赖关系会不那么明确。

## 副作用清理

当我们可能会在侦听器中执行副作用时，例如异步请求，如果在请求完成之前监听的数据源发生了变化，当上一个请求完成时，它仍会使用已经过时的数据源值触发回调。理想情况下，我们希望能够在数据源变为新值时取消过时的请求。

1. 可以使用` onWatcherCleanup() ` API 来注册一个清理函数清理副作用：

```js
import { watch, onWatcherCleanup } from 'vue'

watch(id, (newId) => {
  const controller = new AbortController()

  fetch(`/api/${newId}`, { signal: controller.signal }).then(() => {
    // 回调逻辑
  })

  onWatcherCleanup(() => {
    // 终止过期请求
    controller.abort()
  })
})
```

**注意，onWatcherCleanup 仅在 Vue 3.5+ 中支持，并且必须在 watchEffect 效果函数或 watch 回调函数的同步执行期间调用：你不能在异步函数的 await 语句之后调用它。**

2. 在 Vue 3.5 之前的版本，可以使用 onCleanup 函数清理副作用。

onCleanup 函数需要作为第三个参数传递给侦听器 watch 的回调函数，或 watchEffect 回调函数的第一个参数：

```js
watch(id, (newId, oldId, onCleanup) => {
  // ...
  onCleanup(() => {
    // 清理逻辑
  })
})

watchEffect((onCleanup) => {
  // ...
  onCleanup(() => {
    // 清理逻辑
  })
})
```

通过函数参数传递的 onCleanup 与侦听器实例相绑定，因此不受 onWatcherCleanup 的同步限制。

## 侦听器回调的触发时机

类似于组件更新，侦听器回调函数也会被批量处理以避免重复调用。默认情况下，**侦听器回调会在父组件更新 (如有) 之后、所属组件的 DOM 更新之前被调用**。

如果想在侦听器回调中能访问被 Vue 更新之后的所属组件的 DOM，需要指明 flush: 'post' 选项：

```js
watch(source, callback, {
  flush: 'post'
})

watchEffect(callback, {
  flush: 'post'
})
```

### 后置侦听器 watchPostEffect()

后置刷新的 watchEffect() 有个更方便的别名 watchPostEffect()：

```js
import { watchPostEffect } from 'vue'

watchPostEffect(() => {
  /* 在 Vue 更新后执行 */
})
```

### 同步侦听器

还可以创建一个同步触发的侦听器，它会在 Vue 进行任何更新之前触发：

```js
watch(source, callback, {
  flush: 'sync'
})

watchEffect(callback, {
  flush: 'sync'
})
```

同步触发的 watchEffect() 也有个更方便的别名 watchSyncEffect()：

```js
import { watchSyncEffect } from 'vue'

watchSyncEffect(() => {
  /* 在响应式数据变化时同步执行 */
})
```

**同步侦听器不会进行批处理，每当检测到响应式数据发生变化时就会触发。可以使用它来监视简单的布尔值，但应避免在可能多次同步修改的数据源 (如数组) 上使用。**


## 停止侦听器

在 setup() 或 `<script setup>` 中用同步语句创建的侦听器，会自动绑定到宿主组件实例上，并且会在宿主组件卸载时自动停止。

侦听器必须用同步语句创建：如果用异步回调创建一个侦听器，那么它不会绑定到当前组件上，必须手动停止它，以防内存泄漏。

```js
<script setup>
import { watchEffect } from 'vue'

// 它会自动停止
watchEffect(() => {})

// ...这个则不会！
setTimeout(() => {
  watchEffect(() => {})
}, 100)
</script>
```

要手动停止一个侦听器，请调用 watch 或 watchEffect 返回的函数：

```js
const unwatch = watchEffect(() => {})

// ...当该侦听器不再需要时
unwatch()
```

**注意，需要异步创建侦听器的情况很少，请尽可能选择同步创建。**
