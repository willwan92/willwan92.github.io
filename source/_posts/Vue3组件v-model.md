---
title: Vue3组件v-model
tags: [Vue3]
categories: [vue]
date: 2025-05-06 17:22:54
---

v-model 可以在组件上使用以实现双向绑定。从 Vue 3.4 开始，推荐的实现方式是使用 defineModel() 宏：

```vue
<!-- Child.vue -->
<script setup>
const model = defineModel()

function update() {
  model.value++
}
</script>

<template>
  <div>Parent bound v-model is: {{ model }}</div>
  <button @click="update">Increment</button>
</template>
```

父组件可以用 v-model 绑定一个值：

```vue
<!-- Parent.vue -->
<Child v-model="countModel" />
```

defineModel() 返回的值是一个 ref。它可以像其他 ref 一样被访问以及修改，不过它能起到在父组件和当前变量之间的双向绑定的作用：

- 它的 .value 和父组件的 v-model 的值同步；
- 当它被子组件变更了，会触发父组件绑定的值一起更新。

**这意味着也可以用 v-model 把这个 ref 绑定到一个原生 input 元素上，在提供相同的 v-model 用法的同时轻松包装原生 input 元素：**

```vue
<script setup>
const model = defineModel()
</script>

<template>
  <input v-model="model" />
</template>
```

## v-model 的底层机制

defineModel 是一个便利宏。编译器将其展开为以下内容：

- 一个名为 modelValue 的 prop，本地 ref 的值与其同步；
- 一个名为 update:modelValue 的事件，当本地 ref 的值发生变更时触发。

在 3.4 版本之前，一般会按照如下的方式来实现上述相同的子组件：

```vue
<!-- Child.vue -->
<script setup>
const props = defineProps(['modelValue'])
const emit = defineEmits(['update:modelValue'])
</script>

<template>
  <input
    :value="props.modelValue"
    @input="emit('update:modelValue', $event.target.value)"
  />
</template>
```

然后，父组件中的 v-model="foo" 将被编译为：

```vue
<!-- Parent.vue -->
<Child
  :modelValue="foo"
  @update:modelValue="$event => (foo = $event)"
/>
```

因为 defineModel 是声明了一个 prop，所以可以通过给 defineModel 传递选项，来声明底层 prop 的选项：

```js
// 使 v-model 必填
const model = defineModel({ required: true })

// 提供一个默认值
const model = defineModel({ default: 0 })
```

**注意：如果为 defineModel prop 设置了一个 default 值且父组件没有为该 prop 提供任何值，会导致父组件与子组件之间不同步。**

例如：在下面的示例中，父组件的 myRef 是 undefined，而子组件的 model 是 1：

子组件：

```js
const model = defineModel({ default: 1 })
```

父组件：

```js
const myRef = ref()
```

```vue
<Child v-model="myRef"></Child>
```

## v-model 的参数

组件上的 v-model 也可以接受一个参数：

```vue
<MyComponent v-model:title="bookTitle" />
```

在子组件中，我们可以通过将字符串作为第一个参数传递给 defineModel() 来支持相应的参数：

```vue
<!-- MyComponent.vue -->
<script setup>
const title = defineModel('title')
</script>

<template>
  <input type="text" v-model="title" />
</template>
```

如果需要额外的 prop 选项，应该在 model 名称之后传递：

```js
const title = defineModel('title', { required: true })
```

**v-model 的参数 3.4 之前的用法：**

```vue
<!-- MyComponent.vue -->
<script setup>
defineProps({
  title: {
    required: true
  }
})
defineEmits(['update:title'])
</script>

<template>
  <input
    type="text"
    :value="title"
    @input="$emit('update:title', $event.target.value)"
  />
</template>
```

## 多个 v-model 绑定

利用在 v-model 的参数小节中学到的指定参数与事件名的技巧，可以在单个组件实例上创建多个 v-model 双向绑定。组件上的每一个 v-model 都会同步不同的 prop，而无需额外的选项：

```vue
<UserName
  v-model:first-name="first"
  v-model:last-name="last"
/>
```

```vue
<script setup>
const firstName = defineModel('firstName')
const lastName = defineModel('lastName')
</script>

<template>
  <input type="text" v-model="firstName" />
  <input type="text" v-model="lastName" />
</template>
```

多个 v-model 3.4 之前的用法参看上面 v-model 的参数 3.4 之前的用法。

## 处理 v-model 修饰符

学习输入绑定时，我们知道了 v-model 有一些内置的修饰符，例如 .trim，.number 和 .lazy。在某些场景下，你可能想要一个自定义组件的 v-model 支持自定义的修饰符。

我们看一下如何创建一个自定义的修饰符 capitalize，它会自动将 v-model 绑定输入的字符串值第一个字母转为大写：

```vue
<MyComponent v-model.capitalize="myText" />
```

通过像这样解构 defineModel() 的返回值，可以在子组件中访问添加到组件 v-model 的修饰符：

```vue
<script setup>
const [model, modifiers] = defineModel()

console.log(modifiers) // { capitalize: true }
</script>

<template>
  <input type="text" v-model="model" />
</template>
```

为了能够基于修饰符选择性地调节值的读取和写入方式，我们可以给 defineModel() 传入 get 和 set 这两个选项。这两个选项在从模型引用中读取或设置值时会接收到当前的值，并且它们都应该返回一个经过处理的新值。下面是一个例子，展示了如何利用 set 选项来应用 capitalize (首字母大写) 修饰符：

```vue
<script setup>
const [model, modifiers] = defineModel({
  set(value) {
    if (modifiers.capitalize) {
      return value.charAt(0).toUpperCase() + value.slice(1)
    }
    return value
  }
})
</script>

<template>
  <input type="text" v-model="model" />
</template>
```

**处理 v-model 修饰符 3.4 之前的用法：**

```vue
<script setup>
const props = defineProps({
  modelValue: String,
  modelModifiers: { default: () => ({}) }
})

const emit = defineEmits(['update:modelValue'])

function emitValue(e) {
  let value = e.target.value
  if (props.modelModifiers.capitalize) {
    value = value.charAt(0).toUpperCase() + value.slice(1)
  }
  emit('update:modelValue', value)
}
</script>

<template>
  <input type="text" :value="props.modelValue" @input="emitValue" />
</template>
```

## 带参数的 v-model 修饰符

下面是另一个例子，展示了如何在使用多个不同参数的 v-model 时使用修饰符：

```vue
<UserName
  v-model:first-name.capitalize="first"
  v-model:last-name.uppercase="last"
/>
```

```vue
<script setup>
const [firstName, firstNameModifiers] = defineModel('firstName')
const [lastName, lastNameModifiers] = defineModel('lastName')

console.log(firstNameModifiers) // { capitalize: true }
console.log(lastNameModifiers) // { uppercase: true }
</script>
```

带参数的 v-model 修饰符 3.4 之前的用法参考 v-model 修饰符 3.4 之前的用法。