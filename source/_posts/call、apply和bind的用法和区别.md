---
title: call、apply和bind的用法和区别
tags: []
categories: [javascript]
date: 2021-12-09 17:11:47
---


## call方法

call()方法使用一个指定的 this 值（this指向执行环境上下文）和单独给出的一个或多个参数来**调用一个函数**。

### 语法

> fun.call(thisArg[, arg1[, arg2[, ...]]])
>    * fun：调用的函数。
>    * thisArg：必选的。函数运行时指定的this值（this指向执行环境上下文）。
>    * arg1, arg2, ...：可选的。指定的参数列表。


**注意：** 指定的this值并不一定是该函数执行时真正的this值，如果这个函数处于非严格模式下，则指定为null和undefined的this值会自动指向全局对象(浏览器中就是window对象)，值为基本类型值(数字，字符串，布尔值)的this会指向该原始值的自动包装对象。

### 用途

使用call方法调用函数并且指定this，**借用自己没有的函数**

```js
function speakName() {
    console.log('My name is ' + this.name);
}

var person1 = {
    name: 'chong',
}

speakName(); // speakName为全局函数，this指向全局对象window，由于全局作用域中不存在变量name，所以输出：My name is
speakName.call(person1); // 输出：My name is chong
```

使用call方法**调用父构造函数，实现继承**

```js
function Product(name, price) {
    this.name = name;
    this.price = price;

    if (price < 0) {
        throw RangeError(
            'Cannot create product ' + this.name + ' with a negative price'
        );
    }
}

function Food(name, price) {
    Product.call(this, name, price);
    this.category = 'food';
}

// 上面Food构造函数等同于以下代码
function Food(name, price) {
    this.name = name;
    this.price = price;

    if (price < 0) {
        throw RangeError(
            'Cannot create product ' + this.name + ' with a negative price'
        );
    }


    this.category = 'food';
}
```

## apply方法

apply() 方法用一个指定的 this 值，以及以一个数组（或类数组对象）的形式提供的参数**调用一个函数**。

### 语法

> func.apply(thisArg, [argsArray])
>    * fun：调用的函数。
>    * thisArg：必选的。函数运行时指定的this值。
>    * argsArray 可选的。一个数组或者类数组对象，其中的数组元素将作为单独的参数传给 func 函数。

**apply方法用途和call方法相同，不再赘述**


## bind方法

bind() 方法**用来创建一个新的函数**。bind() 的第一个参数为这个新函数的 this ，其余参数将作为新函数的参数。

### 语法

> func.bind(thisArg[, arg1[, arg2[, ...]]])
>    * fun：调用的函数。
>    * thisArg：必选的。**调用新函数时作为新函数的 this 值。**如果使用new运算符构造绑定函数，则忽略该值。当使用 bind 在 setTimeout 中创建一个函数（作为回调提供）时，作为 thisArg 传递的任何原始值都将转换为 object。如果 bind 函数的参数列表为空，或者thisArg是null或undefined，执行作用域的 this 将被视为新函数的 thisArg。
>    * arg1, arg2, ...： 可选的。给新函数预设的初始参数列表。
>    返回值：**返回一个原函数的拷贝，并拥有指定的 this 值和初始参数。**

## 用途

1. 创建绑定函数
2. 偏函数：使一个函数拥有预设的初始参数

例如：

```js
var x = 9;
var module = {
    x: 81,
    getX: function() {
        return this.x;
    },
    addX: function(n) {
        return this.x + n;
    }
};

module.getX(); // 81
module.addX(9); // 90

var _getX = module.getX;
var _addX = module.addX;
_getX(); // 返回 9, 因为函数是在全局作用域中调用的
_addX(9); // 返回 18

// 创建一个新函数boundGetX，并把this绑定为module对象；
var boundGetX = _getX.bind(module);
boundGetX(); // 81

// 1. 创建绑定函数 boundAddX，把this绑定为module对象，并预设参数为 10；
var boundAddX = _addX.bind(module, 10);
// 调用新函数时不用再传参数（传了也没用）
boundAddX(); // 91

// 2. 创建一个新函数boundAddX，把this绑定为module对象，不预设参数；
var boundAddX = _addX.bind(module);
// 调用新函数时需要传参数
boundAddX(9); // 90
```

3. 配合 setTimeout

在默认情况下，使用 window.setTimeout() 时，this 关键字会指向 window （或 global）对象。当类的方法中需要 this 指向类的实例时，你可能需要显式地把 this 绑定到回调函数，就不会丢失该实例的引用。

```js
function LateBloomer() {
    // 花瓣数
    this.petalCount = Math.ceil(Math.random() * 12) + 1;
}

// 在调用bloom 1 秒钟后，调用declare
LateBloomer.prototype.bloom = function() {
    window.setTimeout(this.declare.bind(this), 1000);
};

LateBloomer.prototype.declare = function() {
  console.log('I am a beautiful flower with ' +
    this.petalCount + ' petals!');
};

var flower = new LateBloomer();
flower.bloom();  // 一秒钟后, 调用 'declare' 方法
```

## 总结

### call和apply比较

- 相同点：它们都可以用一个指定的this值调用一个函数，并且可以给函数传递一个或者多个参数。**简单来说，它们的用途就是一个对象调/借用自身没有的函数。如果这个函数是构造函数，就可以实现继承**。
- 不同点: 只在于传递参数的形式不同。call方法传递参数是通过参数列表，apply方法传递参数是通过数组/类数组。

### call、apply和bind比较

- 相同点：它们都可以改变一个函数执行时的this，并且可以给函数传递一个或者多个参数。
- 不同点: bind方法返回一个新的函数，需要调用才会执行。而call和apply是立即执行。
## 参考

- [Function.prototype.call()](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Function/call)
- [Function.prototype.apply()](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Function/apply)
- [Function.prototype.bind()](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Function/bind)