<a name="index">**目录**</a>

- <a href="#ch1">**1 什么是 rem**</a>
- <a href="#ch2">**2 rem 适配算法**</a>
- <a href="#ch3">**3 适配举例**</a>
- <a href="#ch4">**4 实际开发**</a>

<br>
<br>

### <a name="ch1">1 什么是 rem</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>

rem (font size of the root element) 表示的是根元素的字体大小，如下：

```css
html {
    font-size: 16px;
}
```

根元素字体大小为 16px，那么 1rem = 16px。

<br>
<br>

### <a name="ch2">2 rem 适配算法</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>

rem 作为一个长度单位，除了可以作为字体大小来设置，还可以设置 html 的任何元素的尺寸。一般来说，移动设备大小不一，同一个页面元素要在不同的设备上进行适配，以保证视觉效果。

一个总的原则是 **将元素的尺寸大小基于设备的屏幕宽度进行重新计算** 。我们设屏幕宽度为 *```W```* (单位：px)；该设备 1 个单位 rem 的值为 *```R```* (单位：px)；目标元素要设置的尺寸大小（比如元素的长或者宽）为 *```Sp```* (单位：px) 或 *```Sr```* (单位：rem)。

首先，*```W```* 和 *```Sp```* 都有一个初始值，它们的值等于设计师给定的设计图标注。一般来说设计师会根据某一种设备的屏幕尺寸来进行设计，比如 iPhone6，其屏幕宽度为 375px。假设初始值分别为：*```w```* *```s```*，于是有：

第一步，初始化 *```W```* 和 *```Sp```*；
```javascript
W = w // =375px for iPhone6
Sp = s // 设计稿实际标注的值
```

第二步，计算 *```R```* 以及 *```Sr```*；
```javascript
R = W / 10
Sr = Sp / R // 此为 Sr 的最终值
```

第三步，在不同尺寸设备上重新计算 *```R```*，并将其值设置为根元素的字体大小，并更新 *```Sp```*
```javascript
R = W / 10 // 同时设置：document.getElementsByTagName('html')[0].style.fontSize = R + 'px'
Sp = Sr * R // 重新计算出 px 值
```

**注意：**

- **为什么 *```R```* 的计算是屏幕宽度除以 10 呢？** 其实理论上直接设置 *```R = W```* 也是可以的，但是这样一来，默认的字体大小就显得过大，除以 10 可以当做是一种约定俗成的规定。
- **实际上 *```Sp```* 的计算并不是必要的。** 当在第二步计算出 *```Sr```* 之后，就可以直接将目标元素的尺寸设置为 *```element.style.width/height = Sr + 'rem'```*，之所以每次计算 *```Sp```* 是为了便于说明它的实际像素值。

<br>
<br>

### <a name="ch3">3 适配举例</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>

作为以上算法的一个例子。我们假设设计稿是按照 iPhone6 的尺寸给出的，某个 div 元素的宽度在设计中标注为 60px，于是：

第一步：
```javascript
W = 375
Sp = 60
```

第二步：
```javascript
R = W / 10 = 375 / 10 = 37.5
Sr = Sp / R = 60 / 37.5 = 1.6
```
这一步，我们将直接设置 div 的宽度为 1.6rem 就可以了。

第三步，我们假设新的需要适配的设备为 iPhone6 Plus，宽度为 414px：
```javascript
R = W / 10 = 414 / 10 = 41.4
Sp = Sr * R = 1.6 * 41.4 = 66.24
```
所以在 iPhone6 Plus 上，其宽度实际为 66.24px。

<br>
<br>

### <a name="ch4">4 实际开发</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>

在实际开发中，我们要执行以下两步：

1. 根据设计稿，我们要先完成算法的前两步。即按照设计基于的设备，计算出一个初始的 *```R```* 值，然后将每个目标元素的尺寸换算成 *```Sr```*；
2. 最后，在页面渲染之前，需要重新计算根元素的字体大小，即 *```W```*：

```javascript
document.getElementsByTagName('html')[0].style.fontSize = window.innerWidth / 10 + 'px';
```

以上的第一步往往比较繁琐，因为每一个元素都需要手动换算成 rem 值。一种可行的办法是，在 css 的实际设置中，仍然按照设计稿给出的像素值直接设置，然后写一个脚本程序，其功能是将所有的 css 文件中的尺寸大小都按照上述的算法换算成对应的 *```Sr```* 值。在测试页面之前，先执行该脚本即可。

实际上，已经有相应的插件实现了此脚本的功能了—— *```PxToRem```*，我们可以在 webpack 的 postcss-loader 里包含这个插件，具体的用法，可以参考 webpack 官网指南以及 [PxToRem 插件说明](https://github.com/cuth/postcss-pxtorem)




