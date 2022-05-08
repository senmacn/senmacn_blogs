# 什么是继承

JavaScript 中只有一种结构：对象。每个实例对象（ `object` ）都有一个私有属性（称之为 `__proto__` ）指向它的构造函数的原型对象（`prototype` ）。该原型对象也有一个自己的原型对象( `__proto__` ) ，层层向上直到一个对象的原型对象为 `null` (位于原型链顶端的 `Object`)。

根据定义，`null` 没有原型，并作为这个原型链中的最后一个环节。

# 什么是原型链

ECMAScript 中描述了原型链的概念，并将原型链作为实现继承的主要方法。其基本思想是利用原型让一个引用类型继承另一个引用类型的属性和方法。

让原型对象等于另一个类型的实例，此时的原型对象将包含一个指向另一个原型的指针，相应地，另一个原型中也包含着一个指向另一个构造函数的指针。假如另一个原型又是另一个类型的实例，那么上述关系依然成立，如此层层递进，就构成了实例与原型的链条。

# 基于原型链的继承

JavaScript 对象是动态的属性“包”（指其自己的属性）。JavaScript 对象有一个指向一个原型对象的链。当试图访问一个对象的属性时，它不仅仅在该对象上搜寻，==还会搜寻该对象的原型==，以及该对象的原型的原型，依次层层向上搜索，直到找到一个名字匹配的属性或到达原型链的末尾。

## `[[Prototype]]` 和 `__proto__`

遵循ECMAScript标准，`someObject.[[Prototype]]` 符号是用于指向 `someObject` 的原型。

`[[Prototype]]` 是内部属性，并不能直接访问到，所以使用 `_proto_` 来访问。

> 从 ECMAScript 6 开始，`[[Prototype]]` 可以通过 `Object.getPrototypeOf()` 和 `Object.setPrototypeOf()` 访问器来访问。

## `prototype`

`[[Prototype]]` 和 `__proto__` 不应该与构造函数 `func` 的 `prototype` 属性相混淆。被构造函数创建的实例对象的 `[[Prototype]]` (`__proto__`) 指向 `func` 的 `prototype` 属性。`Object.prototype` 属性表示 `Object`  的原型对象。

基本上所有函数都有这个属性，但是也有一个例外：`let fun = Function.prototype.bind()`, 如果以上述方法创建一个函数，那么可以发现这个函数是不具有 prototype 属性的。

![image](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/83dd80f8c1864e09bd42f2a98f8b67a7~tplv-k3u1fbpfcp-zoom-1.image)

## 对象的创建过程

```Javascript
function new() {
    // 创建一个空的对象，此时 obj.__proto__ === Object.prototype
	let obj = new Object();
	// 获得构造函数
	let Con = [].shift.call(arguments);
	// 链接到原型
	obj.__proto__ = Con.prototype
	// 绑定 this，执行构造函数
	let result = Con.apply(obj, arguments);
	// 确保 new 出来的是个对象
	return typeof result && result  === 'object' ? result : obj;
}

// 等同于 let objFunc = new Function(...);
function objFunc() {
    this.a = 1;
    this.b = 2;
}

// 类似实现 const obj = new objFunc;
const obj = new(objFunc);

obj.__proto__ === objFunc.prototype; // true

objFunc.prototype.__proto__ === Object.prototype; // true

objFunc.__proto__ === Function.prototype; // true

Function.prototype === Function.__proto__; // true ?

// 原型链
objFunc.prototype.b = 3;
objFunc.prototype.c = 4;

// {a:1, b:2} ---> {b:3, c:4} ---> Object.prototype---> null
```

> `Function.proto === Function.prototype`
<br>
`Function` 是引擎自己创建的。首先引擎创建了 `Object.prototype` ，然后创建了 `Function.prototype`，并且通过 `__proto__` 将两者联系了起来。

> 使用 `new Object()` 的方式创建对象需要通过作用域链一层层找到 `Object`，但是使用字面量的方式就没这个问题。

## 函数的继承

在 JavaScript 里，任何函数都可以添加到对象上作为对象的属性。函数的继承与其他的属性继承没有差，包括重写。

```javascript
var o = {
  a: 2,
  m: function(){
    return this.a + 1;
  }
};

console.log(o.m()); // 3
// 当调用 o.m 时，'this' 指向了 o.

var p = Object.create(o);
// p是一个继承自 o 的对象

p.a = 4; // 创建 p 的自身属性 'a'
console.log(p.m()); // 5
// 调用 p.m 时，'this' 指向了 p
// 又因为 p 继承了 o 的 m 函数
// 所以，此时的 'this.a' 即 p.a，就是 p 的自身属性 'a'
```

## 不同的方法创建对象的原型链

### 对象字面量创建

```javascript
var o = {a: 1};
// 原型链：
// o ---> Object.prototype ---> null

var a = ["yo", "whadup", "?"];
// 数组都继承于 Array.prototype
// 原型链：
// a ---> Array.prototype ---> Object.prototype ---> null

function f(){
  return 2;
}
// 函数都继承于 Function.prototype
// 原型链：
// f ---> Function.prototype ---> Object.prototype ---> null
```

### 构造函数创建（`new`）

```javascript
function People() {
  this.name = '';
}

People.prototype = {
  getName: function(){
    return this.name;
  }
};

var man = new People();
```

### 使用 `Object.create` 创建的对象

ECMAScript 5 中引入了一个新方法：`Object.create()`。可以调用这个方法来创建一个新对象。新对象的原型就是调用 create 方法时传入的第一个参数


```javascript
var a = {a: 1};
// a ---> Object.prototype ---> null

var b = Object.create(a);
// b ---> a ---> Object.prototype ---> null
console.log(b.a); // 1 (继承而来)

var c = Object.create(b);
// c ---> b ---> a ---> Object.prototype ---> null

var d = Object.create(null);
// d ---> null
console.log(d.hasOwnProperty); // undefined, 因为d没有继承Object.prototype
```

### 使用 `class` 关键字创建的对象

ECMAScript6 引入了一套新的关键字用来实现 `class`;

1. 类声明不可以提升;
2. 重复声明一个类会引起类型错误;
3. 若之前使用类表达式定义了一个类，则再次声明这个类同样会引起类型错误。

```javascript
class Polygon {
  constructor(height, width) {
    this.height = height;
    this.width = width;
  }
}

class Square extends Polygon {
  constructor(sideLength) {
    super(sideLength, sideLength);
  }
  get area() {
    return this.height * this.width;
  }
  set sideLength(newLength) {
    this.height = newLength;
    this.width = newLength;
  }
}

var square = new Square(2);

// 使用类表达式
let Polygon = class {
  constructor(height, width) {
     this.height = height;
    this.width = width;
  }
  getArea() {
    return this.height * this.width;
  }
};
```

### 性能相关

如上所述，在原型链上查找属性比较耗时，对性能有副作用，这在性能要求苛刻的情况下很重要。另外，试图访问不存在的属性时会遍历整个原型链。

遍历对象的属性时，原型链上的每个可枚举属性都会被枚举出来。要检查对象是否具有自己定义的属性，而不是其原型链上的某个属性，则必须使用所有对象从 `Object.prototype` 继承的 `hasOwnProperty` 方法。

`hasOwnProperty` 是 JavaScript 中唯一一个处理属性并且不会遍历原型链的方法。

> 无法检查属性是否为 `undefined`, 该属性可能已存在，但其值恰好被设置成了 undefined。

### 原型链的问题

在通过原型来实现继承时，原型实际上会变成另一个类型的实例。于是，原先的实例属性也就顺理成章地变成了现在的原型属性了。

```javascript
function SuperType(){ 
    this.colors = ["red", "blue", "green"];
} 

function SubType() {} 

//继承了 SuperType 
SubType.prototype = new SuperType(); 
 
var instance1 = new SubType(); 
instance1.colors.push("black"); 
console.log(instance1.colors);     //"red,blue,green,black" 
 
var instance2 = new SubType(); 
console.log(instance2.colors);       //"red,blue,green,black" 
```

这个例子中的 `SuperType` 构造函数定义了一个 `colors` 属性，该属性包含一个数组（引用类型值）。`SuperType` 的每个实例都会有各自包含自己数组的 `colors` 属性。当 `SubType` 通过原型链继承了 `SuperType` 之后，`SubType.prototype` 就变成了 `SuperType` 的一个实例，因此它也拥有了一个它自己的 `colors` 属性。 `SubType` 的所有实例都会共享这一个 `colors` 属性。

另一个问题是，在创建子类型的实例时，不能向超类型的构造函数中传递参数。实际上，应该说是没有办法在不影响所有对象实例的情况下，给超类型的构造函数传递参数。 


# 原型链的多种实现

## 借用构造函数 

```javascript
function SuperType(name){
    this.name = name;
    this.colors = ["red", "blue", "green"];
} 
 
function SubType(){   
    //继承了 SuperType，同时还传递了参数 
    SuperType.call(this, 'someName'); 
     
    //实例属性 
    this.age = 29; 
} 
 
let instance1 = new SubType(); 
alert(instance1.name);    //"Nicholas"; 
alert(instance1.age);     //29 
 
instance1.colors.push("black"); 
console.log(instance1.colors);    //"red,blue,green,black" 
 
let instance2 = new SubType(); 
console.log(instance2.colors);    //"red,blue,green" 
```

相对于直接修改 `prototype` 实现原型链而言，借用构造函数不仅避免了公用原型属性的问题，还可以在子类型构造函数中向超类型构造函数传递参数。

问题：方法都在构造函数中定义，因此函数复用很难。借用构造函数的技术也是很少单独使用。

## 组合继承 

使用原型链实现对原型属性和方法的继承，而通过借用构造函数来实现对实例属性的继承。既通过在原型上定义方法实现了函数复用，又能够保证每个实例都有它自己的属性。

```javascript
function SuperType(name){ 
    this.name = name; 
    this.colors = ["red", "blue", "green"]; 
} 
 
SuperType.prototype.sayName = function(){ 
    alert(this.name);
}; 
 
function SubType(name, age){   
    //继承属性 
    SuperType.call(this, name); 
     
    this.age = age; 
} 
 
//继承方法 
SubType.prototype = new SuperType();
// 绑定constructor
SubType.prototype.constructor = SubType; 
SubType.prototype.sayAge = function(){ 
    alert(this.age); 
}; 
 
var instance1 = new SubType("Nicholas", 29); 
instance1.colors.push("black"); 
alert(instance1.colors);      //"red,blue,green,black" 
instance1.sayName();          //"Nicholas"; 
instance1.sayAge();           //29 
 
var instance2 = new SubType("Greg", 27); 
alert(instance2.colors);      //"red,blue,green" 
instance2.sayName();          //"Greg"; 
instance2.sayAge();           //27 
```
组合继承避免了原型链和借用构造函数的缺陷，融合了它们的优点，是 JavaScript 中最常用的继承模式。而且，`instanceof` 和 `isPrototypeOf()` 也能够用于识别基于组合继承创建的对象。 

## 原型式继承(`Object.create()`)

```javascript
function object(o){ 
    function F(){} 
    F.prototype = o; 
    return new F(); 
}
```

ECMAScript 5 通过新增 `Object.create()`方法规范化了原型式继承。这个方法接收两个参数：一个用作新对象原型的对象和（可选的）一个为新对象定义额外属性的对象。`Object.create()` 方法的第二个参数与 `Object.defineProperties()` 方法的第二个参数格式相同：每个属性都是通过自己的描述符定义。以这种方式指定的任何属性都会覆盖原型对象上的同名属性。

```javascript
var person = { 
    name: "Nicholas", 
    friends: ["Shelby", "Court", "Van"] 
};

var anotherPerson = Object.create(person, { 
    name: { 
        value: "Greg" 
    } 
}); 
     
console.log(anotherPerson.name); //"Greg" 
```

包含引用类型值的属性始依旧终都会共享相应的值。

## 寄生式继承

```javascript
function createAnother(original){ 
    var clone = object(original);   //通过调用函数创建一个新对象 
    clone.sayHi = function(){       //以某种方式来增强这个对象 
        alert("hi"); 
    }; 
    return clone;            //返回这个对象 
} 
```

使用寄生式继承来为对象添加函数，会由于不能做到函数复用而降低效率；这一点与构造函数模式类似。

## 寄生组合式继承

```javascript
function SuperType(name){ 
    this.name = name; 
    this.colors = ["red", "blue", "green"]; 
} 
 
SuperType.prototype.sayName = function(){ 
    alert(this.name); 
}; 
 
function SubType(name, age){   
    SuperType.call(this, name);         //第二次调用 SuperType() 
     
    this.age = age; 
} 
 
SubType.prototype = new SuperType();    //第一次调用 SuperType() 
SubType.prototype.constructor = SubType; 
SubType.prototype.sayAge = function(){ 
    alert(this.age); 
}; 
```

寄生组合式继承的高效率体现在它只调用了一次 `SuperType` 构造函数，并且因此避免了在 `SubType`。 `prototype` 上面创建不必要的、多余的属性。与此同时，原型链还能保持不变；因此，还能正常使用 `instanceof` 和 `isPrototypeOf()`。
