+++
title = 'Hook'
date = 2026-06-12T11:41:30+08:00
draft = false
categories = ["Tech"]
tags = ["Notes"]
+++

> 起因是看 LangChain 的 Agent 时遇到了 `before_model`、`before_agent` 这类东西，再加上之前零散听过的 "Hook" 概念，于是想把它彻底搞清楚。这篇笔记从 Hook 是什么讲起，再拆解它的四种注册机制

### Hook 是什么

Hook 在不同语境下指代不同的两样东西，一个是 **Hook Point (挂载点)**，一个是 **Hook Function (钩子函数)**

#### **Hook Point**

是框架/系统预留的 "空位" -- 一个特定位置，系统在执行到这里时会主动查询 "有没有人注册了回调? 有的话就调用一下"。它是被动的，本身不含逻辑，只是一个触发机制

> 当执行流到达某个特定位置时，运行时会检查是否有函数被提前注册到此处。如果有，则在此处调用它。这个 "被传递出去、等待被外部调用" 的函数，称为 callback function。而这整个 "将函数作为参数传递给另一个函数或系统，由接收方决定调用时机" 的机制，称为 callback

#### **Hook Function**

是你写的那段代码-- 真正被"挂上去"的逻辑。它是主动的，是实际执行的内容

### 注册机制

> 肯定会想到，那 Hook Function 是怎么注册到 Hook Point 上的呢

下面四个机制看起来差别很大，但本质完全相同：**都是在某个时刻把一个函数引用存到某处，让框架到达 Hook Point 时能找到并调用它**。区别只在于 "谁来存、什么时候存、存在哪里"。带着这个统一视角去看，会清晰很多。

#### 显式调用注册函数

框架暴露一个注册函数，然后开发者主动调用框架提供的注册函数 (如 `add_action`)，传入两个参数:

- 第一个参数: Hook Point 的名称 (字符串)，用于告诉注册表，这个 callback function 对应哪个 Hook Point。
- 第二个参数: callback function，此时只是被传入并存进注册表，尚未执行

```js
// ── 第一步：注册表，用来存放所有注册过的 callback ──
const registry = {};

// ── 第二步：add_action 的内部实现 ──
function add_action(hookName, callback) {
  registry[hookName] = callback;
  // 把 hookName 作为 key，callback 作为 value 存进注册表
  // 此时 callback 只是被存起来，还没有执行
}

// ── 第三步：run_hooks 的内部实现 ──
function run_hooks(hookName, data) {
  if (registry[hookName]) {
    // 去注册表查询有没有这个名字
    registry[hookName](data); // 有的话就调用它，并把 data 传进去
  }
  // 没有的话就跳过
}

// ── 第四步：框架内部的业务函数，包含 Hook Point ──
function save_post(post_id) {
  do_actual_save(post_id); // 框架自己的保存逻辑
  run_hooks("save_post", post_id); // ← Hook Point，执行到这里开始查注册表
}

// ── 第五步：开发者注册自己的 callback ──
add_action("save_post", function (post_id) {
  console.log("文章保存了，id 是", post_id);
});

// ── 第六步：某个地方触发了 save_post ──
save_post(42);

// 执行流：
// 1. save_post(42) 被调用
// 2. do_actual_save(42) 执行
// 3. run_hooks('save_post', 42) 执行，到达 Hook Point
// 4. 去注册表查询 'save_post'，找到了对应的 callback function
// 5. 调用该 callback function，并将 42 作为 post_id 传入
// 6. 打印：文章保存了，id 是 42
```

`run_hooks("save_post", post_id)` 这一行就是 Hook Point，`run_hooks` 内部负责去注册表查找并调用 callback function

**优点：**

- 注册时机灵活 (任何时候都能注册)
- 一个 Hook Point 可以挂多个 Hook Function
- 可以按优先级排序执行

**缺点：**

- 必须在 Hook Point 触发前完成注册
- Hook Point 名称是字符串，拼错不报错

#### 配置文件/装饰器声明

用装饰器或配置文件声明"这个 Hook Function 挂到哪个 Hook Point"。框架在启动时扫描代码或配置，自动完成注册。不需要手动调用注册函数，框架会在初始化阶段替你完成这一步

```python
@app.before_request
def check_auth():
    if not current_user.is_authenticated:
        return redirect('/login')
```

`@app.before_request` 做的额外处理就是：**自动把 `check_auth` 这个函数存进注册表**，相当于替你调用了注册函数，等价于:

```python
def check_auth():
    if not current_user.is_authenticated:
        return redirect('/login')

# 装饰器替你做了这一步
app.register_hook('before_request', check_auth)
```

**优点：**

- 代码意图清晰，Hook 就在函数旁边
- 框架统一管理，不会忘记注册

**缺点：**

- 注册时由框架控制，不够灵活
- 框架耦合度高，换框架要重写

#### 子类覆写方法

父类预先定义好空方法作为 **Hook Point**，子类通过 `@Override` 覆写这个方法，覆写本身就完成了注册，不需要手动调用任何注册函数。当执行流到达父类中调用该方法的那一行时，Java 运行时自动找到子类的实现并执行

```java
// 父类，框架写的
class Animal {

    // Hook Point，空方法，等着子类来填
    void onEat() { }

    // 框架的业务逻辑
    final void eat() {
        System.out.println("开始吃东西");
        onEat();                           // ← Hook Point，触发时调子类实现
        System.out.println("吃完了");
    }
}

// 子类，开发者写的
class Dog extends Animal {

    @Override
    void onEat() {                         // ← 覆写 = 注册 Hook Function
        System.out.println("狗狗摇尾巴");   // ← Hook Function 的逻辑
    }
}
```

> 关于 `super`：上面的例子父类 `onEat` 是空的，调不调用 `super.onEat()` 结果一样。但在真实框架里 (如 Android 的 `onCreate`)，父类方法含有框架必须执行的初始化逻辑，此时 **必须**调 `super`，否则框架内部状态没准备好，会报错或行为异常

**优点：**

- IDE 补全友好
- 结构清晰，每个 Hook Point 对应一个方法名

**缺点：**

- 单继承限制，一个 Hook Point 只能有一个实现
- 当父类方法含有框架逻辑时，必须记得调 `super`，否则破坏框架状态

#### Monkey Patch/运行时替换

Monkey Patch 是一个术语，意思是 **在程序运行时，动态修改已有的代码，而不改动原始源文件**。

这样就不需要框架提前设计 Hook Point，而是在与运行时直接替换对象上的某个函数引用。先把原函数保存起来，再用一个新函数覆盖原来的属性名。新函数在前后插入自己的逻辑，中间调用原函数，完成包裹。这是一种侵入式的挂载方式，常用于测试 mock 和临时打补丁

```Js
const api = {
    fetchUser(id) { return db.query(id) }  // 原始函数
}

const original = api.fetchUser             // 1. 先把原函数保存起来

api.fetchUser = function(id) {             // 2. 用新函数替换掉原属性名
    console.log('before:', id)             // ← 前置逻辑
    const result = original(id)            // 3. 调用原函数，保证原有逻辑不丢失
    console.log('after:', result)          // ← 后置逻辑
    return result
}
```

**优点：**

- 无需框架支持，任何代码都能挂
- 灵活，可以临时挂、用完撤销

**缺点：**

- 原函数引用管理复杂，容易出 bug
- 调试困难，不建议在生产代码中使用

### 小结

现在就能够理解 LangChain 里的 `before_model`、`before_agen` 了，框架在 Agent 的执行流程中预留了一系列 Hook Point，通过注册 callback function 在这些位置插入自己的逻辑 (打日志、修改输入、监控性能等)
