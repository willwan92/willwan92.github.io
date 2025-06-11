---
title: Vue3组件Attributes继承
tags: [Vue3]
categories: [vue]
date: 2025-05-07 15:27:08
---

## 透传 attribute 

**透传 attribute 指的是传递给一个组件，却没有被该组件声明为 props 或 emits 的 attribute 或者 v-on 事件监听器**。最常见的例子就是 class、style 和 id。

**当一个组件以单个元素为根作渲染时，透传的 attribute 会自动被添加到根元素上。**

例如：有一个 `<MyButton>` 组件，它的模板长这样：

```vue
<!-- <MyButton> 的模板 -->
<button>Click Me</button>
```

一个父组件使用了这个组件，并且传入了 class：

```vue
<MyButton class="large" />
```

最后渲染出的 DOM 结果是：


```vue
<button class="large">Click Me</button>
```

**透传的 class 、 style 和 事件监听器会和子组件根元素的合并。**

## 深层组件继承

有些情况下一个组件会在根节点上渲染另一个组件。例如，我们重构一下 `<MyButton>`，让它在根节点上渲染 `<BaseButton>`：

```vue
<!-- <MyButton/> 的模板，只是渲染另一个组件 -->
<BaseButton />
```

此时 <MyButton> 接收的透传 attribute 会直接继续传给 <BaseButton>。

注意点：

1. 透传的 attribute 不会包含 `<MyButton>` 上声明过的 props 或是针对 emits 声明事件的 v-on 侦听函数。

2. 透传的 attribute 若符合声明，也可以作为 props 传入 `<BaseButton>`。


## 禁用 Attributes 继承

如果不想要一个组件自动地继承 attribute，你可以在组件选项中设置 `inheritAttrs: false`。

从 3.3 开始你也可以直接在 `<script setup>` 中使用 defineOptions：

```vue
<script setup>
defineOptions({
  inheritAttrs: false
})
// ...setup 逻辑
</script>
```

**最常见的需要禁用 attribute 继承的场景就是 attribute 需要应用在根节点以外的其他元素上**。通过设置 inheritAttrs 选项为 false，可以完全控制透传进来的 attribute 被如何使用。这些透传进来的 attribute 可以在模板的表达式中直接用 $attrs 访问到。

例如：现在我们要再次使用一下之前小节中的 <MyButton> 组件例子。有时候我们可能为了样式，需要在 `<button>` 元素外包装一层 `<div>`：

```vue
<div class="btn-wrapper">
  <button class="btn">Click Me</button>
</div>
```

我们想要所有像 class 和 v-on 监听器这样的透传 attribute 都应用在内部的 `<button>` 上而不是外层的 `<div>` 上。我们可以通过设定 inheritAttrs: false 和使用 v-bind="$attrs" 来实现：

```vue
<div class="btn-wrapper">
  <button class="btn" v-bind="$attrs">Click Me</button>
</div>
```

注意点：

1. **和 props 有所不同，透传 attributes 在 JavaScript 中保留了它们原始的大小写**，所以像 foo-bar 这样的一个 attribute 需要通过 $attrs['foo-bar'] 来访问。

2. 像 @click 这样的一个 v-on 事件监听器将在此对象下被暴露为一个函数 $attrs.onClick。

## 多根节点的 Attributes 继承

和单根节点组件有所不同，**有着多个根节点的组件没有自动 attribute 透传行为**。如果 $attrs 没有被显式绑定，将会抛出一个运行时警告。

## 在 JavaScript 中访问透传 Attributes

如果需要，**可以在 `<script setup>` 中使用 useAttrs() API 来访问一个组件的所有透传 attribute**：

```vue
<script setup>
import { useAttrs } from 'vue'

const attrs = useAttrs()
</script>
```

如果没有使用 `<script setup>`，attrs 会作为 setup() 上下文对象的一个属性暴露：

```vue
export default {
  setup(props, ctx) {
    // 透传 attribute 被暴露为 ctx.attrs
    console.log(ctx.attrs)
  }
}
```

**注意：attrs 对象是最新的透传 attribute，但它并不是响应式的 (考虑到性能因素)**。你不能通过侦听器去监听它的变化。如果你需要响应性，可以使用 prop。或者你也可以使用 onUpdated() 使得在每次更新时结合最新的 attrs 执行副作用。