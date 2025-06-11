---
title: vue3项目总结
tags: ["vue3"]
categories: ["vue"]
date: 2022-04-06 19:43:30
---

## 基础变化

- vue3 全家桶使用：和 vue2 的相比变化是 new 实例化改为 creatXXX，例如：new Vue() 改为 createApp()

- 注册组件：和 vue2 的相比变化是 Vue.component() 改为 根应用 app.component()

## composition(组合式) API

将同一个逻辑关注点相关代码放在一起。

### 组合式API入口：setup

setup：在组件创建之前执行，setup 选项是一个接收 props 和 context 两个参数的函数。

#### 参数说明

- props：

和vue2一样，setup 函数中的 props 是响应式的，当传入新的 prop 时，它将被更新。

**因为是响应式的，所以不能使用 ES6 解构，它会消除 prop 的响应性。**

如果需要解构 prop，可以在 setup 函数中使用 toRefs 函数来完成此操作：

```js
// MyBook.vue

import { toRefs } from 'vue'

setup(props) {
  const { title } = toRefs(props)

  console.log(title.value)
}
```

如果 title 是可选的 prop，则传入的 props 中可能没有 title 。在这种情况下，toRefs 将不会为 title 创建一个 ref 。你需要使用 toRef 替代它：

```js
// MyBook.vue
import { toRef } from 'vue'
setup(props) {
  const title = toRef(props, 'title')
  console.log(title.value)
}
```

- Context

context 是一个普通 JavaScript 对象，暴露了其它可能在 setup 中有用的值：

```js
// MyBook.vue

export default {
  setup(props, context) {
    // Attribute (非响应式对象，等同于 $attrs)
    console.log(context.attrs)

    // 插槽 (非响应式对象，等同于 $slots)
    console.log(context.slots)

    // 触发事件 (方法，等同于 $emit)
    console.log(context.emit)

    // 暴露公共 property (函数)
    console.log(context.expose)
  }
}
```

context 是一个普通的 JavaScript 对象，它不是响应式的，这意味着可以安全地对 context 使用 ES6 解构。

执行 setup 时，无法访问以下组件选项：

- data
- computed
- methods
- refs (模板 ref)

[setup详细指南](https://v3.cn.vuejs.org/guide/composition-api-setup.html)


### 响应式变量

```js
<!-- MyBook.vue -->
<template>
  <div>{{ collectionName }}: {{ readersNumber }} {{ book.title }}</div>
</template>

<script>
  import { ref, reactive } from 'vue'

  export default {
    props: {
      collectionName: String
    },
    setup(props) {
      // 单个值
      const readersNumber = ref(0)
      // 单个值取值要通过.value，在模板里会自动解包，可以不用手动获取。
      console.log(count.value) // 0
      // 对象
      const book = reactive({ title: 'Vue 3 Guide' })

      // 暴露给 template
      return {
        readersNumber,
        book
      }
    }
  }
</script>
```

[更多响应性基础API](https://v3.cn.vuejs.org/api/reactivity-api.html)

### 在 setup 内定义和使用方法

```js
// src/components/UserRepositories.vue
import { ref, onMounted } from 'vue'

// 在我们的组件中
setup (props) {
  const name = ref('')
  
  const setName = (str) => {
    name.value = str
  }

  setName('wan')

  return {
    name,
    setName
  }
}
```

### 在 setup 内注册生命周期钩子

```js
// src/components/UserRepositories.vue
import { ref, onMounted } from 'vue'

// 在我们的组件中
setup (props) {
  const repositories = ref([])
  const getUserRepositories = async () => {
    repositories.value = await fetchUserRepositories(props.user)
  }

  onMounted(getUserRepositories) // 在 `mounted` 时调用 `getUserRepositories`

  return {
    repositories,
    getUserRepositories
  }
}
```

下表包含如何在 setup () 内部调用生命周期钩子：

| 选项式 API | Hook inside setup |
| --- | --- |
| beforeCreate | Not needed* |
| created | Not needed* |
| beforeMount | onBeforeMount |
| mounted | onMounted |
| beforeUpdate | onBeforeUpdate |
| updated | onUpdated |
| beforeUnmount | onBeforeUnmount |
| unmounted | onUnmounted |
| errorCaptured | onErrorCaptured |
| renderTracked | onRenderTracked |
| renderTriggered | onRenderTriggered |
| activated | onActivated |
| deactivated | onDeactivated |

### 在 setup 内计算属性和侦听器

#### computed：独立的计算属性

```js
import { ref, computed } from 'vue'

const counter = ref(0)
const twiceTheCounter = computed(() => counter.value * 2)

counter.value++
console.log(counter.value) // 1
console.log(twiceTheCounter.value) // 2
```

[更多computed属性说明](https://v3.cn.vuejs.org/api/computed-watch-api.html#computed)


#### watchEffect 侦听器

为了根据响应式状态自动应用和重新应用副作用，我们可以使用 watchEffect 函数。它立即执行传入的一个函数，同时响应式追踪其依赖，并在其依赖变更时重新运行该函数。

```js
const count = ref(0)

watchEffect(() => console.log(count.value))
// -> logs 0

setTimeout(() => {
  count.value++
  // -> logs 1
}, 100)
```

#### watch 侦听器

watch API 与选项式 API this.$watch (以及相应的 watch 选项) 完全等效。watch 需要侦听特定的数据源，并在单独的回调函数中执行副作用。默认情况下，它也是惰性的——即回调仅在侦听源发生变化时被调用。

```js
import { ref, watch } from 'vue'

const counter = ref(0)
watch(counter, (newValue, oldValue) => {
  console.log('The new counter value is: ' + counter.value)
})
```

与 watchEffect 比较，watch 允许我们：

> 懒执行副作用；
> 更具体地说明什么状态应该触发侦听器重新运行；
> 访问侦听状态变化前后的值。

[计算属性和侦听器详细指南](https://v3.cn.vuejs.org/guide/reactivity-computed-watchers.html)

### 模板引用

在使用组合式 API 时，响应式引用和模板引用的概念是统一的。为了获得对模板内元素或组件实例的引用，我们可以像往常一样声明 ref 并从 setup() 返回：

**注意：板引用只会在初始渲染之后可以使用**

```js
<template> 
  <div ref="root">This is a root element</div>
</template>

<script>
  import { ref, onMounted } from 'vue'

  export default {
    setup() {
      const root = ref(null)

      onMounted(() => {
        // DOM 元素将在初始渲染后分配给 ref
        console.log(root.value) // <div>This is a root element</div>
      })

      return {
        root
      }
    }
  }
</script>
```

这里我们在渲染上下文中暴露 root，并通过 ref="root"，将其绑定到 div 作为其 ref。

在虚拟 DOM 补丁算法中，如果 VNode 的 ref 键对应于渲染上下文中的 ref，则 VNode 的相应元素或组件实例将被分配给该 ref 的值。这是在虚拟 DOM 挂载/打补丁过程中执行的，因此**模板引用只会在初始渲染之后获得赋值**。

作为模板使用的 ref 的行为与任何其他 ref 一样：它们是响应式的，可以传递到 (或从中返回) 复合函数中。

[模板引用详细指南](https://v3.cn.vuejs.org/guide/composition-api-template-refs.html)

### 在 setup 内使用vuex

```js
import { useStore } from 'vuex'

export default ({
  setup() {
    const store = useStore()
    const onSubmit = () => {
      store.dispatch('user/Login', formState).then(() => {
        router.push('/index')
      })
    };
  }
})
```

### 在 setup 内使用vue-router

```js
import { useRouter, useRoute } from 'vue-router'

export default ({
  setup() {
    // 使用 router
    const router = useRouter();
    router.push("/user/login");

    // 使用 route
    const route = useRoute();
    console.log(route.query)
  }
})
```
		





