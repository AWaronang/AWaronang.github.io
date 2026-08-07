+++
title = 'TypeScript Generics 学习笔记'
date = 2026-08-07T14:47:47+08:00
draft = false
categories = ["Notes"]
tags = ["Notes", "Input"]

+++

> 一些粗略地见解，如有写错或低级错误，欢迎您的指正

### 1. 为什么需要泛型

泛型解决的核心问题: **如何写出即复用又不丢失类型信息的代码**

来看 "identity" 函数，这个最简单的例子:

```ts
// 方案 A：写死类型 —— 不能复用
function identity(arg: number): number {
  return arg;
}

// 方案 B：用 any —— 能复用，但丢失了类型信息
function identity(arg: any): any {
  return arg; // 传进去 number，返回的却"只是 any"，编译器不知道具体是什么
}

// 方案 C：泛型 —— 既能复用，又保留类型信息
function identity<Type>(arg: Type): Type {
  return arg;
}
```

**Any 和泛型的关键区别**:

- `any` 能跑，但是代价是 **类型信息彻底丢失**。传进去一个 `number`，TS 只知道返回的是 `any`，完全不知道具体是什么类型，后面用起来毫无类型提示和检查，等于白写了 TS
- **泛型解决的就是这个矛盾**：既要函数/类型能适配"任意类型"，又不想丢失具体的类型信息。

`Type` 在这里叫**类型变量**（type variable），它不是一个值，而是一个占位符，专门用来"捕获"调用时传入的类型。

#### 两种调用方式

```ts
// 方式 1：显式指定类型参数
let output1 = identity<string>("myString"); // output1: string

// 方式 2：类型参数推断（更常用）
let output2 = identity("myString"); // TS 自动推断 Type = string
```

大多数时候用方式 2 就够了，编译器会根据传入的实参自动推断。复杂场景 (比如推断不出来的时候) 才需要显式写 `<Type>`

### 2. 泛型变量的"使用限制"

常见的坑： 以为 `Type` 可以随便使用它的属性

```ts
function loggingIdentity<Type>(arg: Type): Type {
  console.log(arg.length);
  // ❌ 报错：Property 'length' does not exist on type 'Type'. [2339]
  return arg;
}
```

**为什么会报错呢?**: 因为 `Type` 代表 "任意类型"，编译器不能假设它一定有 `.length` (比如传入个 `number` 进来就没有)

#### 解法: 明确它是数组

```ts
function loggingIdentity<Type>(arg: Type[]): Type[] {
  console.log(arg.length); // ✅ 数组一定有 .length
  return arg;
}

loggingIdentity([1, 2, 3]); // Type 推断为 number
loggingIdentity(["a", "b"]); // Type 推断为 string
```

也可以用 `Array<Type>` 它们是等价的

```ts
function loggingIdentity<Type>(arg: Array<Type>): Array<Type> {
  console.log(arg.length);
  return arg;
}
```

### 3. 泛型类型 (函数类型 & 接口)

泛型函数本身也有一个"类型"，而这个类型长什么样，取决于函数声明时类型参数是怎么写的

**先复习下泛型函数的类型**

```ts
function identity<Type>(arg: Type): Type {
  return arg;
}

let myIdentity: <Type>(arg: Type) => Type = identity;
```

这里 `<Type>` 写在最前面, 代表: "这是一个**泛型函数**， Type 就是**泛型参数** "

**重点**: 类型参数的名字可以随便换, TS 只看"位置", 不看名字:

```ts
let myIdentity2: <Input>(arg: Input) => Input = identity; // 一样合法
```

就像你写 `function(x) { return x }` 和 `function(y) { return y }` 逻辑完全相同, 变量名不影响逻辑。

#### 思考:

```ts
let myIdentity: <Type>(arg: Type) => Type = identity;
```

- 定义了 `myIdentity` 这个变量，它的变量类型是 `<Type>(arg: Type) => Type`。`myIdentity` 这个变量，以后必须被赋值成一个函数，返回值类型必须和参数类型一致 -- **因为 `Type` 不是一个固定类型，是一个占位符**‘’。
- 将上面定义的泛型函数 `identity` 赋值给刚刚所定义的 `myIdentity` 变量。 TS 会检查 `identity` 函数的类型签名,跟 `myIdentity` 声明的类型签名做比对,发现两者"形状"完全匹配(都是"泛型 + 一个参数 + 返回同类型"),所以赋值合法。

#### 泛型接口: 两种写法，含义不同

**写法 A**: 泛型参数挂在函数签名上 → **每次调用时决定**

```ts
interface GenericIdentityFn {
  <Type>(arg: Type): Type; // 👈 这一整行就是"函数签名", 这是泛型函数签名
}

let myIdentity: GenericIdentityFn = identity;

myIdentity(42); // 这次调用,Type = number
myIdentity("hello"); // 这次调用,Type = string
myIdentity(true); // 这次调用,Type = boolean
```

`myIdentity` 这个变量本身没有绑定任何具体类型, 它保持"万能"状态, 每次调用的瞬间才决定 `Type` 是什么。

**写法 B:** 泛型参数挂在接口本身上 → **声明变量时就决定, 之后锁死**

```ts
interface GenericIdentityFn<Type> {
  (arg: Type): Type;
}

let myIdentity: GenericIdentityFn<number> = identity;

myIdentity(42); // ✅ 合法
myIdentity("hello"); // ❌ 报错!Type 已经被锁定成 number 了
```

### 4. 泛型类

写法和接口类似，类型参数写在类名后面：

```ts
class GenericNumber<NumType> {
  zeroValue: NumType;
  add: (x: NumType, y: NumType) => NumType;
}

let myGenericNumber = new GenericNumber<number>();
myGenericNumber.zeroValue = 0;
myGenericNumber.add = (x, y) => x + y;

// 换成 string 也完全没问题
let stringNumeric = new GenericNumber<string>();
stringNumeric.zeroValue = "";
stringNumeric.add = (x, y) => x + y;
console.log(stringNumeric.add(stringNumeric.zeroValue, "test")); // "test"
```

> ⚠️ **注意**：泛型类的类型参数只作用于**实例成员**，不能用在**静态成员**上（因为静态成员是属于类本身的，不属于某次具体的实例化）。TS 也不支持泛型 enum / namespace。

### 5. 泛类约束 (Generic Constraints)

回到第 2 节的报错：如果我们**只想约束**"必须有 `.length`"，而不是限制成"必须是数组"，怎么办？

#### 用 `extends` 加约束：

```ts
interface Lengthwise {
  length: number;
}

function loggingIdentity<Type extends Lengthwise>(arg: Type): Type {
  console.log(arg.length); // ✅ 编译器知道 Type 至少有 .length
  return arg;
}

loggingIdentity({ length: 10, value: 3 }); // ✅ 有 length，OK
loggingIdentity(3); // ❌ number 没有 length
loggingIdentity("hello"); // ✅ string 有 length
loggingIdentity([1, 2, 3]); // ✅ 数组有 length
```

#### 用一个类型参数约束另一个类型参数（`keyof`）

这是实际项目里非常常用的模式——安全地按 key 取对象的属性：

这里有**两个类型参数**:

- `Type`: 代表传入对象 `obj` 的类型
- `Key`: 代表传入的键名 `key` 的类型, 但它被约束成 `extends keyof Type` —— 意思是 **"Key 必须是 Type 的某个键名, 不能瞎写"**

```ts
function getProperty<Type, Key extends keyof Type>(obj: Type, key: Key) {
  return obj[key];
}

let x = { a: 1, b: 2, c: 3, d: 4 };

getProperty(x, "a"); // ✅ 返回 number
getProperty(x, "m"); // ❌ 报错：'m' 不是 'a'|'b'|'c'|'d' 之一
```

`keyof` 会把一个对象类型的**所有键名**提取出来,变成一个"联合类型" (union type)

```ts
type Point = { x: number; y: number };

type PointKeys = keyof Point; // "x" | "y"
```

```ts
let x = { a: 1, b: 2, c: 3, d: 4 };

getProperty(x, "a");
```

调用时, TS 会做这些推断:

1. 你传了 `x`, 所以 `Type = { a: number; b: number; c: number; d: number }`
2. 根据约束 `Key extends keyof Type`, 算出 `keyof Type = "a" | "b" | "c" | "d"`
3. 所以 `key` 参数只能是 `"a" | "b" | "c" | "d"` 这四个字符串之一
4. 你传了 `"a"`, 合法 ✅

### 6. 用泛型描述类 (构造函数) 本身

工厂函数场景：不是传"某个类型的值"，而是传"某个类"，然后 `new` 出来：

```ts
function create<Type>(c: { new (): Type }): Type {
  return new c();
}
```

结合继承关系约束的例子

```ts
class Animal {
  numLegs: number = 4;
}

class BeeKeeper {
  hasMask: boolean = true;
}
class ZooKeeper {
  nametag: string = "Mikle";
}

class Bee extends Animal {
  numLegs = 6;
  keeper: BeeKeeper = new BeeKeeper();
}
class Lion extends Animal {
  keeper: ZooKeeper = new ZooKeeper();
}

function createInstance<A extends Animal>(c: new () => A): A {
  return new c();
}

createInstance(Lion).keeper.nametag; // ✅ string
createInstance(Bee).keeper.hasMask; // ✅ boolean
```

这个模式是 [Mixins](https://www.typescriptlang.org/docs/handbook/mixins.html) 设计模式的基础，日常业务代码用得较少，了解即可。

### 7. 泛型参数默认值

类似函数默认参数，给类型参数一个"默认类型"，调用时可以不传：

#### 函数参数默认值

```ts
function greet(name: string = "World") {
  console.log(`Hello, ${name}`);
}

greet(); // Hello, World  (用了默认值)
greet("Mark"); // Hello, Mark   (传了就用传的)
```

**泛型参数默认值**的逻辑一模一样，只是把 `=` 加在类型参数后面:

```ts
function wrap<Type = string>(value: Type) {
  return [value];
}
```

意思是: "如果调用的时候没有明确指定 `Type` 是什么, TS 又推断不出来, 那就默认用 `string`"。

#### 例子

假设有这样的一个容器

```ts
interface Container<T, U> {
  element: T;
  children: U;
}
```

**没有默认值之前**，必须写三个重载版本，来覆盖 "不传参数 / 传一个参数 / 传两个参数"这三种情况:

```ts
declare function create(): Container<HTMLDivElement, HTMLDivElement[]>;
declare function create<T extends HTMLElement>(element: T): Container<T, T[]>;
declare function create<T extends HTMLElement, U extends HTMLElement>(
  element: T,
  children: U[],
): Container<T, U[]>;
```

三个重载分别对应三种调用方式

> `declare` 的作用是:**"告诉 TS 这个东西存在,并且长这个样子,但我不在这里给出具体实现"**。

**有了泛型参数默认值之后**, 可以合并成一个:

```ts
declare function create
  T extends HTMLElement = HTMLDivElement,
  U extends HTMLElement[] = T[]
>(element?: T, children?: U): Container<T, U>;

const div = create();
// 没传任何参数,T 用默认值 HTMLDivElement,U 用默认值 T[]
// 结果类型: Container<HTMLDivElement, HTMLDivElement[]>

const p = create(new HTMLParagraphElement());
// 传了一个 HTMLParagraphElement,T 被推断成 HTMLParagraphElement
// U 没传,用默认值 T[],也就是 HTMLParagraphElement[]
// 结果类型: Container<HTMLParagraphElement, HTMLParagraphElement[]>
```

- `T extends HTMLElement = HTMLDivElement` → "T 必须是 HTMLElement 的子类型,如果不指定,默认用 `HTMLDivElement`"
- `U extends HTMLElement[] = T[]` → "U 必须是 HTMLElement 数组,如果不指定,默认用 `T[]`"(注意,默认值里还能**引用前面的类型参数** `T`)

泛型参数默认值遵循以下规则:

- 带有默认值的类型参数被视为"可选"的。
- 必填的类型参数不能排在可选的类型参数之后。
  > ```ts
  > function fn<T = string, U>(x: T, y: U) {}
  > // ❌ 报错! T 有默认值 (可选), U 没有默认值 (必填)
  > // 必填的 U 不能排在可选的 T 后面
  > function fn<U, T = string>(x: T, y: U) {} // ✅
  > ```
- 类型参数的默认类型,必须满足该类型参数的约束条件 (如果有约束的话)。
- 指定类型实参时, 你只需要为"必填"的类型参数提供实参; 未指定的类型参数会自动使用它们的默认类型。
- 如果指定了默认类型, 并且类型推断无法选出候选类型, 那么就会推断为默认类型。
- 与已有的类 (class) 或接口 (interface) 声明发生"合并"的新声明, 可以为一个已存在的类型参数引入默认值。
  > ```ts
  > interface Box<T> {
  >   content: T;
  > }
  >
  > // 这是另一处对同一个接口的声明,TS 会自动把两次声明"合并"
  > interface Box<T = string> {
  >   // ✅ 合法,给已有的类型参数 T 补充了一个默认值
  > }
  > ```
- 与已有的类或接口声明发生"合并"的新声明, 可以引入一个新的类型参数, 只要这个新参数带有默认值。
  > ```ts
  > interface Box<T> {
  >   content: T;
  > }
  >
  > interface Box<T, U = number> {
  >   // ✅ 合法,新增了一个类型参数 U,但因为它带了默认值,
  >   // 所以不会破坏之前那些"只传了一个类型参数"的旧代码
  >   extra: U;
  > }
  > ```

### 8. Variance Annotations（协变/逆变标注）

- **协变（covariant）**：`Producer<Cat>` 可以用在需要 `Producer<Animal>` 的地方（因为 Cat 是 Animal 的子类型，"生产"关系方向一致）
- **逆变（contravariant）**：`Consumer<Animal>` 可以用在需要 `Consumer<Cat>` 的地方（因为能处理 Animal 的函数，必然能处理 Cat，方向相反）

```ts
interface Producer<out T> {
  make(): T;
} // 协变标注
interface Consumer<in T> {
  consume: (arg: T) => void;
} // 逆变标注
interface ProducerConsumer<in out T> {
  // 不变（两个方向都标注）
  consume: (arg: T) => void;
  make(): T;
}
```

TS 会**自动推断**协变/逆变关系，正常写业务代码几乎不需要手写这些标注。只有在处理复杂循环类型、且推断出错时才需要手动干预

## 📖 参考文献

- https://www.typescriptlang.org/docs/handbook/2/generics.html
