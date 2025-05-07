---
title: Vue3插槽Slots
tags: [Vue3]
categories: [vue]
date: 2025-05-07 19:10:01
---

插槽的作用是为子组件传递一些模板片段，让子组件在它们的组件中渲染这些片段。

## 插槽内容与出口

例如：这里有一个 `<FancyButton>` 组件，可以像这样使用：

```vue
<FancyButton>
  Click me! <!-- 插槽内容 -->
</FancyButton>
```

而 <FancyButton> 的模板是这样的：

```vue
<button class="fancy-btn">
  <slot></slot> <!-- 插槽出口 -->
</button>
```

`<slot>` 元素是一个插槽出口 (slot outlet)，标示了父元素提供的插槽内容 (slot content) 将在哪里被渲染。

**插槽内容可以是任意合法的模板内容，不局限于文本。通过使用插槽，组件更加灵活和具有可复用性。**

## 渲染作用域

插槽内容可以访问到父组件的数据作用域，无法访问子组件的数据。

## 默认内容

在外部没有提供任何内容的情况下，可以为插槽指定默认内容。只需要将默认内容写在 `<slot>` 标签之间来作为默认内容：

```vue
<button type="submit">
  <slot>
    Submit <!-- 默认内容 -->
  </slot>
</button>
```

如果我们提供了插槽内容，如果我们提供了插槽内容。

## 具名插槽

如果需要在一个组件中包含多个插槽出口，那么就需要用到具名插槽。

例如：在一个 `<BaseLayout>` 组件中，有如下模板：

```vue
<div class="container">
  <header>
    <!-- 标题内容放这里 -->
  </header>
  <main>
    <!-- 主要内容放这里 -->
  </main>
  <footer>
    <!-- 底部内容放这里 -->
  </footer>
</div>
```

对于这种场景，`<slot>` 元素可以有一个特殊的 attribute `name`，用来给各个插槽分配唯一的 ID，以确定每一处要渲染的内容：

```vue
<div class="container">
  <header>
    <slot name="header"></slot>
  </header>
  <main>
    <slot></slot>
  </main>
  <footer>
    <slot name="footer"></slot>
  </footer>
</div>
```

这类带 name 的插槽被称为具名插槽 (named slots)。没有提供 name 的 `<slot>` 出口会隐式地命名为“default”。

要为具名插槽传入内容，我们需要使用一个含 v-slot 指令的 `<template>` 元素，并将目标插槽的名字传给该指令：

```vue
<BaseLayout>
  <template v-slot:header>
    <h1>Here might be a page title</h1>
  </template>

  <template v-slot:default>
    <p>A paragraph for the main content.</p>
    <p>And another one.</p>
  </template>

  <template v-slot:footer>
    <p>Here's some contact info</p>
  </template>
</BaseLayout>
```

v-slot 有对应的简写 #，因此 `<template v-slot:header>` 可以简写为 `<template #header>`。

当一个组件同时接收默认插槽和具名插槽时，所有位于顶级的非 `<template>` 节点都被隐式地视为默认插槽的内容。

```vue
<BaseLayout>
  <template #header>
    <h1>Here might be a page title</h1>
  </template>

  <!-- 隐式的默认插槽 -->
  <p>A paragraph for the main content.</p>
  <p>And another one.</p>

  <template #footer>
    <p>Here's some contact info</p>
  </template>
</BaseLayout>
```

## 条件插槽

条件插槽的使用场景是：需要根据内容是否被传入了插槽来渲染某些内容。

在下面的示例中，我们定义了一个卡片组件，它拥有三个条件插槽：header、footer 和 default。 当 header、footer 或 default 的内容存在时，我们希望包装它以提供额外的样式：

```vue
<template>
  <div class="card">
    <div v-if="$slots.header" class="card-header">
      <slot name="header" />
    </div>
    
    <div v-if="$slots.default" class="card-content">
      <slot />
    </div>
    
    <div v-if="$slots.footer" class="card-footer">
      <slot name="footer" />
    </div>
  </div>
</template>
```

## 动态插槽名

动态指令参数在 v-slot 上也是有效的，即可以定义下面这样的动态插槽名：

```vue
<base-layout>
  <template v-slot:[dynamicSlotName]>
    ...
  </template>

  <!-- 缩写为 -->
  <template #[dynamicSlotName]>
    ...
  </template>
</base-layout>
```

## 作用域插槽

在某些场景下插槽的内容可能想要同时使用父组件域内和子组件域内的数据。这时可以像对组件传递 props 那样，向一个插槽的出口上传递 attributes：

```vue
<!-- <MyComponent> 的模板 -->
<div>
  <slot :text="greetingMessage" :count="1"></slot>
</div>
```

当需要接收插槽 props 时，默认插槽和具名插槽的使用方式有一些区别。

## 默认插槽接受 props

默认插槽通过子组件标签上的 v-slot 指令，直接接收到了一个插槽 props 对象：

```vue
<MyComponent v-slot="slotProps">
  {{ slotProps.text }} {{ slotProps.count }}
</MyComponent>
```

和函数的参数类似，我们也可以在 v-slot 中使用解构：

```vue
<MyComponent v-slot="{ text, count }">
  {{ text }} {{ count }}
</MyComponent>
```

## 具名作用域插槽接受 props

具名作用域插槽的工作方式也是类似的，插槽 props 可以作为 v-slot 指令的值被访问到：`v-slot:name="slotProps"`。当使用缩写时是这样：

```vue
<MyComponent>
  <template #header="headerProps">
    {{ headerProps }}
  </template>

  <template #default="defaultProps">
    {{ defaultProps }}
  </template>

  <template #footer="footerProps">
    {{ footerProps }}
  </template>
</MyComponent>
```

向具名插槽中传入 props：

```vue
<slot name="header" message="hello"></slot>
```

注意插槽上的 name 是一个 Vue 特别保留的 attribute，不会作为 props 传递给插槽。

如果你同时使用了具名插槽与默认插槽，则需要为默认插槽使用显式的 `<template>` 标签。尝试直接为组件添加 v-slot 指令将导致编译错误。

```vue
<!-- <MyComponent> template -->
<div>
  <slot :message="hello"></slot>
  <slot name="footer" />
</div>
```

```vue
<!-- 该模板无法编译 -->
<MyComponent v-slot="{ message }">
  <p>{{ message }}</p>
  <template #footer>
    <!-- message 属于默认插槽，此处不可用 -->
    <p>{{ message }}</p>
  </template>
</MyComponent>
```

为默认插槽使用显式的 `<template>` 标签有助于更清晰地指出 message 属性在其他插槽中不可用：

```vue
<MyComponent>
  <!-- 使用显式的默认插槽 -->
  <template #default="{ message }">
    <p>{{ message }}</p>
  </template>

  <template #footer>
    <p>Here's some contact info</p>
  </template>
</MyComponent>
```