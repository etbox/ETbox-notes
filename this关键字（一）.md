最近重温js基础，对this的指向有了更深的理解，想与大家分享一下。
聊this关键字要先从this的源头说起。

> 创建函数时，系统会（在后台）创建一个名为this的关键字，它链接到运行改函数的对象。换个说法就是，this对其函数的作用域是可见的，而且它是函数属性/方法所在的一个引用。
> ——《JavaScript启示录》

也就是说，在一个函数中，this是被系统隐式创建的，并且默认指向这个函数本身。我们先来看一个例子（本文默认的执行环境是浏览器，因为在node环境下全局对象为undefined）：

```javascript
var foo = 'foo';
var myObject = {
  foo: 'I am myObject.foo'
};

var sayFoo = function() {
  console.log(this['foo']);
};
// 给myObject设置一个sayFoo属性，并指向到sayFoo函数
myObject.sayFoo = sayFoo;

myObject.sayFoo();  // 输出'I am myObject.foo'
sayFoo();  // 输出'foo'
```

在这个例子中，全局环境下有一个sayFoo函数，myObject也有一个sayFoo函数，但是他们输出的this.foo却不同。原因正是在对象/函数中的this指向的是它本身，而在全局环境下，this指向全局对象。我们再来看一个例子：

```javascript
var foo = {
  func1: function(bar) {
    bar();  // 输出window，而不是foo
    console.log(this);  // 这里的this关键字是foo对象的一个引用
  }
};

foo.func1(function() {
  console.log(this);
});
```

我们将this传入console.log函数中，让它打印出其当前所在的环境。输出将会有两行，第一行输出window，是由bar()执行输出的，而第二行是foo。这里的玄机就在于第二行的foo是由foo自身的函数调用this输出的，而第一个window是由传入func1的匿名函数执行输出的。可以看成是这样子：

```javascript
var foo = {
  func1: function((function() {console.log(this);})()) {
    console.log(this);
  }
};

foo.func1();
```

当然，只是看成这个样子，实际上执行会报错的，这段的理解基于想象：在这里，func1没有传入参数就直接执行了，但是在构造它的时候就传入了一个匿名立即执行函数作为参数。在传入该函数的时候，该函数自己就被执行了，它输出this，这个this应当指向全局环境，因为这个匿名函数不是被其他对象/函数调用的。

以上的想象就解释了为什么第一行输出的会是window。那么现在问题来了，究竟什么时候this的调用才指向这个对象/函数本身，而什么时候指向全局环境呢？我们来看下面这个例子：

```javascript
var myObject = {
  func1: function() {
    console.log(this);  // 输出myObject
    var func2 = function() {
      console.log(this);  // 输出window，从此开始，this都是window对象（严格模式下都是undefined）
      var func3 = function() {
        console.log(this);  // 输出window，即head对象
      }();
    }();
  }
};

myObject.func1();
```

事实上，在一个函数内再创建一个函数的情况下就会有异变发生。func1内的this指向myObject是因为func1是myObject的一个属性，所以它的this指向myObject。而func2是在func1内创建的一个局部变量，它指向一个匿名函数，而它本身并不能被func1或者myObject调用，所以其内部的this不知道要指向谁，只能指向全局环境。同理，func3内的this也是如此。
那么我们该如何正确的在func2和func3内取得myObject呢？我们看看下面这个方法：

```javascript
var myObject = {
  myProperty: 'I can see the light',
  myMethod: function() {
    var that = this;  // myMethod作用域内，保存this引用（也就是，myObject）
    var helperFunction = function() {  // 子函数
      console.log(that.myProperty);  // 输出通过作用域得到'I can see the light'，因为that = this
      console.log(this);  // 如果不使用that，则输出window
    }();
  }
};

myObject.myMethod();
```

在myMethod内，我们创建了一个that变量用来保存当前的this，也就是指向myObject的指针。于是在局部变量helperFunction内部，我们依旧能正确的调用myObject。

现在我们来做一个小结：
1. this是在对象/函数创建时，系统自动给我们创建的，并且默认指向该对象/函数。
1. 在全局环境下，this默认指向window。
1. 用特别的方式调用this，例如在局部变量内调用this，this会指向全局环境。
1. 利用that保存this是一直内部环境调用外部环境的常用方法。
