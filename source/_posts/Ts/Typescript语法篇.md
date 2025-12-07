---
title: Typescript语法篇
date: 2020-09-25 17:14:10
tags:
  - ts
categories:
  - 前端
---
## 1.typescript的原始类型

1. boolean
2. number 
3. string 
4. void（只有`null`和`undefined`可以赋给`void`）
5. undefined 和 null（是所有类型的子类型，严格模式下只能赋值给对应的类型或者any）
6. symbol
7. bigint

<!--more-->

## 2.Typescript 中其他常见类型

1. any（为任意类型）
- **变量如果在声明的时候，未指定其类型或者初始化，那么它会被识别为any类型**
2. unknown

  - 较any类型更安全s

  - 该类型变量被确定为某一类型前，不能进行任何操作
 ```ts
    let value: unknown;
   
    value.foo.bar;  // ERROR
    value();        // ERROR
    new value();    // ERROR
    value[0][1];    // ERROR
 ```
3. never

	- 永不存在值的类型

	- 是任何类型的子类型，可以赋值给任何类型
	- 没有类型是 never 的子类型，任意类型都不能赋值给never类型（包括any）
	- 常用在 **抛出异常的函数** 和 **空数组且永远为空数组**

4. 数组类型(array)

	```ts
	// 有两种定义方式
	const list: Array<number> = [1, 2, 3]	// 泛型
	
	const list: number[] = [1, 2, 3]
	```

5. 元组(tuple)
	- 已知元素数量及类型的数组
    ```ts
    let x: [string, number];	// 两个元素，类型顺序也不能变
    ```
    - ts允许元组使用数组的push方法，但我们访问新加入的元素时会报错
   
   ```ts
   const tuple: [string, number] = ['a', 1];
   tuple.push(2); // ok
   console.log(tuple); // ["a", 1, 2] -> 正常打印出来
   console.log(tuple[2]); // Tuple type '[string, number]' of length '2' has no element at index '2'
   ```
   
6. object

## 3.枚举类型

```ts
enum Direction {
    Up,			// 0
    Down = 2,	// 2
    Left,		// 3
    Right,  	// 4
}
```

不带初始化的枚举 要么放在第一个位置， 要么在被数字常量或其它常量初始化的枚举后面，否则会报错

### 3.1 枚举的本质

枚举类型被编译为 JavaScript的形式如下所示，是具有双向映射的特性（字符串类型除外）

```ts
var Direction;
(function (Direction) {
    Direction[Direction["Up"] = 0] = "Up";
    Direction[Direction["Down"] = 2] = "Down";
    Direction[Direction["Left"] = 3] = "Left";
    Direction[Direction["Right"] = 4] = "Right";
})(Direction || (Direction = {}));
```

### 3.2 const声明的枚举

```ts
const a = Direction.Up;
```

会被编译为

```ts
var a = 0;
```

### 3.2 联合枚举与枚举成员的类型

**如果枚举成员均有字面量类型组成，那么枚举的每个成员和枚举值本身都可以作为类型来使用**

- 任何字符串字面量（例如： `"foo"`， `"bar"`， `"baz"`）
- 任何数字字面量（例如： `1`, `100`）
- 应用了一元 `-`符号的数字字面量（例如： `-1`, `-100`）

```ts
declare let a: Direction

enum Animal {
    Dog,
    Cat
}

a = Direction.Up // ok
a = Animal.Dog // 不能将类型“Animal.Dog”分配给类型“Direction”
```

### 3.3 枚举合并

如果基于之前的Direction再定义了一个枚举类型的Direction，会合并成一整个

```ts
enum Direction {
    Center = 1
}
```

## 5.接口(interface)

### 5.1 属性修饰符

```ts
interface User {
    name: string
    age?: number
    readonly isMale: boolean
}
```

- ?：**可选**属性
- readonly：**只读**属性

### 5.2 属性检查

下面这个程序已经正确地类型化了，因为`width`属性是兼容的，不存在`color`属性，而且额外的`colour`属性是无意义的

```ts
interface Config {
    color?: string;
    width?: number;
}

function createSquare(config: Config): { color: string; area: number } {
    // ...
}

// error: 'colour' not expected in type 'Config'
let mySquare = createSquare({ colour: "red", width: 100 });
```

注意我们传入的参数是 `colour`，并不是 `color`

> 官方文档给了三种方式绕过这种检查

第一种使用类型断言：

```ts
let mySquare = createSquare({ colour: "red", width: 100 } as Config);
```

第二种添加字符串索引签名：

```ts
interface Config {
    color?: string;
    width?: number;
    [propName: string]: any;
}
```

这样Config可以有任意数量的属性，并且只要不是width或color，那么就无所谓他们的类型是什么了。

第三种将字面量赋值给另外一个变量：

```ts
let options: any = { widdth: 5 };
let mySquare = CalculateAreas(options);
```

### 5.3 可索引类型

​	TypeScript支持两种索引签名：**字符串**和**数字**。 可以同时使用两种类型的索引，但是**数字索引的返回值必须是字符串索引返回值类型的子类型**。 这是因为当使用 `number`来索引时，JavaScript会将它转换成`string`然后再去索引对象。 也就是说用 `100`（一个`number`）去索引等同于使用`"100"`（一个`string`）去索引，因此两者需要保持一致。

```ts
class Animal {
    name: string;
}
class Dog extends Animal {
    breed: string;
}

// 错误：使用数值型的字符串索引，有时会得到完全不同的Animal!
interface NotOkay {
    [x: number]: Animal;
    [x: string]: Dog;
}
```

#### 索引类型查询操作符

`keyof`，即索引类型查询操作符，我们可以用 keyof 作用于泛型`T`上来获取泛型T上的所有 public 属性名构成联合类型。举个例子，有一个Images类，包含`src`和`alt`两个public属性，我们用`keyof`取属性名：

```ts
class Images {
    public src: string = 'https://www.google.com.hk/images/branding/googlelogo/1x/googlelogo_color_272x92dp.png'
    public alt: string = '谷歌'
    public width: number = 500
}

type propsNames = keyof Images
```
效果如下：
![2019-06-26-06-17-29](D:\OneDrive - mail2.gdut.edu.cn\typora_img\Typescript语法篇\16dbb13efd03fd86)
`keyof` 正是赋予了开发者查询索引类型的能力。

#### 映射类型

映射类型的语法是`[K in Keys]`如果我们要把所有的属性成员变为可选类型，那么需要`T[K]`取出相应的属性值，最后我们重新生成一个可选的新类型`{ [K in keyof T]?: T[K] }`。

用类型别名表示就是：

```ts
type partial<T> = { [K in keyof T]?: T[K] }
```

### 5.4 继承接口

```ts
interface VIPUser extends User {
    broadcast: () => void
}
```

VIPUser具有User所有的属性

## 6.类(Class)

### 6.1抽象类

> 通常作为派生类的基类使用，与接口不同的是抽象类可以包含成员的实现

下面定义了一个Animal抽象类
```ts
abstract class Animal {
    abstract makeSound(): void;
    move(): void {
        console.log('roaming the earch...');
    }
}
```

如果直接实例化Animal抽象类则会报错，我们可以创建子类继承基类，然后实例化子类

```ts
class Cat extends Animal {

    makeSound() {
        console.log('miao miao')
    }
}

const cat = new Cat()

cat.makeSound() // miao miao
cat.move() // roaming the earch...
```

### 6.2 访问限定符

1. public
	- 类的成员默认为public
	- 可被外部访问
2. private
	- 只能类**内部**访问
3. protected
	- 只能被类的**内部**以及类的**子类**访问

### 6.3 存取器

属性具有get和set修饰符，**只带有`get`不带有`set`的存取器自动被推断为`readonly`**

## 7. 函数(Function)

### 7.1 函数类型

在小括号后面声明返回值类型

```ts
function add(x: number, y: number): number {
    return x + y;
}
```

用接口定义函数的形状
我们也可以使用接口的方式来定义一个函数需要符合的形状：

```ts
interface SearchFunc {
    (source: string, subString: string): boolean;
}

let mySearch: SearchFunc;
mySearch = function(source: string, subString: string) {
    return source.search(subString) !== -1;
}
```

采用函数表达式|接口定义函数的方式时，对等号左侧进行类型限制，可以保证以后对函数名赋值时保证参数个数、参数类型、返回值类型不变。

### 7.2 可选参数

利用`?`或则`默认值`设置可选参数，**？可选参数必须放最后**，**默认值没必要放最后，但是不放最后必须使用undefined显式获取默认值**

```ts
function add(x: number, y?: number): number {
    return x + y;
}
```

### 7.3 剩余参数

剩余参数与JavaScript种的语法类似，需要用`...`来表示剩余参数，而剩余参数`rest`则是一个由number组成的数组，在本函数中用 reduce 进行了累加求和。

```
const add = (a: number, ...rest: number[]) => rest.reduce(((a, b) => a + b), a)
```

### 7.4 this参数

如果直接使用this进行一些操作,typescript会进行报错，可以提供一个显式的`this`参数,该参数是假的,但是可以使重用变得清晰

```ts
interface Deck {
    suits: string[];
    createCardPicker(this: Deck): () => string[];
}
let deck: Deck = {
    suits: ["hearts", "spades", "clubs", "diamonds"],
    // 注意:这个函数现在显式地指定它的被调用者必须是Deck类型
    createCardPicker: function(this: Deck) {
        return () => {
            return this.suits;
        }
    }
}
```

### 7.5 重载(Overload)

函数根据传入不同的参数而返回不同类型的数据
查找重载列表，尝试使用第一个重载定义。 如果匹配的话就使用这个

下面的assigned只能传递1、2、4个参数,用重载可以很好的对不同的参数列表进行检测

```ts
// 重载
interface Direction {
  top: number,
  bottom?: number,
  left?: number,
  right?: number
}
function assigned(all: number): Direction
function assigned(topAndBottom: number, leftAndRight: number): Direction
function assigned(top: number, right: number, bottom: number, left: number): Direction

function assigned (a: number, b?: number, c?: number, d?: number) {
  if (b === undefined && c === undefined && d === undefined) {
    b = c = d = a
  } else if (c === undefined && d === undefined) {
    c = a
    d = b
  }
  return {
    top: a,
    right: b,
    bottom: c,
    left: d
  }
}

assigned(1)
assigned(1,2)
assigned(1,2,3)		// 报错
assigned(1,2,3,4)
```

## 8. 泛型(generic)

下图`T`为一种类型变量，用于表示一种类型而不是值，我们给identity添加了类型变量`T`。 `T`帮助我们捕获用户传入的类型（比如：`number`）。之后我们再次使用了`T`当做返回值类型，现在我们知道identity的参数类型和放回置类型是相同的了。

```ts
function identity<T>(arg: T): T {
    return arg;
}
```

### 8.1 泛型接口

下图的`T`为整个接口的一个参数，而再使用`GenericIdentityFn`时，还得传入一个类型参数来指定泛型类型（这里是：`number`）

```ts
interface GenericIdentityFn<T> {
    (arg: T): T;
}

function identity<T>(arg: T): T {
    return arg;
}

let myIdentity: GenericIdentityFn<number> = identity;
```

### 8.2 泛型类

与接口一样，直接把泛型类型放在类后面

```ts
class GenericNumber<T> {
    zeroValue: T;
    add: (x: T, y: T) => T;
}

let myGenericNumber = new GenericNumber<number>();
```

### 8.3 泛型约束

下面例子想要访问arg.length属性，但是是编译器并不能证明每种类型都有`length`属性，所以就报错了

```ts
function loggingIdentity<T>(arg: T): T {
    console.log(arg.length);  // Error: T doesn't have .length
    return arg;
}
```

我们可以定义一个接口来描述约束条件，然后需要传入符合约束类型的值，必须包含必须的属性

```ts
interface Lengthwise {
    length: number;
}

function loggingIdentity<T extends Lengthwise>(arg: T): T {
    console.log(arg.length);  // Now we know it has a .length property, so no more error
    return arg;
}

loggingIdentity(3);  // Error, number doesn't have a .length property
loggingIdentity({length: 10, value: 3}); // right
```

### 8.4 多重类型进行泛型约束

用**交叉类型**`&`进行多重约束

```ts
interface FirstInterface {
  doSomething(): number
}

interface SecondInterface {
  doSomethingElse(): string
}

class Demo<T extends FirstInterface & SecondInterface> {
  private genericProperty: T

  useT() {
    this.genericProperty.doSomething() // right
    this.genericProperty.doSomethingElse() // right
  }
}
```

### 8.5 泛型里使用类类型

我们假设需要声明一个泛型拥有构造函数，比如：

```ts
function create<T>(type: T): T {
  return new type() // This expression is not constructable.
}
```

但是这样会报错，因为我们没有声明`T`是构造函数，我们需要显式的用`new`来声明这个泛型`T`是构造函数

```ts
function create<T>(type: {new(): T}): T {
    return new type();
}
```

参数`type`的类型`{new(): T}`就表示此泛型`T`是可被构造的，在被实例化后的类型是泛型`T`

一个更高级的例子，使用原型属性推断并约束构造函数与类实例的关系。

```ts
class BeeKeeper {
    hasMask: boolean;
}

class ZooKeeper {
    nametag: string;
}

class Animal {
    numLegs: number;
}

class Bee extends Animal {
    keeper: BeeKeeper;
}

class Lion extends Animal {
    keeper: ZooKeeper;
}

function createInstance<A extends Animal>(c: new () => A): A {
    return new c();
}

createInstance(Lion).keeper.nametag;  // typechecks!
createInstance(Bee).keeper.hasMask;   // typechecks!
```

## 9. 类型兼容性

Ts结构化类型系统的基本规则是，如果`x`要兼容`y`，那么`y`至少具有与`x`相同的属性，编译器会检查`x`中的每个属性，看是否能在`y`中也找到对应属性

## 10. 高级类型

### 10.1 交叉类型

用`&`可以将多个类型合并为一个类型

### 10.2 联合类型

用`|`表示一个值可为几种类型之一

```ts
function formatCommandline(command: string[] | string) {
  let line = '';
  if (typeof command === 'string') {
    line = command.trim();
  } else {
    line = command.join(' ').trim();
  }
}
```

###  10.3 类型别名

`type`虽然看起来和interface一样，但是可以用在原始类型、联合类型、元组等需要手写的类型，**类型别名也可以是泛型**

```ts
type some = boolean | string

const b: some = true // ok
const c: some = 'hello' // ok
const d: some = 123 // 不能将类型“123”分配给类型“some”
```

类型别名和接口的区别

1. 类型别名不能被`extends`和`implements`（自己也不能`extends`和`implements`其它类型）
2. interface 可以实现接口合并声明

### 10.4 条件类型

```ts
T extends U ? X : Y
```

上面的代码可以理解为: 若 `T` 能够赋值给 `U`，那么类型是 `X`，否则为 `Y`,有点类似于JavaScript中的三元条件运算符

## 11. 类型保护和类型断言

### 11.1 ！

> 类型检查器认为 `null`与 `undefined`可以赋值给任何类型

可以使用`--strictNullChecks`来使变量不自动的包含`null`或 `undefined`，但是开启后可选参数会被自动地加上`| undefined`。

如果编译器不能够去除 `null`或 `undefined`，你可以使用**类型断言**手动去除。 语法是添加`!`后缀： `identifier!`从 `identifier`的类型里去除了 `null`和 `undefined`

```ts
function broken(name: string | null): string {
  function postfix(epithet: string) {
    return name.charAt(0) + '.  the ' + epithet; // error, 'name' is possibly null
  }
  name = name || "Bob";
  return postfix("great");
}

function fixed(name: string | null): string {
  function postfix(epithet: string) {
    return name!.charAt(0) + '.  the ' + epithet; // ok
  }
  name = name || "Bob";
  return postfix("great");
}
```

### 11.2 is进行类型保护

is之后的类型必须是参数类型中的一个，在后续调用改参数时，ts会将变量缩减为那个具体的类型

```ts
function isString(test: any): test is string{
    return typeof test === 'string';
}

function example(foo: number | string){
    if(isString(foo)){
        console.log('it is a string' + foo);
        // 如果上面没写test is string，foo.length将会报错
        console.log(foo.length); // string function
    }
}
example('hello world');
```

## 参考

https://www.tslang.cn/docs/handbook
https://ts.xcatliu.com/basics/type-of-function.html
