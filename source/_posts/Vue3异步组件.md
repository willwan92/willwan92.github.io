---
title: Vue3异步组件
tags: [Vue3]
categories: [vue]
date: 2025-05-09 14:28:02
---

## 

如果需要从服务器加载组件，Vue 提供了 defineAsyncComponent 方法来实现此功能：

```vue
import { defineAsyncComponent } from 'vue'

const AsyncComp = defineAsyncComponent(() => {
  return new Promise((resolve, reject) => {
    // ...从服务器获取组件
    resolve(/* 获取到的组件 */)
  })
})
// ... 像使用其他一般组件一样使用 `AsyncComp`
```

defineAsyncComponent 方法接收一个返回 Promise 的加载函数。这个 Promise 的 resolve 回调方法应该在从服务器获得组件定义时调用。你也可以调用 reject(reason) 表明加载失败。

ES 模块动态导入也会返回一个 Promise，所以多数情况下会将它和 defineAsyncComponent 搭配使用。类似 Vite 和 Webpack 这样的构建工具也支持此语法 (并且会将它们作为打包时的代码分割点)，因此我们也可以用它来导入 Vue 单文件组件：

```vue
import { defineAsyncComponent } from 'vue'

const AsyncComp = defineAsyncComponent(() =>
  import('./components/MyComponent.vue')
)
```

最后得到的 AsyncComp 是一个外层包装过的组件，仅在页面需要它渲染时才会调用加载内部实际组件的函数。它会将接收到的 props 和插槽传给内部组件，所以你可以使用这个异步的包装组件无缝地替换原始组件，同时实现延迟加载。

与普通组件一样，异步组件可以使用 app.component() 全局注册：

```vue
app.component('MyComponent', defineAsyncComponent(() =>
  import('./components/MyComponent.vue')
))
```

也可以直接在父组件中直接定义它们：

```vue
<script setup>
import { defineAsyncComponent } from 'vue'

const AdminPage = defineAsyncComponent(() =>
  import('./components/AdminPageComponent.vue')
)
</script>

<template>
  <AdminPage />
</template>
```

## 加载与错误状态

defineAsyncComponent() 也支持在高级选项中处理这些状态：

```vue
const AsyncComp = defineAsyncComponent({
  // 加载函数
  loader: () => import('./Foo.vue'),

  // 加载异步组件时使用的组件
  loadingComponent: LoadingComponent,
  // 展示加载组件前的延迟时间，默认为 200ms
  delay: 200,

  // 加载失败后展示的组件
  errorComponent: ErrorComponent,
  // 如果提供了一个 timeout 时间限制，并超时了
  // 也会显示这里配置的报错组件，默认值是：Infinity
  timeout: 3000
})
```

## 惰性激活（3.5+）

在使用服务器端渲染，这部分才会适用。在 Vue 3.5+ 中，异步组件可以通过提供激活策略来控制何时进行激活。

- Vue 提供了一些内置的激活策略。这些内置策略需要分别导入，以便在未使用时进行 tree-shake。

- 该设计有意保持在底层，以确保灵活性。将来可以在此基础上构建编译器语法糖，无论是在核心还是更上层的解决方案 (如 Nuxt) 中实现。

### 在空闲时进行激活

通过 requestIdleCallback 进行激活：

```vue
import { defineAsyncComponent, hydrateOnIdle } from 'vue'

const AsyncComp = defineAsyncComponent({
  loader: () => import('./Comp.vue'),
  hydrate: hydrateOnIdle(/* 传递可选的最大超时 */)
})
```

### 在可见时激活

通过 IntersectionObserver 在元素变为可见时进行激活。

```vue
import { defineAsyncComponent, hydrateOnVisible } from 'vue'

hydrateOnVisible({ rootMargin: '100px' })

const AsyncComp = defineAsyncComponent({
  loader: () => import('./Comp.vue'),
  hydrate: hydrateOnVisible()
})

```

### 在媒体查询匹配时进行激活

```vue
import { defineAsyncComponent, hydrateOnMediaQuery } from 'vue'

const AsyncComp = defineAsyncComponent({
  loader: () => import('./Comp.vue'),
  hydrate: hydrateOnMediaQuery('(max-width:500px)')
})
```

### 交互时激活

当组件元素上触发指定事件时进行激活。完成激活后，触发激活的事件也将被重放。

```vue
import { defineAsyncComponent, hydrateOnInteraction } from 'vue'

const AsyncComp = defineAsyncComponent({
  loader: () => import('./Comp.vue'),
  hydrate: hydrateOnInteraction('click')
})
```

激活事件也可以是多个事件类型的列表：

```vue
hydrateOnInteraction(['wheel', 'mouseover'])
```

### 自定义激活策略

```vue
import { defineAsyncComponent, type HydrationStrategy } from 'vue'

const myStrategy: HydrationStrategy = (hydrate, forEachElement) => {
  // forEachElement 是一个遍历组件未激活的 DOM 中所有根元素的辅助函数，
  // 因为根元素可能是一个片段而非单个元素
  forEachElement(el => {
    // ...
  })
  // 准备好时调用 `hydrate`
  hydrate()
  return () => {
    // 如必要，返回一个销毁函数
  }
}

const AsyncComp = defineAsyncComponent({
  loader: () => import('./Comp.vue'),
  hydrate: myStrategy
})
```

## 搭配 Suspense 使用

`<Suspense>` **是一个内置组件，用来在组件树中协调对异步依赖的处理。它让我们可以在组件树上层等待下层的多个嵌套异步依赖项解析完成，并可以在等待时渲染一个加载状态**。

### 异步依赖

要了解 <Suspense> 所解决的问题和它是如何与异步依赖进行交互的，我们需要想象这样一种组件层级结构：

```
<Suspense>
└─ <Dashboard>
   ├─ <Profile>
   │  └─ <FriendStatus>（组件有异步的 setup()）
   └─ <Content>
      ├─ <ActivityFeed> （异步组件）
      └─ <Stats>（异步组件）
```

在这个组件树中有多个嵌套组件，要渲染出它们，首先得解析一些异步资源。如果没有 `<Suspense>`，则它们每个都需要处理自己的加载、报错和完成状态。在最坏的情况下，我们可能会在页面上看到三个旋转的加载态，在不同的时间显示出内容。

有了 `<Suspense>` 组件后，我们就可以在等待整个多层级组件树中的各个异步依赖获取结果时，在顶层展示出加载中或加载失败的状态。

`<Suspense>` 可以等待的异步依赖有两种：

1. **带有异步 setup() 钩子的组件。这也包含了使用 `<script setup>` 时有顶层 await 表达式的组件**。

2. **异步组件**。

### async setup()

组合式 API 中组件的 setup() 钩子可以是异步的：

```vue
export default {
  async setup() {
    const res = await fetch(...)
    const posts = await res.json()
    return {
      posts
    }
  }
}
```

如果使用 `<script setup>`，那么顶层 await 表达式会自动让该组件成为一个异步依赖：

```vue
<script setup>
const res = await fetch(...)
const posts = await res.json()
</script>

<template>
  {{ posts }}
</template>
```

### 异步组件

异步组件默认就是“suspensible”的。这意味着如果组件关系链上有一个 `<Suspense>`，那么这个异步组件就会被当作这个 `<Suspense>` 的一个异步依赖。在这种情况下，加载状态是由 `<Suspense>` 控制，而该组件自己的加载、报错、延时和超时等选项都将被忽略。

异步组件也可以通过在选项中指定 `suspensible: false` 表明不用 Suspense 控制，并让组件始终自己控制其加载状态。

### 加载中状态

`<Suspense>` 组件有两个插槽：#default 和 #fallback。**两个插槽都只允许一个直接子节点**。

```vue
<Suspense>
  <!-- 具有深层异步依赖的组件 -->
  <Dashboard />

  <!-- 在 #fallback 插槽中显示 “正在加载中” -->
  <template #fallback>
    Loading...
  </template>
</Suspense>
```

在初始渲染时，`<Suspense>` 将在内存中渲染其默认的插槽内容。如果在这个过程中遇到任何异步依赖，则会进入挂起状态。在挂起状态期间，展示的是后备内容。当所有遇到的异步依赖都完成后，`<Suspense>` 会进入完成状态，并将展示出默认插槽的内容。

如果在初次渲染时没有遇到异步依赖，`<Suspense>` 会直接进入完成状态。

进入完成状态后，只有当默认插槽的根节点被替换时，`<Suspense>` 才会回到挂起状态。组件树中新的更深层次的异步依赖不会造成 `<Suspense>` 回退到挂起状态。

发生回退时，后备内容不会立即展示出来。相反，`<Suspense>` 在等待新内容和异步依赖完成时，会展示之前 #default 插槽的内容。这个行为可以通过一个 timeout prop 进行配置：在等待渲染新内容耗时超过 timeout 之后，`<Suspense>` 将会切换为展示后备内容。若 timeout 值为 0 将导致在替换默认内容时立即显示后备内容。

### 事件​

`<Suspense>` 组件会触发三个事件：pending、resolve 和 fallback。pending 事件是在进入挂起状态时触发。resolve 事件是在 default 插槽完成获取新内容时触发。fallback 事件则是在 fallback 插槽的内容显示时触发。

例如，可以使用这些事件在加载新组件时在之前的 DOM 最上层显示一个加载指示器。

### 错误处理

`<Suspense>` 组件自身目前还不提供错误处理，不过你可以使用 errorCaptured 选项或者 onErrorCaptured() 钩子，在使用到 `<Suspense>` 的父组件中捕获和处理异步错误。

### 和其他组件结合

我们常常会将 `<Suspense>` 和 `<Transition>`、`<KeepAlive>` 、  <RouterView> 等组件结合。要保证这些组件都能正常工作，嵌套的顺序非常重要。

下面的示例展示了如何嵌套这些组件，使它们都能按照预期的方式运行。若想组合得更简单，你也可以删除一些你不需要的组件：

```vue
<RouterView v-slot="{ Component }">
  <template v-if="Component">
    <Transition mode="out-in">
      <KeepAlive>
        <Suspense>
          <!-- 主要内容 -->
          <component :is="Component"></component>

          <!-- 加载中状态 -->
          <template #fallback>
            正在加载...
          </template>
        </Suspense>
      </KeepAlive>
    </Transition>
  </template>
</RouterView>
```

Vue Router 使用动态导入对懒加载组件进行了内置支持。这些与异步组件不同，目前他们不会触发 `<Suspense>`。但是，它们仍然可以有异步组件作为后代，这些组件可以照常触发 `<Suspense>`。

### `<Suspense>` 嵌套使用（仅在 3.3+ 支持）

当我们有多个类似于下方的异步组件 (常见于嵌套或基于布局的路由) 时：

```vue
<Suspense>
  <component :is="DynamicAsyncOuter">
    <component :is="DynamicAsyncInner" />
  </component>
</Suspense>
```

`<Suspense>` 创建了一个边界，它将如预期的那样解析树下的所有异步组件。然而，当我们更改 DynamicAsyncOuter 时，`<Suspense>` 会正确地等待它，但当我们更改 DynamicAsyncInner 时，嵌套的 DynamicAsyncInner 会呈现为一个空节点，直到它被解析为止 (而不是之前的节点或回退插槽)。

为了解决这个问题，我们可以使用嵌套的方法来处理嵌套组件的补丁，就像这样：

```vue
<Suspense>
  <component :is="DynamicAsyncOuter">
    <Suspense suspensible> <!-- 像这样 -->
      <component :is="DynamicAsyncInner" />
    </Suspense>
  </component>
</Suspense>
```

如果你不设置 suspensible 属性，内部的 `<Suspense>` 将被父级 `<Suspense>` 视为同步组件。这意味着它将会有自己的回退插槽，如果两个 Dynamic 组件同时被修改，则当子 `<Suspense>` 加载其自己的依赖关系树时，可能会出现空节点和多个修补周期，这可能不是理想情况。设置后，所有异步依赖项处理都会交给父级 `<Suspense>` (包括发出的事件)，而内部 `<Suspense>` 仅充当依赖项解析和修补的另一个边界。

