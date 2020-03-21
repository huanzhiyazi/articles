<a name="index">**目录**</a>

- <a href="#ch1">**1 问题描述**</a>
- <a href="#ch2">**2 原因分析**</a>
- <a href="#ch3">**3 思考总结**</a>

<br>
<br>

### <a name="ch1">1 问题描述</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>

首先假设有一个这样的函数：

```js
const dateSample = (date, defValue = 0) => {
  var date = new Date(date, defValue);
  return date;
}
```

我们可以看到，这个函数在编写习惯上非常糟糕，因为输入参数名和函数体中的变量声明名字是一样的。如果刚开始你用的 Babel 6 版本进行转义，你会发现它运行的非常正常。但是如果你将 Babel 升级到最新的 Babel 7 版本，会发现不管输入 date 是什么值，函数总是返回 `Invalid date`。

<br>
<br>

### <a name="ch2">2 原因分析</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>

毫无疑问，两个版本的 Babel 转义行为不一样。所以我们可以来对比 Babel 6 和 Babel 7 对这段代码的转义结果。

- **首先，用 Babel 6 转义后，输出代码如下：**

```js
var dateSample = function dateSample(date) {
  var defValue = arguments.length > 1 && arguments[1] !== undefined ? arguments[1] : 0;

  var date = new Date(date, defValue);
  return date;
}
```

可以看到，除了 lambda 和参数默认值转义外，其它基本维持不变。根据 **函数作用域链** 的原理，后续虽然声明了 `var date`，但是初始值仍然是输入参数 `date`。很好的维持了该函数的本义。

- **然后，用 Babel 7 来转义相同的初始代码，结果如下：**

```js
var dateSample = function dateSample(date) {
  var defValue = arguments.length > 1 && arguments[1] !== undefined ? arguments[1] : 0;
  return function () {
    var date = new Date(date, defValue);
    return date;
  }();
};
```

看到这段代码，相信你可以恍然大悟！除了参数默认值转义以外，函数主体被作为闭包执行返回了！由于返回的是闭包，而闭包内部声明了 `var date`，所以同样根据 **函数作用域链** 的原理，外部的 `date` 变量优先级要小于闭包内的 `date` 引用优先级，所以 `date` 的有效初始化值是 `undefined` 而不是输入参数 `date`。这就解释了，为什么用 Babel 7 转义后的函数每次返回都是 `Invalid date` 了，转义后的输入参数 `date` 永远不会被引用到了！

<br>
<br>

### <a name="ch3">3 思考总结</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>

实践发现，该函数被转义成闭包必须同时满足变量重复声明和携带参数默认值两个条件。所以，如果把函数改为形如下面的形式，也会出现同样的问题：

```js
const dateSample = (date = '20200202') => {
  var date = new Date(date);
  return date;
}
```

Babel 7 转义后，变成：

```js
var dateSample = function dateSample() {
  var date = arguments.length > 0 && arguments[0] !== undefined ? arguments[0] : '20200202';
  return function () {
    var date = new Date(date);
    return date;
  }();
}
```

而如果原函数写成：

```js
const dateSample = (date) => {
  var date = new Date(date);
  return date;
}
```

转义就是正常的：

```js
var dateSample = function dateSample(date) {
  var date = new Date(date);
  return date;
};
```

或者，消除变量重复声明：

```js
const dateSample = (rawdate, defValue = 0) => {
  var date = new Date(rawdate, defValue);
  return date;
}
```

也是没有问题的，转义结果如下：

```js
var dateSample = function dateSample(rawdate) {
  var defValue = arguments.length > 1 && arguments[1] !== undefined ? arguments[1] : 0;
  var date = new Date(rawdate, defValue);
  return date;
}
```

<br>

我们很难说这个是 Babel 7 的 bug。对于开发者来说，这是其次的，重要在于我们在开发过程中难免会不小心声明了重复的变量，而原生 js 中用 `var` 重复声明变量又是允许的，这更容易导致这种编程习惯的滋生。所以我们在开发过程中，**一方面，需要尽量避免重复变量声明；另一方面，尽量避免用 var 声明变量。** 我们看到用 `var` 声明变量之后，上述转义的问题不会显示提示给开发者出现了问题，但是在业务中必然成为一个严重的 bug 存在。如果我们把函数写成：

```js
const dateSample = (date, defValue = 0) => {
  let date = new Date(date, defValue);
  return date;
}
```

或者：

```js
const dateSample = (date, defValue = 0) => {
  const date = new Date(date, defValue);
  return date;
}
```

那么这会变成一个语法错误显示出来，能够及早的发现潜在的错误。因为 `let` 和 `const` 是局部变量声明，在局部作用范围内是不允许重复声明变量的！



























