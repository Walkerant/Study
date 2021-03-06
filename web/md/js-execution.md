前面我们已经学习了JavaScript的事件循环机制，了解了一段代码是如何被JavaScript引擎执行的，这是粒度最粗的执行单位。接下来，我们开始学习粒度较小的单位：函数的执行机制，以及和函数执行过程相关的所有问题。

# 闭包closure

在计算机领域，闭包closure有三个完全不同的意义：
1. 在编译原理中，它是处理语法产生式的一个步骤；
2. 在计算几何中，它表示包裹平面点集的凸多边形；
3. 在编程语言领域，它表示一种特殊的函数。

上个世界60年代，主流编程语言是基于lambda演算的函数式编程语言，所以最初的闭包定义，使用了大量的函数式术语。一个比较模糊的定义是“带有一系列信息的λ表达式”。其实，λ表达式就是函数。

>我们可以这样简单理解一下，闭包其实只是一个绑定了执行环境的函数，这个函数并不是印在书本里的一条简单的表达式，闭包与普通函数的区别是，它携带了执行的环境，就像人在外星中需要自带吸氧的装备一样，这个函数也带有在程序中生存的环境。

普通的函数就是一系列表达式的集合，只要给定参数，就会得到确定的结果。而闭包还可以携带大量上下文信息，这函数的执行有时候很难理解，但这也是闭包之所以强大的地方。

最初的闭包定义中，包含两部分内容：
- 环境部分
    - 环境
    - 标识符列表
- 表达式部分

这个定义对应到JavaScript标准中，则是：
- 环境部分
    - 环境：函数的词法环境（执行上下文的一部分）
    - 标识符列表：函数中用到的未声明变量
- 表达式部分：函数体

我们可以认为，JavaScript函数完全符合闭包的定义。它的环境部分是由函数词法环境部分组成，标识符列表是函数中用到的未声明变量，表达式部分就是函数体。

## 变量作用域
每个编程语言中都有作用域这个概念，JavaScript也不例外。

JavaScript中的作用域无非两种：全局变量和局部变量，函数内部可以直接读取全局变量，而函数外部无法获取内部的局部变量。这里有一个地方需要注意，函数内部声明变量的时候，一定要使用`var`命令。如果不用的话，你实际上声明了一个全局变量！
```js
    function f1(){
    　　　n = 999;
    }

    f1();

    alert(n); // 999
```

如果非要在函数外部获取函数内的局部变量，通过在函数里面定义一个函数，将要返回的局部变量返回，再返回新定义的函数也能实现。
```js
　　function f1(){

　　　　var n=999;

　　　　function f2(){
　　　　　　alert(n);
　　　　}

　　　　return f2;

　　}

　　var result=f1();

　　result(); // 999
```

## 闭包作用
闭包的作用主要有两个，一个是前面说的读取函数内部的局部变量，另一个是让某些变量始终保存在内存中。

```js
　　function f1(){

　　　　var n=999;

　　　　nAdd = function(){n+=1}   //没有使用var关键字，所以定义了一个全局变量

　　　　function f2(){
　　　　　　alert(n);
　　　　}

　　　　return f2;

　　}

　　var result=f1();

　　result(); // 999

　　nAdd();

　　result(); // 1000
```

在上面的代码中，`f1`运行之后，就会将局部变量`n`保存在内存中，同时在全局作用域中加入`nAdd`，运行`nAdd`保存在内存中的局部变量`n`就会被加1，从而得到1000。

# 执行上下文

执行一段JavaScript代码，不光需要全局变量和局部变量，还需要处理`this`、`with`等特殊语法，这些信息让JavaScript代码的执行变得更加复杂。

**JavaScript标准把一段代码，执行所需要的一切信息定义为“执行上下文”。**

## 版本演变
执行上下文的定义经历了多个版本的演变。

### ES3
执行上下文在ES3中，包含三个部分。

- scope：作用域，也常常被叫做作用域链。
- variable object：变量对象，用于存储变量的对象。
- this value：this值。

### ES5
在ES5中，改进了命名方式，把执行上下文最初的三个部分改为下面这个样子。

- lexical environment：词法环境，当获取变量时使用。
- variable environment：变量环境，当声明变量时使用。
- this value：this值。

### ES2018
在ES2018中，执行上下文又变成了这个样子，this值被归入lexical environment，但是增加了不少内容。

- lexical environment：词法环境，当获取变量或者this值时使用。
- variable environment：变量环境，当声明变量时使用
- code evaluation state：用于恢复代码执行位置。
- Function：执行的任务是函数时使用，表示正在被执行的函数。
- ScriptOrModule：执行的任务是脚本或者模块时使用，表示正在被执行的代码。
- Realm：使用的基础库和内置对象实例。
- Generator：仅生成器上下文有这个属性，表示当前生成器。

如果从实现语言的角度去分析这些内容，这些内容确实不容易理解。但从实际代码出发，去一步一步分析在代码执行过程中，需要哪些信息，然后再思考，有助于理解这些内容。比如这些代码：

```js
    var b = {}
    let c = 1
    this.a = 2;
```

要执行它，我们需要知道以下信息：

1. `var` 把 `b` 声明到哪里；
2. `b` 表示哪个变量；
3. `b` 的原型是哪个对象；
4. `let` 把 `c` 声明到哪里；
5. `this` 指向哪个对象。

## var

`var`是用于声明变量的关键字，它最大的缺陷就是会穿透当前作用域，让变量跑到到上层作用域中。例如下面的代码：
```js
    if (true) {
        var test = true;
    }

    alert(test); // true

    for (var i = 0; i < 10; i++) {
        console.log(i)
    }

    alert(i);   // 10
```

所以，`var`会穿透`if`、`for`等代码块，进入更上层的作用域。但是，如果在function内定义的变量，则不会影响函数外部。
```js
    function sayHi() {
        if (true) {
            var phrase = "Hello";
        }

        alert(phrase); // works
    }

    sayHi();
    alert(phrase); // Error: phrase is not defined
```

针对`var`的这种问题，在没有`let`的时代，用**立即执行的函数表达式（IIFE）**，通过创建一个函数，并立即执行，就像上面的例子一样，可以完美解决变量提升的缺陷。
```js
    (function() {
        var message = "Hello";
        alert(message); // Hello
    })();
```

## let

`let`是从ES6开始引入的新的变量声明方式，比起`var`的诸多弊病，`let`做了非常明确的梳理和规定。

为了实现`let`，JavaScript在运行时引入了块级作用域。以下语句中，都会产生let使用的作用域：
- `for`
- `if`
- `switch`
- `try/catch/finally`

另外，如果用`let`重新变量，会报错。如下所示：
```js
let user;
let user; // Uncaught SyntaxError: Identifier 'user' has already been declared
```

为了减少不必要的麻烦，建议使用多使用 `let`，甚至全用 `let` 声明变量。

# 函数Function

**执行上下文**是JavaScript代码执行所需要的一切信息。也就是说，一段代码的执行结果依赖于执行上下文的内容，如果执行上下文不一样了，相同的代码很可能产生不同的结果。在JavaScript中，切换执行上下文最重要的场景就在函数调用。下面，我们先来认识一下，JavaScript中一共有多少种函数。

## 函数类型
### 普通函数
普通函数是用`function`关键字定义的函数。
```js
    function foo() {
        // code
    }
```

### 箭头函数
箭头函数使用 `=>` 运算符定义的函数。
```js
const foo = () => {
    // code
}
```

### class中定义的函数
`class`中定义的函数，也就是类的访问器属性。
```js
    class C {
        foo(){
            // code
        }
    }
```

### 生成器函数
用 `function *` 定义的函数。
```js
    function foo*(){
        // code
    }
```

### 类（构造器）
用`class`定义的类，实际上也是函数。
```js
    class Foo {
        constructor(){
            // code
        }
    }
```

### 异步函数
普通函数、箭头函数、生成器函数加上`async`关键字。
```js
    async function foo(){
        // code
    }
    const foo = async () => {
        // code
    }
    async function foo*(){
        // code
    }
```

总共8种函数类型，它们的执行上下文，对于普通变量没有什么特殊之处，都是遵循了“继承定义时环境”的规则，主要差异来自 `this` 关键字。

# this
## 普通函数的this
 `this` 关键字是JavaScript执行上下文中非常重要的一个组成部分，同一个函数调用方式不同，得到的this值也不同。例如下面这段代码：
 ```js
    function showThis(){
        console.log(this);
    }

    var o = {
        showThis: showThis
    }

    showThis();     // global
    o.showThis();   // o
 ```

 对此现象，一般认为普通函数的 `this` 指向函数运行所在的环境。例如，在上面的例子中，`showThis()`运行在全局环境中，而 `o.showThis()` 运行在 `o` 这一对象中，因此才得出这样的结果。

 更准确的理解是，**`this` 是由调用它所使用的引用决定的**。
 
 我们获取函数的表达式，实际上返回的并非函数本身，而是一个Reference类型（JavaScript七种标准类型之一）。

 Reference类型由两部分组成：一个对象和一个属性值。在上面的例子中，`showThis()` 产生的Reference类型便是全局对象`global` 或者 `window`，和属性showThis构成；`o.showThis()` 产生的Reference类型又是对象o和属性 `showThis` 构成。

所以，`this` 的真正含义：调用函数时使用的引用Reference，决定了函数执行时刻的 `this` 值。

## 箭头函数的this

把上面的函数改成箭头函数，执行之后发现不管用什么调用，它的值都不变。
 ```js
    var showThis = () => {
        console.log(this);
    }

    var o = {
        showThis: showThis
    }

    showThis();     // global
    o.showThis();   // global

    var o = {}
    o.foo = function foo(){
        console.log(this);
        return () => {
            console.log(this);
            return () => console.log(this);
        }
    }

    o.foo()()();    // o, o, o
 ```

## 访问器属性的this

```js
    class C {
        showThis() {
            console.log(this);
        }
    }
    var o = new C();
    var showThis = o.showThis;

    showThis();     // undefined
    o.showThis();   // o
```
在类中的“方法”，结果又不太一样，使用showThis这个引用去调用方法时，得到了undefined，在对象上调用得到对象本身。

按照上面的方法，可以验证得出：生成器函数、异步生成器函数和异步普通函数跟普通函数行为是一致的，异步箭头函数与箭头函数行为是一致的。

## this的机制
如上文所示，函数不但能够记住定义时的变量，而且还能记住this。

为什么能记住这些值？实际上，JavaScript标准中，为函数规定了用于保存定义时上下文信息的私有属性[[`Environment`]]。当一个函数执行时，会创建一条执行环境记录，记录的外层词法环境会被设置为函数的[[`Environment`]]。这就是在切换执行上下文。

JavaScript用一个栈来管理执行上下文，这个栈中的每一项又包含一个链表。如下图所示：

![执行上下文](../images/js-execution-context.jpg)

调用函数时，会入栈一个新的执行上下文；调用结束时，此执行上下文会被弹出栈。

` this `是一个更为复杂的机制，JavaScript标准定义了 [[`thisMode`]] 私有属性。它有三个取值：
- **lexical**：表示从上下文中找`this`，这对应了箭头函数。
- **global**：表示当`this`为`undefined`时，取全局对象，对应了普通函数。
- **strict**：当严格模式时使用，`this`严格按照调用时传入的值，可能为`null`或者`undefined`。

在上面的代码中，对象方法和普通函数的` this `有差异，就是因为class被设计为默认按照strict模式执行。

上面说函数执行时，会创建一条新的环境记录，将外层词法环境设置为函数的[[`Environment`]]。除此之外，还会根据`this`关键字的[[`thisMode`]]来标记此记录的[[`ThisBindingStatus`]]私有属性。

当代码执行遇见`this`关键字时，会逐层检查当前环境中的[[`ThisBindingStatus`]]，当找到有`this`的环境记录时，便可获取当前的`this`值。

这种规则的导致的结果就是，嵌套的箭头函数中的`this`都指向外层`this`值。
```js
    var o = {}
    o.foo = function foo(){
        console.log(this);
        return () => {
            console.log(this);
            return () => console.log(this);
        }
    }

    o.foo()()(); // o, o, o
```

## 操作this的内置函数
JavaScript提供了两个函数：`Function.prototype.call` 和 `Function.prototype.apply` 可以指定函数调用时传入的`this`值。

```js
    function foo(a, b, c){
        console.log(this);
        console.log(a, b, c);
    }
    foo.call({}, 1, 2, 3);
    foo.apply({}, [1, 2, 3]);
```

`call `和` apply `的作用是一样的，只是传参的方式不一样。前者传入分开的参数，后者传入一个参数数组。

此外，还有 `Function.prototype.bind`，它可以生成一个绑定过的函数。

# 总结
要想彻底理解JavaScript的代码执行机制，就必须理解执行上下文、`this`、闭包和函数，它们共同构成JavaScript最常用的代码执行单元。
- 执行上下文：一段代码执行所需要的所有信息，包括变量、`this`等，经历了多个版本的更迭，内容越来越丰富；
- 函数：一段逻辑上相关的代码组织形式，是最基本的代码执行单元，JavaScript中共有8种函数；
- `this`：指向函数运行所在的环境，由调用它所使用的的引用Reference决定，里面有更加复杂的内在机制；
- 闭包：其实就是绑定了执行环境信息的函数，这让函数的执行变得复杂，同时也是强大的原因。